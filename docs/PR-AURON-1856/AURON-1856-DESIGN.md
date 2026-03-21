# Design: Flink Node Converter Tools (AURON #1856)

**Issue**: https://github.com/apache/auron/issues/1856
**AIP**: AIP-1 — Introduce Flink integration of native engine
**Author**: @weiqingy
**Status**: Draft (Rev 2 — incorporating reviewer feedback)

## 1. Motivation

Per AIP-1, the Flink integration requires a **planner-side framework** that converts Flink physical plan nodes (`ExecNode` tree) into Auron's native execution plan (`PhysicalPlanNode` protobuf) for Rust/DataFusion execution. The data exchange layer is now in place:

```
Flink RowData → Arrow (Writer #1850) → Native Engine → Arrow → Flink RowData (Reader #1851 DONE)
```

What's missing is the **conversion infrastructure** — the machinery that decides *which* Flink nodes can be converted and *how* to translate them into native plan representations. This is analogous to Spark's `AuronConverters` + `NativeConverters` pattern and Gluten's `RexNodeConverter` + `RexCallConverterFactory` pattern.

This issue delivers three foundational classes:

| Class | Role |
|-------|------|
| `FlinkNodeConverter` | Interface defining the contract for converting a Flink ExecNode to a native `PhysicalPlanNode` |
| `FlinkNodeConverterFactory` | Singleton registry that holds converters and dispatches to the right one |
| `ConverterContext` | Holds shared state (configuration, input schema, classloader) needed during conversion |

These classes are **framework only** — no actual converter implementations (e.g., Calc converter) are included. Subsequent issues will implement converters that plug into this framework.

## 2. Design Approach

### Two Candidate Approaches

**Approach A — Singleton factory with typed converter dispatch (recommended)**

A singleton factory holds a static registry of converters keyed by ExecNode class. Each converter translates a specific ExecNode type into Auron's `PhysicalPlanNode` protobuf. A `ConverterContext` carries the input schema and configuration needed for type-aware conversion.

This draws on best practices from:
- **Gluten** (`RexCallConverterFactory` + `RexNodeConverter`): singleton factory, `isSuitable()` + `toTypedExpr()` pattern, `RexConversionContext` with input attribute names
- **Spark Auron** (`AuronConverters` + `NativeConverters`): dispatch by plan type, expression → protobuf conversion, fail-safe try-convert semantics

**Approach B — Visitor-based tree rewriting**

Implement `AbstractExecNodeExactlyOnceVisitor` and use Flink's visitor pattern to walk the entire ExecNodeGraph, converting nodes inline.

- Tightly coupled to Flink's visitor API
- Conversion logic mixed with traversal logic
- Harder to test individual converters in isolation

### Decision: Approach A

The converter framework should be independent of how the graph is traversed. The future `ExecNodeGraphProcessor` that wires the framework into Flink's planner pipeline will handle traversal. The three classes in this PR focus on the **per-node conversion contract and dispatch**.

## 3. Detailed Design

### 3.1 Package Structure

All classes in `auron-flink-planner`, matching the planner module's role:

```
auron-flink-extension/auron-flink-planner/src/main/java/
  org/apache/auron/flink/table/planner/converter/
  ├── FlinkNodeConverter.java         (interface)
  ├── FlinkNodeConverterFactory.java  (singleton registry + dispatch)
  └── ConverterContext.java           (shared conversion state)
```

### 3.2 FlinkNodeConverter (Interface)

```java
package org.apache.auron.flink.table.planner.converter;

import org.apache.auron.protobuf.PhysicalPlanNode;
import org.apache.flink.table.planner.plan.nodes.exec.ExecNode;

/**
 * Defines the contract for converting a Flink {@link ExecNode} to an Auron
 * native {@link PhysicalPlanNode}.
 *
 * <p>Each implementation handles a specific ExecNode type (e.g., StreamExecCalc).
 * The converter checks whether a given node can be natively executed and, if so,
 * produces the equivalent native plan representation.
 *
 * <p>Implementations may need to convert not only the ExecNode itself but also
 * its internal expressions (e.g., RexNode projections, AggregateCall). The
 * {@link ConverterContext} provides the input schema needed for type-aware
 * expression conversion.
 */
public interface FlinkNodeConverter {

    /**
     * Returns the ExecNode class this converter handles.
     * Used by {@link FlinkNodeConverterFactory} for lookup dispatch.
     */
    Class<? extends ExecNode<?>> getExecNodeClass();

    /**
     * Checks whether the given node can be converted to native execution.
     *
     * <p>A converter may decline based on unsupported types, expressions,
     * or configuration. For example, a Calc converter may check whether all
     * projection expressions involve supported data types by inspecting the
     * input schema from the context. This method must not modify the node.
     */
    boolean isSupported(ExecNode<?> node, ConverterContext context);

    /**
     * Converts the given node to a native {@link PhysicalPlanNode}.
     *
     * <p>The returned protobuf represents the equivalent computation in
     * Auron's native engine. For nodes with expressions (e.g., Calc with
     * projections/conditions, Agg with AggregateCalls), the converter is
     * responsible for translating those expressions into the corresponding
     * native representation within the PhysicalPlanNode.
     *
     * @throws IllegalArgumentException if the node type does not match
     *         {@link #getExecNodeClass()}
     */
    PhysicalPlanNode convert(ExecNode<?> node, ConverterContext context);
}
```

**Design rationale**:

- **`getExecNodeClass()`** — Enables the factory to build a type-based lookup map. Each converter is registered for one ExecNode class. This is analogous to Spark's pattern-match dispatch in `AuronConverters.convertSparkPlan()` and Gluten's operator-name-keyed map in `RexCallConverterFactory`.

- **`isSupported(node, context)`** — Separated from `convert()` so the caller can check feasibility without side effects. Takes `ConverterContext` because support decisions depend on the **input schema** (are the input types supported for native execution?) and **configuration flags** (is this operator enabled?). This mirrors Gluten's `RexCallConverter.isSuitable(RexCall, context)`.

- **`convert(node, context)` returns `PhysicalPlanNode`** — The converter produces Auron's native plan protobuf, not a replacement Flink ExecNode. This is the key design decision informed by reviewer feedback and POC experience:
  - The real work is translating Flink plan semantics → native plan semantics
  - The native plan is what gets sent to the Rust engine via JNI
  - A future Auron-specific ExecNode wrapper (which holds the PhysicalPlanNode and executes it) is a separate concern handled by the graph processor
  - This matches Spark's pattern where `NativeProjectBase` builds `ProjectionExecNode` → `PhysicalPlanNode` protobuf internally
  - It also matches Gluten's pattern where `RexNodeConverter.toTypedExpr()` returns the native expression type directly

- **Expression conversion is the converter's responsibility** — A Calc converter must translate `List<RexNode> projection` and `RexNode condition` into native expressions within the `PhysicalPlanNode`. An Agg converter must translate `AggregateCall` instances. This is not limited to ExecNode-level conversion — the converter handles whatever internal structures the node contains.

### 3.3 ConverterContext

```java
package org.apache.auron.flink.table.planner.converter;

import org.apache.auron.configuration.AuronConfiguration;
import org.apache.flink.configuration.ReadableConfig;
import org.apache.flink.table.types.logical.RowType;

/**
 * Provides shared state to {@link FlinkNodeConverter} implementations
 * during conversion.
 *
 * <p>Carries the input schema, configuration, and classloader needed for
 * type-aware conversion of Flink plan nodes and their expressions.
 */
public class ConverterContext {

    private final ReadableConfig tableConfig;
    private final AuronConfiguration auronConfiguration;
    private final ClassLoader classLoader;
    private final RowType inputType;

    public ConverterContext(
            ReadableConfig tableConfig,
            AuronConfiguration auronConfiguration,
            ClassLoader classLoader,
            RowType inputType) { ... }

    /** Flink table-level configuration. */
    public ReadableConfig getTableConfig() { ... }

    /** Auron-specific configuration (batch size, memory fraction, enable flags, etc.). */
    public AuronConfiguration getAuronConfiguration() { ... }

    /** ClassLoader for the current Flink context. */
    public ClassLoader getClassLoader() { ... }

    /** Input schema of the node being converted. */
    public RowType getInputType() { ... }
}
```

**Design rationale**:

The context carries four things converters need:

| Field | Source | Why converters need it |
|-------|--------|----------------------|
| `RowType inputType` | From the ExecNode's input edge output type | Resolve `RexInputRef` column references to names/types; check type support; determine if casts are needed (e.g., math on incompatible types). Inspired by Gluten's `RexConversionContext.getInputAttributeNames()` but richer — `RowType` carries both names and `LogicalType`s. |
| `ReadableConfig tableConfig` | `PlannerBase.getTableConfig()` | Needed by `ExecNodeContext.newPersistedConfig()` when creating nodes |
| `AuronConfiguration auronConfiguration` | `AuronAdaptor.getInstance().getAuronConfiguration()` | Auron-specific settings (enable flags, batch size, etc.) |
| `ClassLoader classLoader` | `PlannerBase.getFlinkContext().getClassLoader()` | Needed by Flink's code generation and service loading |

**Why `RowType inputType`?** When converting expressions within a Calc node, the converter needs to know what types the input columns have. For example:
- `RexInputRef($0)` → need to know that column 0 is `INTEGER` to produce the right `PhysicalExprNode`
- A math expression `$0 + $1` → need to check both operand types are numeric
- Type mismatch → need to insert a cast in the native plan

This was identified as a critical gap in the original design by the reviewer based on POC experience.

**Why not hold `PlannerBase` directly?** The Flink `PlannerBase` is a heavyweight object tied to Flink's internal Calcite integration. Extracting only what converters need keeps the framework testable and decoupled.

### 3.4 FlinkNodeConverterFactory

```java
package org.apache.auron.flink.table.planner.converter;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

import org.apache.auron.protobuf.PhysicalPlanNode;
import org.apache.flink.table.planner.plan.nodes.exec.ExecNode;

/**
 * Singleton registry of {@link FlinkNodeConverter} instances. Dispatches
 * conversion requests to the appropriate converter based on the ExecNode's
 * concrete class.
 *
 * <p>Converters are registered via {@link #register(FlinkNodeConverter)}.
 * The singleton pattern ensures a single consistent registry across the
 * application, following the Gluten community's RexCallConverterFactory
 * pattern.
 *
 * <p>Usage:
 * <pre>
 *   FlinkNodeConverterFactory factory = FlinkNodeConverterFactory.getInstance();
 *   Optional&lt;PhysicalPlanNode&gt; result = factory.convertNode(node, context);
 * </pre>
 */
public class FlinkNodeConverterFactory {

    private static final FlinkNodeConverterFactory INSTANCE = new FlinkNodeConverterFactory();

    private final Map<Class<? extends ExecNode<?>>, FlinkNodeConverter> converterMap;

    private FlinkNodeConverterFactory() {
        this.converterMap = new HashMap<>();
    }

    /** Returns the singleton instance. */
    public static FlinkNodeConverterFactory getInstance() {
        return INSTANCE;
    }

    /**
     * Registers a converter for its declared ExecNode class.
     *
     * @throws IllegalArgumentException if a converter is already registered
     *         for the same ExecNode class
     */
    public void register(FlinkNodeConverter converter) {
        Class<? extends ExecNode<?>> nodeClass = converter.getExecNodeClass();
        FlinkNodeConverter existing = converterMap.put(nodeClass, converter);
        if (existing != null) {
            converterMap.put(nodeClass, existing); // restore
            throw new IllegalArgumentException(
                    "Duplicate converter for " + nodeClass.getName());
        }
    }

    /**
     * Attempts to convert the given node to a native PhysicalPlanNode.
     *
     * <p>Returns the native plan if a matching converter exists and
     * supports the node. Returns empty if:
     * <ul>
     *   <li>No converter is registered for the node's class</li>
     *   <li>The converter's {@code isSupported} returns false</li>
     *   <li>The converter throws an exception during conversion</li>
     * </ul>
     */
    public Optional<PhysicalPlanNode> convertNode(ExecNode<?> node, ConverterContext context) {
        FlinkNodeConverter converter = converterMap.get(node.getClass());
        if (converter == null) {
            return Optional.empty();
        }
        if (!converter.isSupported(node, context)) {
            return Optional.empty();
        }
        try {
            return Optional.of(converter.convert(node, context));
        } catch (Exception e) {
            // Log warning, return empty (fail-safe)
            return Optional.empty();
        }
    }

    /**
     * Returns the converter registered for the given node's class, if any.
     */
    public Optional<FlinkNodeConverter> getConverter(ExecNode<?> node) {
        return Optional.ofNullable(converterMap.get(node.getClass()));
    }
}
```

**Design rationale**:

- **Singleton** — Following Gluten's `RexCallConverterFactory` pattern. Converter registrations are compile-time constants, so a single global registry is natural. The future graph processor simply calls `FlinkNodeConverterFactory.getInstance()`. For testing, the singleton can be accessed and populated with test converters.

- **`register()` method** — Converters register themselves, rather than being passed as a list. This enables future converters (added in separate issues/PRs) to register in their own static initializers or via a startup hook without modifying the factory.

- **`convertNode()` returns `Optional<PhysicalPlanNode>`** — Returns `Optional.empty()` when conversion is not possible (no converter, not supported, or failure). This is cleaner than returning the original ExecNode — the caller knows unambiguously whether conversion succeeded and can decide what to do (e.g., keep the original Flink node for JVM execution).

- **Fail-safe** — If a converter throws during `convert()`, the exception is logged and `Optional.empty()` is returned. This prevents a buggy converter from breaking the entire query. Same pattern as Spark's `tryConvert()`.

- **Exact-class lookup** — Uses `node.getClass()` as the map key. A converter registered for `StreamExecCalc` will not match `BatchExecCalc` even though both extend `CommonExecCalc`. This is intentional: stream and batch nodes may have different semantics, and converters should be explicit.

## 4. Interaction Diagram

How these three classes fit into the future end-to-end flow:

```
┌──────────────────────────────────────────────────────────────────┐
│  Future ExecNodeGraphProcessor (NOT in this PR)                  │
│                                                                  │
│  1. Receives ExecNodeGraph from PlannerBase                      │
│  2. Walks graph via ExecNodeVisitor                              │
│  3. For each node:                                               │
│     a. Build ConverterContext (with node's input RowType)        │
│     b. factory.convertNode(node, context)                        │
│     c. If present: wrap PhysicalPlanNode in AuronExecNode        │
│     d. Reconnect input edges to replacement node                 │
│  4. Returns modified ExecNodeGraph                               │
└──────────────────────────────────────────────────────────────────┘
        │ uses                │ uses               │ uses
        ▼                     ▼                    ▼
┌──────────────┐   ┌──────────────────┐   ┌─────────────────────┐
│ConverterCtx  │   │ConverterFactory  │   │ FlinkNodeConverter  │
│              │   │   (singleton)    │   │    (interface)      │
│ inputType    │   │                  │   │                     │
│ tableConfig  │   │ getInstance()   │   │ getExecNodeClass()  │
│ auronConfig  │   │ register()      │   │ isSupported()       │
│ classLoader  │   │ convertNode()  │──▶│ convert()           │
│              │   │ getConverter()  │   │  → PhysicalPlanNode │
└──────────────┘   └──────────────────┘   └─────────────────────┘
                                                    △
                                                    │ implements
                                   ┌────────────────┼────────────────┐
                                   │                │                │
                            ┌──────┴─────┐  ┌──────┴─────┐  ┌──────┴─────┐
                            │ CalcConv   │  │ AggConv    │  │ FilterConv │
                            │ (future)   │  │ (future)   │  │ (future)   │
                            │            │  │            │  │            │
                            │ Converts:  │  │ Converts:  │  │ Converts:  │
                            │ RexNode    │  │ AggCall    │  │ RexNode    │
                            │ projection │  │ groups     │  │ condition  │
                            │ condition  │  │            │  │            │
                            └────────────┘  └────────────┘  └────────────┘
```

## 5. Prior Art Comparison

| Aspect | Spark Auron | Gluten Flink | This PR |
|--------|------------|-------------|---------|
| Converter interface | `AuronConvertProvider` | `RexCallConverter` | `FlinkNodeConverter` |
| Factory | `AuronConverters` (object) | `RexCallConverterFactory` (singleton) | `FlinkNodeConverterFactory` (singleton) |
| Discovery | ServiceLoader | Static registration | `register()` method |
| Context | Implicit static config | `RexConversionContext` (input attr names) | `ConverterContext` (input RowType + config) |
| Input schema | Not in context (accessed via SparkPlan) | `inputAttributeNames: List<String>` | `inputType: RowType` (names + types) |
| Return type | `SparkPlan` (replacement node) | `TypedExpr` (Velox native expr) | `PhysicalPlanNode` (Auron native protobuf) |
| Fail-safe | `tryConvert()` catches, falls back | Throws on unsupported | `Optional.empty()` on failure |
| Scope | ExecNode-level only | Expression-level (RexNode) | ExecNode-level (expressions handled within converter) |

**Key design choices informed by prior art**:
- **Singleton factory** — from Gluten's pattern, simpler than constructor injection
- **Input schema in context** — from Gluten's `RexConversionContext`, essential for type-aware expression conversion (reviewer's POC experience confirmed this)
- **Return `PhysicalPlanNode`** — from reviewer's POC experience; the real output is the native plan protobuf, matching Spark's internal pattern where native plan nodes build `PhysicalPlanNode` protobuf
- **`Optional` return** — improvement over both Spark (returns original node) and Gluten (throws); caller knows unambiguously whether conversion succeeded

## 6. Dependencies

### Build Dependencies (already in auron-flink-planner pom.xml)

- `auron-flink-runtime` — provides `AuronConfiguration`, `PhysicalPlanNode` (protobuf)
- `flink-table-planner` — provides `ExecNode`, `ExecNodeBase`
- `flink-table-runtime` — provides `RowData`, `ReadableConfig`
- JUnit Jupiter — test scope

### New Dependencies: None

### Code Dependencies

- `org.apache.flink.table.planner.plan.nodes.exec.ExecNode` — core interface for plan nodes
- `org.apache.flink.table.types.logical.RowType` — Flink's row type (input schema)
- `org.apache.flink.configuration.ReadableConfig` — Flink configuration interface
- `org.apache.auron.configuration.AuronConfiguration` — Auron configuration base class
- `org.apache.auron.protobuf.PhysicalPlanNode` — Auron's native plan protobuf

## 7. Test Plan

### 7.1 FlinkNodeConverterFactoryTest

| Test | What It Validates |
|------|-------------------|
| `testEmptyFactory` | Factory with no converters returns empty for any input |
| `testConverterDispatch` | Registered mock converter is called for matching node type |
| `testUnsupportedNodePassthrough` | Converter's `isSupported` returns false → empty returned |
| `testConversionFailureFallback` | Converter throws → empty returned (fail-safe) |
| `testDuplicateConverterRejected` | Two converters for same class → `IllegalArgumentException` |
| `testGetConverterPresent` | `getConverter` returns the registered converter |
| `testGetConverterAbsent` | `getConverter` returns empty for unregistered node type |

### 7.2 ConverterContextTest

| Test | What It Validates |
|------|-------------------|
| `testConstruction` | All four fields accessible via getters |
| `testNullTableConfigRejected` | Constructor rejects null `tableConfig` |
| `testNullClassLoaderRejected` | Constructor rejects null `classLoader` |
| `testNullInputTypeRejected` | Constructor rejects null `inputType` |

### Build & Run

```bash
./build/mvn test -pl auron-flink-extension/auron-flink-planner -am \
  -Pscala-2.12 -Pflink-1.18 -Pspark-3.5 -Prelease \
  -Dtest=FlinkNodeConverterFactoryTest,ConverterContextTest
```

## 8. Alternatives Considered

### 8.1 Return `ExecNode<?>` instead of `PhysicalPlanNode`

Have converters return a replacement Flink ExecNode rather than a native protobuf.

**Rejected because** (based on reviewer's POC experience):
- The real work is translating Flink semantics → native plan semantics
- The PhysicalPlanNode protobuf is what gets sent to Rust via JNI
- Creating a wrapper ExecNode (that holds the PhysicalPlanNode for execution) is a separate concern for the graph processor
- Matches Spark's internal pattern where native plan nodes produce PhysicalPlanNode protobuf

### 8.2 Visitor-based tree rewriter

Combine traversal and conversion into a single `AbstractExecNodeExactlyOnceVisitor` subclass.

**Rejected because**:
- Mixes traversal logic with conversion logic
- Hard to test converters in isolation
- The traversal strategy is a separate concern (future `ExecNodeGraphProcessor`)

### 8.3 Holding PlannerBase in ConverterContext

Pass the full `PlannerBase` object to converters.

**Rejected because**:
- Makes converters untestable without full Flink planner setup
- Exposes far more API surface than needed
- Couples to Flink's internal planner class (not a stable public API)

### 8.4 Constructor-injected factory (original design v1)

Pass converters as a `List` to the factory constructor.

**Revised to singleton** based on reviewer feedback and Gluten's pattern:
- Converter registrations are compile-time constants
- Singleton is simpler to use — no need to thread a factory instance through the call chain
- New converters in separate PRs can register themselves without modifying the factory setup code

### 8.5 Context without input schema (original design v1)

`ConverterContext` with only config and classloader.

**Revised to include `RowType inputType`** based on reviewer's POC experience:
- Expression conversion requires knowing input column types
- Type mismatches need cast insertion in the native plan
- `RexInputRef` resolution requires the input schema
- Matches Gluten's `RexConversionContext.getInputAttributeNames()` but richer (types + names)

## 9. Scope Boundaries

### In Scope
- `FlinkNodeConverter` interface
- `FlinkNodeConverterFactory` singleton class
- `ConverterContext` class (with input schema)
- Unit tests for factory and context

### Out of Scope (follow-up issues)
- `ExecNodeGraphProcessor` implementation (graph traversal + wiring)
- Actual `FlinkNodeConverter` implementations (Calc, Agg, Filter, etc.)
- Expression-level converter utilities (RexNode → PhysicalExprNode helpers)
- Flink configuration flags (e.g., `ENABLE_CALC`)
- `FlinkMetricNode` class
- Changes to existing runtime classes
