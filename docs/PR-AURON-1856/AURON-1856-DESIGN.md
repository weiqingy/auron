Does# Design: Flink Node Converter Tools (AURON #1856)

**Issue**: https://github.com/apache/auron/issues/1856
**AIP**: AIP-1 — Introduce Flink integration of native engine
**Author**: @weiqingy
**Status**: Draft

## 1. Motivation

Per AIP-1, the Flink integration requires a **planner-side framework** that intercepts Flink physical plan nodes (`ExecNode` tree) and replaces supported operators with native (Rust/DataFusion) execution. The data exchange layer is now in place:

```
Flink RowData → Arrow (Writer #1850) → Native Engine → Arrow → Flink RowData (Reader #1851 DONE)
```

What's missing is the **conversion infrastructure** — the machinery that decides *which* Flink nodes to replace, *how* to replace them, and *what context* converters need. This is analogous to Spark's `AuronConverters` + `AuronConvertProvider` pattern.

This issue delivers three foundational classes:

| Class | Role |
|-------|------|
| `FlinkNodeConverter` | Interface defining the contract for converting a specific ExecNode type |
| `FlinkNodeConverterFactory` | Registry that holds converters and dispatches to the right one |
| `ConverterContext` | Holds shared state (configuration, classloader) needed during conversion |

These classes are **framework only** — no actual converter implementations (e.g., Calc converter) are included. Subsequent issues will implement converters that plug into this framework.

## 2. Design Approach

### Two Candidate Approaches

**Approach A — Typed converter registry with factory dispatch (recommended)**

Each converter declares the `ExecNode` class it handles. The factory builds a `Map<Class, Converter>` lookup. When asked to convert a node, the factory finds the matching converter, checks support, and delegates.

- Clean separation of concerns — each converter is self-contained
- Easy to add new converters — just implement the interface and register
- Testable — converters and factory can be tested in isolation with mocks
- Matches the Spark pattern (`AuronConvertProvider.isSupported/convert`)

**Approach B — Visitor-based tree rewriting**

Implement `AbstractExecNodeExactlyOnceVisitor` and use Flink's visitor pattern to walk the entire ExecNodeGraph, replacing nodes inline.

- Tightly coupled to Flink's visitor API
- Conversion logic mixed with traversal logic
- Harder to test individual converters in isolation
- The graph walk is a **consumer** of the converter framework, not the framework itself

### Decision: Approach A

The converter framework should be independent of how the graph is traversed. The future `ExecNodeGraphProcessor` that wires the framework into Flink's planner pipeline will handle traversal. The three classes in this PR focus on the **per-node conversion contract and dispatch**.

This separation means:
1. The framework is testable without Flink's planner infrastructure
2. Converters can be developed and tested independently
3. The traversal strategy can change without touching converter code

## 3. Detailed Design

### 3.1 Package Structure

All classes in `auron-flink-planner`, matching the planner module's role:

```
auron-flink-extension/auron-flink-planner/src/main/java/
  org/apache/auron/flink/table/planner/converter/
  ├── FlinkNodeConverter.java         (interface)
  ├── FlinkNodeConverterFactory.java  (registry + dispatch)
  └── ConverterContext.java           (shared conversion state)
```

### 3.2 FlinkNodeConverter (Interface)

```java
package org.apache.auron.flink.table.planner.converter;

import org.apache.flink.table.planner.plan.nodes.exec.ExecNode;

/**
 * Defines the contract for converting a Flink {@link ExecNode} to a
 * native-executing replacement.
 *
 * <p>Each implementation handles a specific ExecNode type (e.g., StreamExecCalc).
 * The converter checks whether a given node can be natively executed and, if so,
 * produces a replacement node with the same output type and input edges.
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
     * or configuration. This method must not modify the node.
     */
    boolean isSupported(ExecNode<?> node, ConverterContext context);

    /**
     * Converts the given node to a native-executing replacement.
     *
     * <p>The replacement must have the same output type as the original and
     * accept the same inputs. Input edges are reconnected by the caller.
     *
     * @throws IllegalArgumentException if the node type does not match
     *         {@link #getExecNodeClass()}
     */
    ExecNode<?> convert(ExecNode<?> node, ConverterContext context);
}
```

**Design rationale**:

- **`getExecNodeClass()`** — Enables the factory to build a type-based lookup map at construction time. Each converter is registered exactly once for one ExecNode class. This is analogous to Spark's pattern-match dispatch in `AuronConverters.convertSparkPlan()` where each `case e: ProjectExec =>` branch is effectively a converter for that type.

- **`isSupported(node, context)`** — Separated from `convert()` so the caller can check feasibility without side effects. This enables a future strategy pass (like Spark's `AuronConvertStrategy`) that tags nodes as convertible before actually converting them. Takes `ConverterContext` because support decisions often depend on configuration flags (e.g., `ENABLE_CALC`) and type support checks.

- **`convert(node, context)`** — Returns a new `ExecNode<?>` that replaces the original. The replacement is a **different node instance** (replacement pattern, not wrapping), because `ExecNodeBase.translateToPlan()` caches its `Transformation` result and the native execution path requires a fundamentally different `Transformation`.

### 3.3 ConverterContext

```java
package org.apache.auron.flink.table.planner.converter;

import org.apache.auron.configuration.AuronConfiguration;
import org.apache.flink.configuration.ReadableConfig;

/**
 * Provides shared state to {@link FlinkNodeConverter} implementations
 * during conversion.
 *
 * <p>Extracts only what converters need from the Flink planner, keeping
 * the converter framework decoupled from Flink internals and testable.
 */
public class ConverterContext {

    private final ReadableConfig tableConfig;
    private final AuronConfiguration auronConfiguration;
    private final ClassLoader classLoader;

    public ConverterContext(
            ReadableConfig tableConfig,
            AuronConfiguration auronConfiguration,
            ClassLoader classLoader) { ... }

    /** Flink table-level configuration. */
    public ReadableConfig getTableConfig() { ... }

    /** Auron-specific configuration (batch size, memory fraction, etc.). */
    public AuronConfiguration getAuronConfiguration() { ... }

    /** ClassLoader for the current Flink context. */
    public ClassLoader getClassLoader() { ... }
}
```

**Design rationale — why not hold `PlannerBase` directly?**

The Flink `PlannerBase` is a heavyweight object deeply tied to Flink's internal Calcite integration. Holding it in `ConverterContext` would:
1. Make converters untestable without spinning up a full Flink planner
2. Couple the framework to Flink's planner internals (which change across versions)
3. Expose far more API surface than converters actually need

Instead, we extract the three things converters need:

| Field | Source | Why converters need it |
|-------|--------|----------------------|
| `ReadableConfig tableConfig` | `PlannerBase.getTableConfig()` | Needed by `ExecNodeContext.newPersistedConfig()` when creating replacement nodes |
| `AuronConfiguration auronConfiguration` | `AuronAdaptor.getInstance().getAuronConfiguration()` | Auron-specific settings (enable flags, batch size, etc.) |
| `ClassLoader classLoader` | `PlannerBase.getFlinkContext().getClassLoader()` | Needed by Flink's code generation and service loading |

The future `ExecNodeGraphProcessor` that instantiates `ConverterContext` will extract these from `ProcessorContext.getPlanner()`.

### 3.4 FlinkNodeConverterFactory

```java
package org.apache.auron.flink.table.planner.converter;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import org.apache.flink.table.planner.plan.nodes.exec.ExecNode;

/**
 * Registry of {@link FlinkNodeConverter} instances. Dispatches conversion
 * requests to the appropriate converter based on the ExecNode's concrete class.
 *
 * <p>Usage:
 * <pre>
 *   List<FlinkNodeConverter> converters = List.of(new CalcConverter(), ...);
 *   FlinkNodeConverterFactory factory = new FlinkNodeConverterFactory(converters);
 *   ExecNode<?> result = factory.convertNode(node, context);
 * </pre>
 */
public class FlinkNodeConverterFactory {

    private final Map<Class<? extends ExecNode<?>>, FlinkNodeConverter> converterMap;

    /**
     * Creates a factory with the given converters.
     * Each converter is registered for the ExecNode class it declares.
     *
     * @throws IllegalArgumentException if two converters declare the same class
     */
    public FlinkNodeConverterFactory(List<FlinkNodeConverter> converters) {
        this.converterMap = new HashMap<>();
        for (FlinkNodeConverter converter : converters) {
            FlinkNodeConverter existing =
                    converterMap.put(converter.getExecNodeClass(), converter);
            if (existing != null) {
                throw new IllegalArgumentException(
                        "Duplicate converter for " + converter.getExecNodeClass().getName());
            }
        }
    }

    /**
     * Attempts to convert the given node to native execution.
     *
     * <p>Returns the converted node if a matching converter exists and
     * supports the node. Returns the original node unchanged if:
     * <ul>
     *   <li>No converter is registered for the node's class</li>
     *   <li>The converter's {@code isSupported} returns false</li>
     *   <li>The converter throws an exception during conversion</li>
     * </ul>
     */
    public ExecNode<?> convertNode(ExecNode<?> node, ConverterContext context) {
        FlinkNodeConverter converter = converterMap.get(node.getClass());
        if (converter == null) {
            return node;
        }
        if (!converter.isSupported(node, context)) {
            return node;
        }
        try {
            return converter.convert(node, context);
        } catch (Exception e) {
            // Log warning, return original node unchanged (fail-safe)
            return node;
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

- **Constructor injection** — Converters are passed in as a list, not discovered via ServiceLoader. This keeps the factory simple and testable. ServiceLoader-based discovery can be added in a follow-up if extensibility beyond compile-time registration is needed.

- **Exact-class lookup** — Uses `node.getClass()` as the map key. This means a converter registered for `StreamExecCalc` will not match `BatchExecCalc` even though both extend `CommonExecCalc`. This is intentional: stream and batch nodes have different semantics (e.g., `retainHeader`), and converters should be explicit about what they handle. If a single converter should handle both, it can be registered twice with different `getExecNodeClass()` return values (via two instances or a parameterized constructor).

- **Fail-safe `convertNode()`** — If a converter throws during `convert()`, the original node is returned unchanged and the exception is logged. This prevents a buggy converter from breaking the entire query. The same pattern is used in Spark's `tryConvert()` (`AuronConverters.scala:141`).

- **Duplicate detection** — The constructor throws if two converters claim the same ExecNode class. This catches misconfiguration early rather than silently shadowing.

## 4. Interaction Diagram

How these three classes fit into the future end-to-end flow:

```
┌─────────────────────────────────────────────────────────────┐
│  Future ExecNodeGraphProcessor (NOT in this PR)             │
│                                                             │
│  1. Receives ExecNodeGraph from PlannerBase                 │
│  2. Creates ConverterContext from ProcessorContext           │
│  3. Creates FlinkNodeConverterFactory with registered       │
│     converters (e.g., CalcConverter)                        │
│  4. Walks graph via ExecNodeVisitor                         │
│  5. For each node: factory.convertNode(node, context)       │
│  6. Reconnects input edges to replacement nodes             │
│  7. Returns modified ExecNodeGraph                          │
└─────────────────────────────────────────────────────────────┘
        │ uses                │ uses               │ uses
        ▼                     ▼                    ▼
┌──────────────┐   ┌──────────────────┐   ┌────────────────┐
│ConverterCtx  │   │ConverterFactory  │   │ FlinkNodeConv  │
│              │   │                  │   │  (interface)   │
│ tableConfig  │   │ converterMap     │   │ getExecNodeCls │
│ auronConfig  │   │ convertNode()   │──▶│ isSupported()  │
│ classLoader  │   │ getConverter()  │   │ convert()      │
└──────────────┘   └──────────────────┘   └────────────────┘
                                                  △
                                                  │ implements
                                          ┌───────┴────────┐
                                          │ CalcConverter   │
                                          │ (future issue)  │
                                          └────────────────┘
```

## 5. Prior Art Comparison

| Aspect | Spark Pattern | Flink Design (this PR) |
|--------|--------------|----------------------|
| Converter interface | `AuronConvertProvider` (Scala trait) | `FlinkNodeConverter` (Java interface) |
| Dispatch | Pattern matching in `convertSparkPlan()` | Map lookup in `FlinkNodeConverterFactory.convertNode()` |
| Discovery | `ServiceLoader.load(AuronConvertProvider)` | Constructor injection (ServiceLoader as follow-up) |
| Context | Implicit — config accessed via static `SparkAuronConfiguration` | Explicit — `ConverterContext` passed to every method |
| Fail-safe | `tryConvert()` catches exceptions, falls back | `convertNode()` catches exceptions, returns original |
| Support check | `isEnabled` + `isSupported` on provider | `isSupported(node, context)` on converter |
| Node replacement | Returns new `SparkPlan` subclass extending `NativeSupports` | Returns new `ExecNode<?>` (details in future converter PRs) |

**Key improvement over Spark**: Explicit `ConverterContext` instead of static/global config access. This makes the framework testable and avoids hidden dependencies.

## 6. Dependencies

### Build Dependencies (already in auron-flink-planner pom.xml)

- `auron-flink-runtime` — provides `AuronConfiguration`, `SupportsAuronNative`
- `flink-table-planner` — provides `ExecNode`, `ExecNodeBase`
- `flink-table-runtime` — provides `RowData`, `ReadableConfig`
- JUnit Jupiter — test scope

### New Dependencies: None

### Code Dependencies

- `org.apache.flink.table.planner.plan.nodes.exec.ExecNode` — core interface for plan nodes
- `org.apache.flink.configuration.ReadableConfig` — Flink configuration interface
- `org.apache.auron.configuration.AuronConfiguration` — Auron configuration base class

## 7. Test Plan

### 7.1 FlinkNodeConverterFactoryTest

| Test | What It Validates |
|------|-------------------|
| `testEmptyFactory` | Factory with no converters returns original node for any input |
| `testConverterDispatch` | Registered mock converter is called for matching node type |
| `testUnsupportedNodePassthrough` | Converter's `isSupported` returns false → original returned |
| `testConversionFailureFallback` | Converter throws → original returned (fail-safe) |
| `testDuplicateConverterRejected` | Two converters for same class → `IllegalArgumentException` |
| `testGetConverterPresent` | `getConverter` returns the registered converter |
| `testGetConverterAbsent` | `getConverter` returns empty for unregistered node type |

Tests will use simple mock/stub implementations of `FlinkNodeConverter` and `ExecNode` to avoid requiring a full Flink planner environment.

### 7.2 ConverterContextTest

| Test | What It Validates |
|------|-------------------|
| `testConstruction` | All three fields accessible via getters |
| `testNullChecks` | Constructor rejects null `tableConfig` and `classLoader` |

### Build & Run

```bash
./build/mvn test -pl auron-flink-extension/auron-flink-planner -am \
  -Pscala-2.12 -Pflink-1.18 -Pspark-3.5 -Prelease \
  -Dtest=FlinkNodeConverterFactoryTest,ConverterContextTest
```

## 8. Alternatives Considered

### 8.1 Visitor-based tree rewriter (Approach B)

Combine traversal and conversion into a single `AbstractExecNodeExactlyOnceVisitor` subclass.

**Rejected because**:
- Mixes traversal logic with conversion logic
- Hard to test converters in isolation
- The traversal strategy is a separate concern (future `ExecNodeGraphProcessor`)
- Converter implementations would be tightly coupled to Flink's visitor API

### 8.2 Holding PlannerBase in ConverterContext

Pass the full `PlannerBase` object to converters.

**Rejected because**:
- Makes converters untestable without full Flink planner setup
- Exposes far more API surface than needed
- Couples to Flink's internal planner class (not a stable public API)

### 8.3 Static converter registry (singleton pattern)

Use a global static map of converters instead of a factory instance.

**Rejected because**:
- Global state makes testing hard
- Multiple converter configurations cannot coexist
- Thread-safety concerns in concurrent environments
- Goes against the explicit-dependency pattern used elsewhere in Auron

### 8.4 ServiceLoader-based discovery in this PR

Automatically discover converters via `META-INF/services`.

**Deferred** (not rejected): ServiceLoader discovery is useful for extensibility but adds complexity. The constructor-injection approach is sufficient for Phase 1 where converters are known at compile time. ServiceLoader can be added as a convenience in a follow-up PR.

## 9. Scope Boundaries

### In Scope
- `FlinkNodeConverter` interface
- `FlinkNodeConverterFactory` class
- `ConverterContext` class
- Unit tests for factory and context

### Out of Scope (follow-up issues)
- `ExecNodeGraphProcessor` implementation (graph traversal + wiring)
- Actual `FlinkNodeConverter` implementations (Calc, Filter, etc.)
- ServiceLoader SPI registration for extensible discovery
- Flink configuration flags (e.g., `ENABLE_CALC`)
- `FlinkMetricNode` class
- Changes to existing runtime classes
