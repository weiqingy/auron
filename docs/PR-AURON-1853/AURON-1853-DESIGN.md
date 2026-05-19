# Design — AURON-1853: Convert Flink `StreamExecCalc` to Native Calc

**Author**: weiqingy
**Date**: 2026-05-19 (initial) — **Rev 2**: 2026-05-19 (simplified per reviewer feedback: drop factory + helper classes)
**Status**: Rev 2 — pending reviewer feedback
**Issue**: https://github.com/apache/auron/issues/1853
**Depends on**: #1856 (converter framework, merged), #1859 (RexNode converters, merged), #1857 (FlinkAuronCalcOperator, PR #2263 merged 2026-05-18)
**Unblocks**: #1860, #1861, #1862, #1863, #1864, #1865

---

## Rev 2 Changes (2026-05-19)

Two simplifications in response to reviewer feedback on Rev 1:

**Q1 (drop the factory)**: `FlinkAuronCalcOperatorFactory` removed. `OneInputTransformation` already accepts an `OneInputStreamOperator` directly (verified constructor `OneInputTransformation(Transformation, String, OneInputStreamOperator, TypeInformation, int)` in `flink-streaming-java-1.18.1.jar`), and Flink wraps it internally in `SimpleOperatorFactory`. Constructing `FlinkAuronCalcOperator` inline matches Gluten's pattern in `gluten-flink/.../stream/StreamExecCalc.java` (`new GlutenOneInputOperator(...)` passed directly to `ExecNodeUtil.createOneInputTransformation`). The custom factory was over-engineering.

**Q2 (drop the bespoke helpers)**: `RexProgramToPlanBuilder` and `AuronCalcConversionResult` removed. The plan-build logic moves inline into the shadowed `StreamExecCalc.translateToPlanInternal`. The shared abstraction the design relies on is the converter framework (`FlinkNodeConverterFactory` + `FlinkRexNodeConverter`), which is already universal across operators (#1860/#1861/#1864/#1865 reuse it as-is). Fallback signal becomes `Optional<PhysicalPlanNode>` returned from a small private helper inside the shadowed class — no custom sum type.

**Net result**:

| | Rev 1 | Rev 2 |
|---|---|---|
| Files created | 5 (StreamExecCalc + factory + builder + result + tests) | **1** (StreamExecCalc) |
| Files modified | 1 (`FlinkNodeConverterFactory` static initializer) + 1 (`FlinkAuronConfiguration` config option) | same (2) |
| Test classes | 4 (factory + builder + StreamExecCalc + ITCase) | **2** (StreamExecCalc + ITCase) |
| Lines of new code | ~600 | ~250 (estimate) |

All Rev 1 architectural decisions preserved: JAR shadowing, plan shape `Project[Filter?[FFIReader-placeholder]]`, `FAIL_BACK_FLINK_ENGINE_ENABLED` config (default `true`), per-Calc `super.translateToPlanInternal` fallback, runtime resource-ID rewrite at operator's `open()`, identity from `ExecNode.getId()`.

---

## Problem Statement

`FlinkAuronCalcOperator` (#1857, merged) executes a native `Project[Filter[FFIReader]]` plan but is **unreachable from real Flink SQL jobs** — the job graph today contains Flink's own `StreamExecCalc`, which builds a JVM-codegen operator via `CodeGenOperatorFactory<RowData>` instead of the Auron operator. AURON-1853 closes the loop: at job-submission time, detect `StreamExecCalc` instances whose `projection` and `condition` `RexNode`s are entirely Auron-supported, replace the JVM operator with `FlinkAuronCalcOperator`, and fall back transparently to the JVM operator whenever any RexNode is unsupported. This is the **graph-rewriter layer** of the three-layer architecture established in #1857 (plan-layer #1856/#1859 + operator-layer #1857 + **graph-rewriter** #1853).

End state after merge: a `SELECT a + b FROM t` query passes through Auron's native arithmetic instead of Flink's codegen-Calc bytecode. The first end-to-end Flink-on-Auron UT becomes possible, validating #1850/#1851/#1856/#1857/#1859 together.

---

## Approach Candidates

Three candidates were considered. The trade-off is between **invasiveness** (how deep into Flink's namespace we reach), **UX** (does the user need to set a config?), and **the issue text's literal direction** ("rewrite the Flink `StreamExecCalc` class").

### Approach A — Shadow `StreamExecCalc` in Flink's package (Gluten's pattern) — **CHOSEN**

Ship a class at FQCN `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc` inside `auron-flink-planner`. Java's classloader resolves a single class per FQCN; whichever JAR sits first on the classpath wins. With `auron-flink-planner` placed ahead of `flink-table-planner` (the normal case for Auron-enabled Flink deployments), Flink's planner constructs Auron's shadowed class whenever it builds a Calc ExecNode.

The shadowed class extends `CommonExecCalc` (Flink's parent, in `org.apache.flink.table.planner.plan.nodes.exec.common`), keeps Flink's two public constructors and `@ExecNodeMetadata(name="stream-exec-calc", version=1, minPlanVersion=v1_15, minStateVersion=v1_15)` annotation byte-for-byte, and overrides `translateToPlanInternal(PlannerBase, ExecNodeConfig)`. The override builds the Auron `PhysicalPlanNode` inline via #1859's converter framework; on success it constructs a `FlinkAuronCalcOperator` and returns a `OneInputTransformation` wrapping it; on **any** failure (unsupported RexNode, conversion exception, schema mismatch) it delegates to `super.translateToPlanInternal(planner, config)` per the `FAIL_BACK_FLINK_ENGINE_ENABLED` config, which produces Flink's stock codegen operator unchanged.

**Pros**:
- Matches the issue text literally ("rewrite the Flink `StreamExecCalc` class").
- Direct prior art in Apache Gluten's `gluten-flink/` (production-tested at scale).
- No user configuration required — works once the JAR is on the classpath in the standard order.
- Fallback is **per-Calc + free** — we just call `super.translateToPlanInternal`.
- No subclassing of Flink's `Planner` / `PlannerFactory`; no custom factory SPI.
- Preserves chaining for free — same return type (`Transformation<RowData>`), same outer wiring; the factory inherits `ChainingStrategy.ALWAYS` from the operator's `TableStreamOperator` parent (#1857 P6).

**Cons**:
- Places Auron code in `org.apache.flink.table.planner.*` namespace — uncomfortable but standard practice in this corner of the Flink ecosystem. Gluten, Apache Iceberg's Flink connector, and several other projects use the same pattern when no clean extension hook exists.
- Classpath ordering must put `auron-flink-planner` ahead of `flink-table-planner`. Documented in the auron-flink-assembly module README and reinforced by the `auron-flink-assembly` packaging order.
- Breaks if Flink ever changes `StreamExecCalc`'s constructor signature or `CommonExecCalc.translateToPlanInternal`'s signature. Both signatures are stable since Flink 1.15 (annotated via `minPlanVersion=v1_15`).
- Tests must verify the shadowed class loads when both JARs are on the classpath.

### Approach B — Custom `PlannerFactory` SPI + subclassed `StreamPlanner` — REJECTED

Register an `AuronPlannerFactory` via `META-INF/services/org.apache.flink.table.factories.Factory` returning an `AuronStreamPlanner extends StreamPlanner` that overrides `getExecNodeGraphProcessors()` to return `super.getExecNodeGraphProcessors() :+ new AuronCalcRewriteProcessor()`. The processor walks the `ExecNodeGraph`, finds `StreamExecCalc` instances, and substitutes a custom ExecNode that returns a Transformation wired to a substituted operator.

**Why rejected**:
- `getExecNodeGraphProcessors()` returns `scala.collection.Seq<...>` from a hardcoded method body inside `StreamPlanner` (verified via `javap` on `flink-table-planner_2.12-1.18.1.jar`). It is overridable, but only via subclassing the planner.
- `PlannerFactory` SPI uses `factoryIdentifier()` for selection. Registering a duplicate `"default"` identifier collides with Flink's `DefaultPlannerFactory`. Registering a new identifier (e.g. `"auron"`) forces every user to configure `table.planner=auron`, breaking drop-in usage.
- Subclassing `StreamPlanner` is heavyweight; the parent's constructor is non-trivial and the surface area to maintain across Flink versions is large.
- No production precedent in this style for Flink (the documented `ExecNodeGraphProcessor` extension point has no SPI registration mechanism in 1.18).

### Approach C — Pure ExecNodeGraph traversal via reflection — REJECTED

Reflectively add our processor to `StreamPlanner.getExecNodeGraphProcessors()`'s returned Seq at runtime via setAccessible/private-field mutation, or use a bytecode-rewriter library at startup. Both are unreliable across Flink versions and disallowed by `## Constraints` (no bytecode-rewriting libs, no reflection-into-Flink-internals beyond reading public-shaped fields).

---

## Decision

**Adopt Approach A — Shadow `StreamExecCalc` in `org.apache.flink.table.planner.plan.nodes.exec.stream`.**

Rationale, in order of weight:

1. **Issue text direction**: "Rewrite the Flink `StreamExecCalc` class to enable conversion…" — the natural reading is class-level shadowing-style substitution.
2. **Flink lacks a clean extension hook for this**: `ExecNodeGraphProcessor` is the only documented hook, but `flink-table-planner_2.12-1.18.1.jar` ships **no `META-INF/services/...ExecNodeGraphProcessor` registration file** — the processor list is hardcoded inside `StreamPlanner.getExecNodeGraphProcessors()`. Verified via `javap`. Approach B (a custom `PlannerFactory` SPI subclass) is possible but introduces a planner-identifier collision or forces a user config (`table.planner=auron`), which breaks drop-in usage. There is no third clean hook.
3. **Reviewer alignment**: @Tartarus0zm's #1857 forward-looking note ("how the graph-rewriter substitutes Flink's codegen factory… with one that constructs `FlinkAuronCalcOperator`") is satisfied cleanly by Approach A — the new factory replaces `CodeGenOperatorFactory<RowData>` inside the shadowed `translateToPlanInternal`.
4. **Production precedent**: Apache Gluten ships this exact mechanism for the identical use case (`gluten-flink/planner/src/main/java/org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.java`). It is the de-facto extension pattern in this corner of the Flink ecosystem and has been validated at scale.
5. **Per-Calc fallback is built in for free** — `super.translateToPlanInternal(planner, config)` produces unchanged Flink behavior whenever our conversion path bails. One method call's overhead.
6. **Chaining preserved naturally** — same return type, same return path, same factory contract.
7. **No user configuration to enable substitution** — drop-in once the JAR is present in the standard `auron-flink-assembly` bundle. (`FAIL_BACK_FLINK_ENGINE_ENABLED` controls fallback *behavior* on conversion failure, not whether substitution is attempted.)

The compromises (namespace pollution, classpath ordering, signature-stability dependency) are real but bounded: confined to one class, one method, and an annotation that's been stable since Flink 1.15.

---

## Detailed Design

### Three-layer placement

```
┌──────────────────────────────────────────────────────────────────────┐
│ GRAPH-REWRITER LAYER  (★ #1853 — this PR ★)                          │
│                                                                       │
│  Shadowed class: org.apache.flink.table.planner.plan.nodes.exec       │
│                  .stream.StreamExecCalc  (in auron-flink-planner)     │
│                                                                       │
│  Override: translateToPlanInternal(PlannerBase, ExecNodeConfig)       │
│      1. Try to build PhysicalPlanNode via converters                  │
│      2. On success: return OneInputTransformation wired to            │
│                    constructed FlinkAuronCalcOperator                 │
│      3. On any failure: super.translateToPlanInternal(...)            │
└─────────┬─────────────────────────────────────┬───────────────────────┘
          │ calls                                │ calls
          ▼                                      ▼
┌─────────────────────────────────┐   ┌─────────────────────────────┐
│ PLAN LEVEL  (#1856, #1859 done) │   │ OPERATOR LEVEL  (#1857 done)│
│ FlinkNodeConverterFactory       │   │ FlinkAuronCalcOperator      │
│ RexNode → PhysicalExprNode      │   │                             │
└─────────────────────────────────┘   └─────────────────────────────┘
```

### Class layout

**One** new file in `auron-flink-planner`:

```
auron-flink-planner/src/main/java/
└── org/apache/flink/table/planner/plan/nodes/exec/stream/
    └── StreamExecCalc.java         (shadowed — same FQCN as Flink's; inline plan build + fallback)
```

**Two** files modified:
```
auron-flink-planner/src/main/java/org/apache/auron/flink/table/planner/converter/
└── FlinkNodeConverterFactory.java          (add static initializer registering the 3 built-in converters)

auron-flink-extension/auron-flink-runtime/src/main/java/org/apache/auron/flink/configuration/
└── FlinkAuronConfiguration.java            (add FAIL_BACK_FLINK_ENGINE_ENABLED config option)
```

Tests:
```
auron-flink-planner/src/test/java/org/apache/flink/table/planner/plan/nodes/exec/stream/
└── StreamExecCalcTest.java                  (shadowing verification + plan-build paths + fallback config)

auron-flink-planner/src/test/java/.../runtime/
└── AuronCalcRewriteITCase.java              (E2E SQL: TestValuesTableFactory → Calc → Sink)
```

### File 1 — Shadowed `StreamExecCalc` (all logic inline)

```java
// org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc
// Lives in auron-flink-planner; shadows Flink's class via classpath ordering.

@ExecNodeMetadata(
    name = "stream-exec-calc",
    version = 1,
    minPlanVersion = FlinkVersion.v1_15,
    minStateVersion = FlinkVersion.v1_15)
public class StreamExecCalc extends CommonExecCalc
        implements StreamExecNode<RowData> {

    private static final Logger LOG = LoggerFactory.getLogger(StreamExecCalc.class);

    public StreamExecCalc(
            ReadableConfig tableConfig,
            List<RexNode> projection,
            @Nullable RexNode condition,
            InputProperty inputProperty,
            RowType outputType,
            String description) {
        super(
                ExecNodeContext.newNodeId(),
                ExecNodeContext.newContext(StreamExecCalc.class),
                tableConfig,
                projection,
                condition,
                /*operatorBaseClass=*/ TableStreamOperator.class,
                /*retainHeader=*/ false,
                Collections.singletonList(inputProperty),
                outputType,
                description);
    }

    @JsonCreator
    public StreamExecCalc(
            @JsonProperty(FIELD_NAME_ID) int id,
            @JsonProperty(FIELD_NAME_TYPE) ExecNodeContext context,
            @JsonProperty(FIELD_NAME_CONFIGURATION) ReadableConfig persistedConfig,
            @JsonProperty(FIELD_NAME_PROJECTION) List<RexNode> projection,
            @Nullable @JsonProperty(FIELD_NAME_CONDITION) RexNode condition,
            @JsonProperty(FIELD_NAME_INPUT_PROPERTIES) List<InputProperty> inputProperties,
            @JsonProperty(FIELD_NAME_OUTPUT_TYPE) RowType outputType,
            @JsonProperty(FIELD_NAME_DESCRIPTION) String description) {
        super(
                id, context, persistedConfig, projection, condition,
                TableStreamOperator.class, false, inputProperties, outputType, description);
    }

    @Override
    @SuppressWarnings("unchecked")
    protected Transformation<RowData> translateToPlanInternal(
            PlannerBase planner, ExecNodeConfig config) {
        Transformation<RowData> upstream =
                (Transformation<RowData>) getInputEdges().get(0).translate(planner);
        RowType inputRowType = (RowType) getInputEdges().get(0).getOutputType();
        RowType outputRowType = (RowType) getOutputType();

        Optional<PhysicalPlanNode> plan = tryBuildAuronPlan(inputRowType, outputRowType);

        if (!plan.isPresent()) {
            boolean fallbackEnabled = AuronAdaptor.getInstance()
                    .getAuronConfiguration()
                    .get(FlinkAuronConfiguration.FAIL_BACK_FLINK_ENGINE_ENABLED);
            if (fallbackEnabled) {
                LOG.debug("Falling back to Flink's CodeGen Calc for node {}", getId());
                return super.translateToPlanInternal(planner, config);
            }
            throw new IllegalStateException(
                    "Auron Calc conversion failed for node " + getId()
                    + " and fallback is disabled");
        }

        FlinkAuronCalcOperator operator = new FlinkAuronCalcOperator(
                plan.get(), inputRowType, outputRowType, "FlinkAuronCalc-" + getId());

        return ExecNodeUtil.createOneInputTransformation(
                upstream,
                createTransformationMeta(CALC_TRANSFORMATION, config),
                operator,
                InternalTypeInfo.of(outputRowType),
                upstream.getParallelism(),
                /*memoryBytes=*/ 0);
    }

    /**
     * Attempts to compose a native plan from this Calc's projection and condition.
     * Returns empty if any RexNode is unsupported by the converter framework, or if
     * plan composition throws.
     */
    private Optional<PhysicalPlanNode> tryBuildAuronPlan(
            RowType inputRowType, RowType outputRowType) {
        try {
            ConverterContext ctx = new ConverterContext(
                    getPersistedConfig(),
                    AuronAdaptor.getInstance().getAuronConfiguration(),
                    Thread.currentThread().getContextClassLoader(),
                    inputRowType);
            FlinkNodeConverterFactory converters = FlinkNodeConverterFactory.getInstance();

            // 1. Convert filter (if any)
            PhysicalExprNode filterExpr = null;
            if (condition != null) {
                Optional<PhysicalExprNode> c = converters.convertRexNode(condition, ctx);
                if (!c.isPresent()) {
                    return Optional.empty();
                }
                filterExpr = c.get();
            }

            // 2. Convert projection
            List<PhysicalExprNode> projectExprs = new ArrayList<>(projection.size());
            for (RexNode rex : projection) {
                Optional<PhysicalExprNode> c = converters.convertRexNode(rex, ctx);
                if (!c.isPresent()) {
                    return Optional.empty();
                }
                projectExprs.add(c.get());
            }

            // 3. Compose: Project[Filter?[FFIReader-placeholder]]
            FFIReaderExecNode ffiReader = FFIReaderExecNode.newBuilder()
                    .setNumPartitions(1)
                    .setSchema(SchemaConverters.convertToAuronSchema(inputRowType, false))
                    .setExportIterProviderResourceId(
                            FlinkAuronCalcOperator.RESOURCE_ID_PLACEHOLDER)
                    .build();
            PhysicalPlanNode current = PhysicalPlanNode.newBuilder().setFfiReader(ffiReader).build();

            if (filterExpr != null) {
                FilterExecNode filterNode = FilterExecNode.newBuilder()
                        .setInput(current).addExpr(filterExpr).build();
                current = PhysicalPlanNode.newBuilder().setFilter(filterNode).build();
            }

            ProjectionExecNode.Builder proj = ProjectionExecNode.newBuilder().setInput(current);
            for (int i = 0; i < projectExprs.size(); i++) {
                proj.addExpr(projectExprs.get(i));
                proj.addExprName(outputRowType.getFieldNames().get(i));
                proj.addDataType(SchemaConverters.convertToAuronArrowType(outputRowType.getTypeAt(i)));
            }
            return Optional.of(PhysicalPlanNode.newBuilder().setProjection(proj.build()).build());

        } catch (Throwable t) {
            // Defense-in-depth: converter framework catches Exception per #1859, but
            // schema conversion or proto composition could still throw (e.g. unsupported
            // LogicalType in SchemaConverters). Treat any failure as fallback.
            LOG.debug("Auron Calc plan composition threw for node {}", getId(), t);
            return Optional.empty();
        }
    }
}
```

**Plan-shape rule**: always emits `Project[Filter?[FFIReader-placeholder]]`. The `Project` is unconditional even when the projection is identity, because `FlinkAuronCalcOperator.injectFfiReaderLeaf` accepts `Project[FFIReader]` and `Project[Filter[FFIReader]]`; the bare-`FFIReader` and `Filter[FFIReader]` shapes from #1857's contract aren't produced by a normal Flink Calc (Calc always has a projection).

**Identity projection** — Flink may produce a Calc that re-emits all input columns unchanged. The Projection's expressions are `RexInputRef`s with the same indices, which #1859's `RexInputRefConverter` handles. No special case needed.

**Why `Throwable` in the catch**: defense-in-depth. The converter framework catches per-RexNode `Exception` and returns `Optional.empty()`. The outer net handles `AssertionError` from Calcite (rare but observed) plus any `RuntimeException` from `SchemaConverters` on an unsupported `LogicalType`. Fallback is the safe default for **any** failure.

**Note on `FlinkAuronCalcOperator.RESOURCE_ID_PLACEHOLDER`**: this constant doesn't exist in #1857-merged code yet. Either add a 1-line `public static final String RESOURCE_ID_PLACEHOLDER = "placeholder"` to the operator class (reviewer OK?), or hardcode the literal `"placeholder"` here. `injectFfiReaderLeaf` doesn't validate the placeholder value, so either path works; the constant just keeps the contract co-located with the operator.

### File 2 — modify `FlinkNodeConverterFactory`

Add a static initializer that registers the three built-in converters once:

```java
public class FlinkNodeConverterFactory {
    private static final FlinkNodeConverterFactory INSTANCE = new FlinkNodeConverterFactory();
    static {
        INSTANCE.registerRexConverter(new RexInputRefConverter());
        INSTANCE.registerRexConverter(new RexLiteralConverter());
        INSTANCE.registerRexConverter(new RexCallConverter());
    }
    // ... rest unchanged
}
```

The constructor is package-private so tests can still create fresh instances; production code only reaches the singleton via `getInstance()`, which is now self-sufficient.

### File 3 — extend `FlinkAuronConfiguration` with `FAIL_BACK_FLINK_ENGINE_ENABLED`

A boolean config option that lets the user decide whether conversion failure should silently fall back to Flink's stock Calc or fail the job. Default `true` matches the issue text ("If unsupported, continue using FlinkCalc operators") — the user sees identical behavior to a non-Auron Flink cluster when a RexNode is missing converter coverage. Advanced users who want to surface missing-converter coverage at job-submission time can set it `false`. Lives in the existing `FlinkAuronConfiguration` (created by #1854):

```java
public class FlinkAuronConfiguration extends AuronConfiguration {
    public static final String FLINK_PREFIX = "flink.";

    // Existing:
    public static final ConfigOption<Long> NATIVE_MEMORY_SIZE = ...;

    // NEW — AURON-1853:
    public static final ConfigOption<Boolean> FAIL_BACK_FLINK_ENGINE_ENABLED =
            ConfigOptions.key("auron.failback.flink.engine.enabled")
                    .description(
                            "When an Auron operator conversion fails, "
                            + "does it fall back to the Flink engine for execution?")
                    .booleanType()
                    .defaultValue(true);

    // ... rest unchanged
}
```

Key: `auron.failback.flink.engine.enabled`. When the `FlinkAuronConfiguration` is consulted via the Flink-prefix convention, the user-facing key is `flink.auron.failback.flink.engine.enabled` (`flink.` prefix added by the existing `FlinkAuronConfiguration` proxy from #1854). The shadowed `StreamExecCalc` reads this option at `translateToPlanInternal` time and obeys its semantics — see §"Failure-handling behavior" below.

---

## Failure-handling behavior

The `FAIL_BACK_FLINK_ENGINE_ENABLED` contract:

| `FAIL_BACK_FLINK_ENGINE_ENABLED` | Conversion result | Action in `translateToPlanInternal` |
|---|---|---|
| `true` (default) | success (plan built) | Construct `FlinkAuronCalcOperator`, return Auron-backed `OneInputTransformation` |
| `true` | failure (helper returned `Optional.empty()`) | Log `DEBUG`, return `super.translateToPlanInternal(planner, config)` (Flink's stock Calc) |
| `true` | thrown exception (caught inside helper's outer `Throwable` net) | Helper returns `Optional.empty()`, then same fallback as above |
| `false` | success | Construct `FlinkAuronCalcOperator`, return Auron-backed `OneInputTransformation` (unchanged) |
| `false` | failure | Throw `IllegalStateException("Auron Calc conversion failed for node N and fallback is disabled")` — fail fast |
| `false` | thrown exception (caught inside helper) | Same — helper returns `Optional.empty()` first, then throw `IllegalStateException` at the call site |

The shadowed `StreamExecCalc.translateToPlanInternal` reads the option only when conversion fails (success path doesn't pay for the lookup). Using stock `IllegalStateException` rather than a custom subclass — there's no caller that catches by type, and Flink's `translateToPlanInternal` doesn't declare any checked exception. See the code sketch in File 1 above.

**Test additions** for this behavior:
- `testFallbackEnabledTrueByDefault` (in `FlinkAuronConfigurationTest` if it already exists; else inline in `StreamExecCalcTest`): verify default is `true`.
- `testStreamExecCalcThrowsWhenFallbackDisabled` (in `StreamExecCalcTest`): set the config to `false`, give it an unsupported RexNode, assert `IllegalStateException`.
- `testStreamExecCalcFallsBackWhenFallbackEnabled`: same setup, config `true`, assert `super.translateToPlanInternal` was invoked (returned Transformation's operator is **not** `FlinkAuronCalcOperator`).

---

## Prior Art Comparison

| Aspect | AURON-1853 (this design) | Apache Gluten `gluten-flink` | Auron `spark-extension` |
|---|---|---|---|
| Substitution hook | Shadow `StreamExecCalc` (Flink package) | Same | `ColumnarRule.preColumnarTransitions` (Spark hook) |
| Plan-build timing | Eager, at `translateToPlanInternal` | Same | Lazy, at task execution (closure) |
| Plan shape | `Project[Filter?[FFIReader]]` | `Project[Filter?][Velox-source]` | `Project[Filter?][FFIReader]` |
| Fallback granularity | Per-Calc, via `super.translateToPlanInternal` | Per-Calc, via exception (no graceful super delegate) | Per-operator, via try/catch + tag |
| Operator-ID seed | `ExecNode.getId()` (int → string prefix) | atomic counter (no Flink ID link) | UUID per task |
| Factory pattern | Custom `StreamOperatorFactory` (this PR) | Pre-constructed operator in factory | None — direct `SparkPlan` subclass |
| Chaining strategy | Inherits `ALWAYS` from `TableStreamOperator` | Same | N/A (Spark) |
| Source fusion | Out of scope (#1865) | Same | Different model |
| Converter registration | Static initializer in factory | Inline in `translateToPlanInternal` | Static registry |
| Native plan serialization | proto bytes in factory | Velox-specific plan node | Closure carries lambda |

The design intentionally **diverges from Gluten on three points** (each justified): (1) graceful `super.translateToPlanInternal` fallback instead of exception propagation, (2) Flink-stable `ExecNode.getId()` for operator identity, (3) a `StreamOperatorFactory` indirection for testability.

---

## Dependencies

**No new dependencies.** Investigation confirmed all required artifacts on the existing classpath:
- `flink-table-planner_2.12` (compile) — has all Flink internals we reference.
- `flink-streaming-java` (provided) — has `StreamOperatorFactory`, `OneInputTransformation`, `StreamOperatorParameters`.
- `auron-flink-runtime` (compile, transitive `auron-core` + `proto`) — has `FlinkAuronCalcOperator`, `SchemaConverters`, `PhysicalPlanNode` builders.
- `auron-flink-planner` already depends on `auron-flink-runtime` (verified `auron-flink-planner/pom.xml:66`).

No POM edits required. No Rust changes. No proto changes (all three ExecNode messages — `ProjectionExecNode`, `FilterExecNode`, `FFIReaderExecNode` — already exist in `auron.proto`).

---

## Test Strategy

Two test classes; tiered light-to-heavy.

### Shadowed-class tests (planner-integrated)

**`StreamExecCalcTest`** (in `auron-flink-planner/src/test/java/org/apache/flink/table/planner/plan/nodes/exec/stream/` — same package as the class under test so `protected` field access works):

Plan-build paths (cover all branches of the inlined `tryBuildAuronPlan` helper):
- `testProjectAndFilterEmitsAuronOperator` — happy path, `condition != null`, arithmetic projection. Assert returned Transformation's operator is `FlinkAuronCalcOperator`.
- `testProjectOnlyEmitsAuronOperator` — happy path, `condition == null`.
- `testIdentityProjectionEmitsAuronOperator` — `RexInputRef`s only.
- `testSchemaPropagatedToProjectionExecNode` — assert the inlined plan's `ProjectionExecNode.expr_name` and `data_type` match the output RowType.

Fallback paths (default config `FAIL_BACK_FLINK_ENGINE_ENABLED=true`):
- `testFallsBackWhenUnsupportedRexNodeInCondition` — inject an unregistered RexNode subclass in the condition; assert returned Transformation's operator is `CodeGenOperator` (or whatever Flink's default produces — assert it's NOT a `FlinkAuronCalcOperator`).
- `testFallsBackWhenUnsupportedRexNodeInProjection` — same, in projection.
- `testFallsBackWhenSchemaConversionThrows` — inject a `RowType` with an unsupported logical type (e.g. RAW); assert fallback occurs.

Failure-disabled path:
- `testThrowsWhenFallbackDisabled` — set `FAIL_BACK_FLINK_ENGINE_ENABLED=false` on the `AuronAdaptor`, inject an unsupported RexNode, assert `IllegalStateException` is thrown.

Classpath verification:
- `testShadowedClassReplacesFlinkClass` — load `StreamExecCalc.class` via `Class.forName`, assert its `getProtectionDomain().getCodeSource().getLocation()` points at `auron-flink-planner` (not `flink-table-planner`). Skipped if the test runs without the auron JAR ahead of Flink's on the classpath.

### End-to-end test (full SQL job)

**`AuronCalcRewriteITCase`** — extends `AuronFlinkTableTestBase`:
- `testArithmeticProjectionEndToEnd` — `SELECT a + 1, b * 2 FROM source` over `TestValuesTableFactory` data; verify output rows match expected math. Implicitly verifies that the Auron operator actually runs (because the Flink codegen path would only succeed if the rewriter did NOT engage; we add a `setUp` step that asserts via the operator's metric that native execution counted ≥1 batch).
- `testFilterAndProjectEndToEnd` — `SELECT a * b FROM source WHERE a > 0`.
- `testFallbackOnUnsupportedExprStillExecutes` — `SELECT MY_UDF(a) FROM source` (UDF not in the Auron converter registry); job runs to completion using Flink's stock Calc.
- `testMixedCalcsInOneJob` — two Calcs in series, one supported and one unsupported; the supported one uses Auron, the unsupported one uses Flink, the job succeeds.

### What we DON'T test (and why)

| Skipped | Why |
|---|---|
| Savepoint restart preserving `auronOperatorId` | `ExecNode.getId()` stability across savepoint restarts is unverified (CR4); we document it as a known limitation rather than write a flaky test. |
| Processor ordering vs. Flink's `MultipleInputNodeCreationProcessor` (CR6) | Approach A bypasses the processor pipeline entirely — fusion processors never see our wrapped node because we are still a `StreamExecCalc`. Risk mitigated by design choice. |
| Classpath ordering at JAR-build time | `auron-flink-assembly`'s `maven-shade-plugin` already orders Auron classes ahead of Flink's — verified during investigation. A regression test would mean shading manipulation, which is out of scope. |
| Performance benchmarks | Phase 1 MVP. Benchmarks land in a follow-up after #1853 + #1860–#1864. |

---

## Out of Scope

| Item | Tracking |
|---|---|
| Logical/comparison/cast RexNode converters | #1860 / #1861 / #1864 |
| UDFs (`FlinkAuronUDFWrapperContext`) | #1862 |
| `UNIX_TIMESTAMP` and time functions | #1863 |
| Source fusion (`Calc` merged into native Kafka source) | #1865 |
| Native `MultipleInput` (multi-input fusion) | #1865 |
| Substituting `StreamExecCalcBatch` (batch planner equivalent) | Future, if batch support is requested |
| Cross-restart `auronOperatorId` stability test | CR4; documented limitation |
| Substituting non-Calc operators (Aggregate, Join) | Beyond Phase 1 |

---

## Alternatives Considered

Approach B (PlannerFactory SPI + subclassed `StreamPlanner`) and Approach C (reflection into `getExecNodeGraphProcessors`) are both detailed under §"Approach Candidates" with the reasons for rejection.

Within Approach A, Rev 1 originally proposed two helper classes (`RexProgramToPlanBuilder`, `AuronCalcConversionResult`) and a `StreamOperatorFactory` (`FlinkAuronCalcOperatorFactory`). Rev 2 removed all three:
- Construct `FlinkAuronCalcOperator` inline (Flink wraps in `SimpleOperatorFactory` automatically; matches Gluten's pattern).
- Inline plan-build inside the shadowed `StreamExecCalc.translateToPlanInternal` via a small private helper that returns `Optional<PhysicalPlanNode>`.
- No bespoke sum type — `Optional<PhysicalPlanNode>` carries the success/failure signal; converter framework already logs the underlying reason at WARN level.

---

## Relationship to Prior Phase-1 Issues

This design implements the **graph-rewriter layer** of the Flink Phase 1 stateless-operator-support track. Phase 1 sub-issues already merged provide the components this PR consumes:

| Component | Issue | Status |
|---|---|---|
| `RowData → Arrow` writer | #1850 | merged |
| `Arrow → RowData` reader | #1851 | merged |
| `FlinkNodeConverter` / `ConverterContext` framework | #1856 | merged |
| Math RexNode converters (`+ - * /`, CAST) | #1859 | merged |
| `FlinkAuronCalcOperator` (operator-layer) | #1857 | merged |
| `FlinkAuronAdaptor` / `FlinkAuronAdaptorProvider` bootstrap | #1855 | merged |
| `FlinkAuronConfiguration` proxy | #1854 | merged (extended by File 6 in this PR) |
| `SupportsAuronNative` / `FlinkAuronOperator` interfaces | #1858 | merged |

What this PR delivers, in one sentence: the substitution of `StreamExecCalc` so that those Phase 1 components actually get exercised on a real Flink SQL job.

### Deferred to later Phase 1 sub-issues

| Capability | Tracked |
|---|---|
| `unix_timestamp` and other Flink built-in functions | #1863 |
| Logical RexNode converters (`AND`, `OR`, `NOT`, comparison) | #1860 |
| User-defined Flink functions | #1862 |
| Whole-stage native operator merging (Calc fused with native source) | #1865 |

### Phase 2 / Phase 3

Out of scope: stateful operators (Agg, Join). These are separate tracks with their own design proposals.

---

## Open Items for Reviewer

Rev 2 closed items Q1 and Q2 from the prior review round. Remaining:

1. **`FAIL_BACK_FLINK_ENGINE_ENABLED` config option**: Boolean, default `true`. When `false`, conversion failure throws `IllegalStateException` at translation time. Useful for advanced users who want to surface missing converter coverage at job submission. Reviewer OK with introducing this option?
2. **`FlinkAuronCalcOperator.RESOURCE_ID_PLACEHOLDER` constant**: requires a 1-line addition to the #1857-merged operator class to expose a stable placeholder string the rewriter can pass in. Acceptable? Or hardcode the literal `"placeholder"` string at the rewriter site?
3. **`FlinkNodeConverterFactory` static initializer**: minimal change to register the three built-in converters (#1859) on class load so production callers don't need to register them manually. Alternative is to have the shadowed `StreamExecCalc.translateToPlanInternal` register on first call (more defensive but uglier). Preference?
4. **Test location for the shadowed class**: tests for `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc` must live in the same package (Java `protected` field access on `CommonExecCalc`), even though it's the Auron module's test sources. OK with this packaging?
