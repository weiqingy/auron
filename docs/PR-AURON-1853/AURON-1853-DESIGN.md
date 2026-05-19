# Design — AURON-1853: Convert Flink `StreamExecCalc` to Native Calc

**Author**: weiqingy
**Date**: 2026-05-19
**Status**: Rev 1 — pending reviewer feedback
**Issue**: https://github.com/apache/auron/issues/1853
**Depends on**: #1856 (converter framework, merged), #1859 (RexNode converters, merged), #1857 (FlinkAuronCalcOperator, PR #2263 merged 2026-05-18)
**Unblocks**: #1860, #1861, #1862, #1863, #1864, #1865

---

## Problem Statement

`FlinkAuronCalcOperator` (#1857, merged) executes a native `Project[Filter[FFIReader]]` plan but is **unreachable from real Flink SQL jobs** — the job graph today contains Flink's own `StreamExecCalc`, which builds a JVM-codegen operator via `CodeGenOperatorFactory<RowData>` instead of the Auron operator. AURON-1853 closes the loop: at job-submission time, detect `StreamExecCalc` instances whose `projection` and `condition` `RexNode`s are entirely Auron-supported, replace the JVM operator with `FlinkAuronCalcOperator`, and fall back transparently to the JVM operator whenever any RexNode is unsupported. This is the **graph-rewriter layer** of the three-layer architecture established in #1857 (plan-layer #1856/#1859 + operator-layer #1857 + **graph-rewriter** #1853).

End state after merge: a `SELECT a + b FROM t` query passes through Auron's native arithmetic instead of Flink's codegen-Calc bytecode. The first end-to-end Flink-on-Auron UT becomes possible, validating #1850/#1851/#1856/#1857/#1859 together.

---

## Approach Candidates

Three candidates were considered. The trade-off is between **invasiveness** (how deep into Flink's namespace we reach), **UX** (does the user need to set a config?), and **the issue text's literal direction** ("rewrite the Flink `StreamExecCalc` class").

### Approach A — Shadow `StreamExecCalc` in Flink's package (Gluten's pattern) — **CHOSEN**

Ship a class at FQCN `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc` inside `auron-flink-planner`. Java's classloader resolves a single class per FQCN; whichever JAR sits first on the classpath wins. With `auron-flink-planner` placed ahead of `flink-table-planner` (the normal case for Auron-enabled Flink deployments), Flink's planner constructs Auron's shadowed class whenever it builds a Calc ExecNode.

The shadowed class extends `CommonExecCalc` (Flink's parent, in `org.apache.flink.table.planner.plan.nodes.exec.common`), keeps Flink's two public constructors and `@ExecNodeMetadata(name="stream-exec-calc", version=1, minPlanVersion=v1_15, minStateVersion=v1_15)` annotation byte-for-byte, and overrides `translateToPlanInternal(PlannerBase, ExecNodeConfig)`. The override builds the Auron `PhysicalPlanNode` via #1859's converters; on success it returns a `OneInputTransformation` wired to `FlinkAuronCalcOperatorFactory`; on **any** failure (unsupported RexNode, conversion exception, schema mismatch) it delegates to `super.translateToPlanInternal(planner, config)`, which produces Flink's stock codegen operator unchanged.

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

Register an `AuronPlannerFactory` via `META-INF/services/org.apache.flink.table.factories.Factory` returning an `AuronStreamPlanner extends StreamPlanner` that overrides `getExecNodeGraphProcessors()` to return `super.getExecNodeGraphProcessors() :+ new AuronCalcRewriteProcessor()`. The processor walks the `ExecNodeGraph`, finds `StreamExecCalc` instances, and substitutes a custom ExecNode that returns a Transformation with `FlinkAuronCalcOperatorFactory`.

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
│                    FlinkAuronCalcOperatorFactory                      │
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

Five new files, all in `auron-flink-planner`:

```
auron-flink-planner/src/main/java/
└── org/apache/flink/table/planner/plan/nodes/exec/stream/
    └── StreamExecCalc.java         (shadowed — same FQCN as Flink's)

└── org/apache/auron/flink/table/planner/processor/
    ├── FlinkAuronCalcOperatorFactory.java      (StreamOperatorFactory<RowData> impl)
    ├── RexProgramToPlanBuilder.java            (pure helper: RexNodes → PhysicalPlanNode)
    └── AuronCalcConversionResult.java          (sum type: Success(plan) | Fallback(reason))
```

One file modified in `auron-flink-planner`:
```
└── org/apache/auron/flink/table/planner/converter/
    └── FlinkNodeConverterFactory.java          (add static initializer registering the 3 built-in converters)
```

Tests:
```
auron-flink-planner/src/test/java/
├── org/apache/flink/table/planner/plan/nodes/exec/stream/
│   └── StreamExecCalcTest.java                  (verifies shadowing + override behavior)
└── org/apache/auron/flink/table/planner/processor/
    ├── FlinkAuronCalcOperatorFactoryTest.java   (serialization + createStreamOperator)
    └── RexProgramToPlanBuilderTest.java         (plan-shape variants + fallback)

auron-flink-planner/src/test/java/.../runtime/
└── AuronCalcRewriteITCase.java                  (E2E SQL: TestValuesTableFactory → Calc → Sink)
```

### File 1 — Shadowed `StreamExecCalc`

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
    protected Transformation<RowData> translateToPlanInternal(
            PlannerBase planner, ExecNodeConfig config) {
        Transformation<RowData> upstream =
                (Transformation<RowData>) getInputEdges().get(0).translate(planner);
        RowType inputRowType = (RowType) getInputEdges().get(0).getOutputType();

        AuronCalcConversionResult result =
                tryConvert(projection, condition, inputRowType, (RowType) getOutputType(),
                        planner, getPersistedConfig());

        if (!result.isSuccess()) {
            LOG.debug("Falling back to Flink's CodeGen Calc for node {}: {}",
                    getId(), result.getFallbackReason());
            return super.translateToPlanInternal(planner, config);
        }

        String auronOperatorId = "FlinkAuronCalc-" + getId();
        FlinkAuronCalcOperatorFactory factory = new FlinkAuronCalcOperatorFactory(
                result.getPlan(), inputRowType, (RowType) getOutputType(), auronOperatorId);

        return ExecNodeUtil.createOneInputTransformation(
                upstream,
                createTransformationMeta(CALC_TRANSFORMATION, config),
                factory,
                InternalTypeInfo.of(getOutputType()),
                upstream.getParallelism(),
                /*memoryBytes=*/ 0);
    }

    private AuronCalcConversionResult tryConvert(
            List<RexNode> projection, @Nullable RexNode condition,
            RowType inputRowType, RowType outputRowType,
            PlannerBase planner, ReadableConfig persistedConfig) {
        try {
            ConverterContext ctx = new ConverterContext(
                    persistedConfig,
                    AuronAdaptor.getInstance().getAuronConfiguration(),
                    Thread.currentThread().getContextClassLoader(),
                    inputRowType);
            return RexProgramToPlanBuilder.build(projection, condition, inputRowType, outputRowType, ctx);
        } catch (Throwable t) {
            // Defensive: converter framework already catches per-RexNode failures, but
            // schema conversion or plan composition could still throw.
            LOG.debug("Auron Calc conversion failed for node {}", getId(), t);
            return AuronCalcConversionResult.fallback("conversion threw " + t.getClass().getSimpleName());
        }
    }
}
```

**Why `Throwable` in the outer catch**: defense-in-depth. The converter factory already catches `Exception` per #1859; this outer net handles `AssertionError` from Calcite (rare but observed) and `IllegalArgumentException` from `injectFfiReaderLeaf` (different code path, not a concern here, but cheap insurance). Fallback is the safe default for **any** failure.

### File 2 — `RexProgramToPlanBuilder`

```java
// org.apache.auron.flink.table.planner.processor.RexProgramToPlanBuilder
public final class RexProgramToPlanBuilder {
    private RexProgramToPlanBuilder() {}

    public static AuronCalcConversionResult build(
            List<RexNode> projection,
            @Nullable RexNode condition,
            RowType inputRowType,
            RowType outputRowType,
            ConverterContext ctx) {

        FlinkNodeConverterFactory factory = FlinkNodeConverterFactory.getInstance();

        // 1. Convert filter condition (if present)
        PhysicalExprNode filterExpr = null;
        if (condition != null) {
            Optional<PhysicalExprNode> converted = factory.convertRexNode(condition, ctx);
            if (!converted.isPresent()) {
                return AuronCalcConversionResult.fallback(
                        "unsupported RexNode in condition: " + condition.getClass().getSimpleName());
            }
            filterExpr = converted.get();
        }

        // 2. Convert projection expressions (always non-empty for a real Calc)
        List<PhysicalExprNode> projectExprs = new ArrayList<>(projection.size());
        for (RexNode rex : projection) {
            Optional<PhysicalExprNode> converted = factory.convertRexNode(rex, ctx);
            if (!converted.isPresent()) {
                return AuronCalcConversionResult.fallback(
                        "unsupported RexNode in projection: " + rex.getClass().getSimpleName());
            }
            projectExprs.add(converted.get());
        }

        // 3. Build FFIReader leaf (placeholder resource ID; operator rewrites at open())
        Schema inputSchema = SchemaConverters.convertToAuronSchema(inputRowType, false);
        FFIReaderExecNode ffiReader = FFIReaderExecNode.newBuilder()
                .setNumPartitions(1)               // each Flink subtask = 1 native partition (mirrors #1857)
                .setSchema(inputSchema)
                .setExportIterProviderResourceId(FlinkAuronCalcOperator.RESOURCE_ID_PLACEHOLDER)
                .build();
        PhysicalPlanNode current = PhysicalPlanNode.newBuilder()
                .setFfiReader(ffiReader)
                .build();

        // 4. Optionally wrap with Filter
        if (filterExpr != null) {
            FilterExecNode filterNode = FilterExecNode.newBuilder()
                    .setInput(current)
                    .addExpr(filterExpr)
                    .build();
            current = PhysicalPlanNode.newBuilder().setFilter(filterNode).build();
        }

        // 5. Wrap with Projection (output names + types come from outputRowType)
        ProjectionExecNode.Builder projBuilder = ProjectionExecNode.newBuilder().setInput(current);
        for (int i = 0; i < projectExprs.size(); i++) {
            projBuilder.addExpr(projectExprs.get(i));
            projBuilder.addExprName(outputRowType.getFieldNames().get(i));
            projBuilder.addDataType(SchemaConverters.convertToAuronArrowType(
                    outputRowType.getTypeAt(i)));
        }
        PhysicalPlanNode finalPlan = PhysicalPlanNode.newBuilder()
                .setProjection(projBuilder.build())
                .build();

        return AuronCalcConversionResult.success(finalPlan);
    }
}
```

Plan-shape rule: **always emits `Project[Filter?[FFIReader]]`**. The `Project` is unconditional even when the projection is identity, because `FlinkAuronCalcOperator.injectFfiReaderLeaf` accepts `Project[FFIReader]` and `Project[Filter[FFIReader]]`; the bare-`FFIReader` and `Filter[FFIReader]` shapes from #1857's contract are not produced by a normal Flink Calc (Calc always has a projection). This keeps the rewriter's output deterministic.

**Identity projection** — Flink may produce a Calc that re-emits all input columns unchanged. The Projection's expressions are `RexInputRef`s with the same indices, which #1859's `RexInputRefConverter` handles fine. No special case needed.

### File 3 — `FlinkAuronCalcOperatorFactory`

```java
// org.apache.auron.flink.table.planner.processor.FlinkAuronCalcOperatorFactory
public class FlinkAuronCalcOperatorFactory
        implements StreamOperatorFactory<RowData> {

    private static final long serialVersionUID = 1L;

    private final byte[] planBytes;                  // proto-serialized PhysicalPlanNode
    private final byte[] inputRowTypeBytes;          // Flink-serialized RowType (via DataTypes registry)
    private final byte[] outputRowTypeBytes;
    private final String auronOperatorId;
    private ChainingStrategy chainingStrategy = ChainingStrategy.ALWAYS;

    public FlinkAuronCalcOperatorFactory(
            PhysicalPlanNode plan, RowType inputRowType, RowType outputRowType, String auronOperatorId) {
        this.planBytes = plan.toByteArray();
        this.inputRowTypeBytes = SerializerUtils.serialize(inputRowType);
        this.outputRowTypeBytes = SerializerUtils.serialize(outputRowType);
        this.auronOperatorId = auronOperatorId;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends StreamOperator<RowData>> T createStreamOperator(
            StreamOperatorParameters<RowData> parameters) {
        PhysicalPlanNode plan;
        try {
            plan = PhysicalPlanNode.parseFrom(planBytes);
        } catch (InvalidProtocolBufferException e) {
            throw new IllegalStateException("Failed to deserialize PhysicalPlanNode", e);
        }
        RowType inputRowType = SerializerUtils.deserialize(inputRowTypeBytes, RowType.class);
        RowType outputRowType = SerializerUtils.deserialize(outputRowTypeBytes, RowType.class);

        FlinkAuronCalcOperator op = new FlinkAuronCalcOperator(
                plan, inputRowType, outputRowType, auronOperatorId);
        op.setup(parameters.getContainingTask(), parameters.getStreamConfig(), parameters.getOutput());
        return (T) op;
    }

    @Override
    public void setChainingStrategy(ChainingStrategy strategy) { this.chainingStrategy = strategy; }

    @Override
    public ChainingStrategy getChainingStrategy() { return chainingStrategy; }

    @Override
    public Class<? extends StreamOperator> getStreamOperatorClass(ClassLoader classLoader) {
        return FlinkAuronCalcOperator.class;
    }
}
```

**Why `byte[]` for RowType too**: `RowType` is `Serializable`, but Flink also provides a stable LogicalType-DataType conversion. We use the standard utility in `org.apache.flink.table.runtime.typeutils.InternalTypeInfo`. **Implementation phase**: verify whether `RowType` Java serialization is safe across cluster nodes; if not, fall back to `LogicalTypeSerializer.toString` + `parse`.

**Note**: `FlinkAuronCalcOperator.RESOURCE_ID_PLACEHOLDER` does not exist yet in #1857. Either we add a public constant on the operator class (small one-line addition; needs reviewer OK) or we use the magic string `"placeholder"` (which `injectFfiReaderLeaf` already accepts since it doesn't validate the placeholder value — verify by re-reading #1857 source). **Design phase action**: confirm what placeholder string `FlinkAuronCalcOperator.open()`-side leaf rewrite expects.

### File 4 — `AuronCalcConversionResult`

```java
// org.apache.auron.flink.table.planner.processor.AuronCalcConversionResult
public final class AuronCalcConversionResult {
    private final PhysicalPlanNode plan;     // null on fallback
    private final String fallbackReason;     // null on success

    private AuronCalcConversionResult(PhysicalPlanNode plan, String reason) {
        this.plan = plan;
        this.fallbackReason = reason;
    }

    public static AuronCalcConversionResult success(PhysicalPlanNode plan) {
        return new AuronCalcConversionResult(plan, null);
    }

    public static AuronCalcConversionResult fallback(String reason) {
        return new AuronCalcConversionResult(null, reason);
    }

    public boolean isSuccess() { return plan != null; }
    public PhysicalPlanNode getPlan() { return Objects.requireNonNull(plan); }
    public String getFallbackReason() { return Objects.requireNonNull(fallbackReason); }
}
```

A 20-line value class. Could be replaced with `Optional<PhysicalPlanNode>` + logging at the call site, but the named "fallback reason" string is useful for log forensics (the user can grep `WARN` lines and learn exactly which RexNode killed the conversion).

### File 5 — modify `FlinkNodeConverterFactory`

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

### File 6 — extend `FlinkAuronConfiguration` with `FAIL_BACK_FLINK_ENGINE_ENABLED`

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
| `true` (default) | success | Return Auron-backed `OneInputTransformation` |
| `true` | failure (`AuronCalcConversionResult.fallback`) | Log `DEBUG`, return `super.translateToPlanInternal(planner, config)` (Flink's stock Calc) |
| `true` | thrown exception (defensive `Throwable` catch) | Log `WARN` with the exception, return `super.translateToPlanInternal(...)` |
| `false` | success | Return Auron-backed `OneInputTransformation` (unchanged) |
| `false` | failure (`AuronCalcConversionResult.fallback`) | Throw `AuronCalcConversionException` carrying the fallback reason — fail fast, no silent silently-falling-back-to-Flink |
| `false` | thrown exception | Re-throw as `AuronCalcConversionException` wrapping the cause |

The shadowed `StreamExecCalc.translateToPlanInternal` reads the option once at translation time:

```java
@Override
protected Transformation<RowData> translateToPlanInternal(
        PlannerBase planner, ExecNodeConfig config) {
    Transformation<RowData> upstream = ...;
    RowType inputRowType = ...;

    boolean fallbackEnabled = AuronAdaptor.getInstance()
            .getAuronConfiguration()
            .get(FlinkAuronConfiguration.FAIL_BACK_FLINK_ENGINE_ENABLED);

    AuronCalcConversionResult result = tryConvert(...);

    if (!result.isSuccess()) {
        if (fallbackEnabled) {
            LOG.debug("Falling back to Flink's CodeGen Calc for node {}: {}",
                    getId(), result.getFallbackReason());
            return super.translateToPlanInternal(planner, config);
        } else {
            throw new AuronCalcConversionException(
                    "Auron Calc conversion failed and fallback is disabled (node "
                    + getId() + "): " + result.getFallbackReason());
        }
    }

    // ... build Auron Transformation as before
}
```

`AuronCalcConversionException extends RuntimeException` — new class in the processor package, single field (cause + message). Unchecked because Flink's `translateToPlanInternal` is not declared to throw checked exceptions.

**Test additions** for this behavior:
- `testFallbackEnabledTrueByDefault` (config-test, in `FlinkAuronConfigurationTest` if it exists, else new): verify default is `true`.
- `testStreamExecCalcThrowsWhenFallbackDisabled` (in `StreamExecCalcTest`): set the config to `false`, give it an unsupported RexNode, assert the exception is thrown.
- `testStreamExecCalcFallsBackWhenFallbackEnabled`: same setup, config `true`, assert `super.translateToPlanInternal` was invoked and a `CodeGenOperatorFactory` is the returned Transformation's factory.

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

Five test classes; tiered light-to-heavy.

### Unit tests (mock-heavy, fast)

**`RexProgramToPlanBuilderTest`** — directly exercise the plan builder with hand-built `RexNode` trees.
- `testProjectAndFilterEmitsProjectFilterFfiReader` — happy path, `condition != null`, simple projection.
- `testProjectOnlyEmitsProjectFfiReader` — happy path, `condition == null`.
- `testUnsupportedRexNodeInConditionReturnsFallback` — uses an unregistered RexNode subclass; asserts `isSuccess() == false`.
- `testUnsupportedRexNodeInProjectionReturnsFallback` — same, in projection.
- `testIdentityProjectionEmitsProject` — `RexInputRef`s only; asserts plan is still valid.
- `testSchemaPropagatedFromOutputRowType` — verifies ProjectionExecNode's `expr_name` and `data_type` match the output RowType.

**`FlinkAuronCalcOperatorFactoryTest`**:
- `testSerializableRoundTrip` — serialize the factory, deserialize, compare plan bytes.
- `testCreateStreamOperatorReturnsFlinkAuronCalcOperator` — mock `StreamOperatorParameters`, assert returned operator is non-null and the right type.
- `testChainingStrategyDefaultsToAlways` — assert `getChainingStrategy() == ChainingStrategy.ALWAYS`.

### Shadowed-class tests (planner-integrated, medium)

**`StreamExecCalcTest`** (in `auron-flink-planner/src/test/java/org/apache/flink/table/planner/plan/nodes/exec/stream/`):
- `testTranslateToPlanInternalUsesAuronFactoryWhenConvertible` — construct a `StreamExecCalc` with an arithmetic projection, mock the planner, assert the returned Transformation's factory is a `FlinkAuronCalcOperatorFactory`.
- `testTranslateToPlanInternalFallsBackWhenUnsupportedRexNode` — construct with a `RexFieldAccess` (unregistered today), assert the returned Transformation's factory is `CodeGenOperatorFactory` (Flink's default).
- `testTranslateToPlanInternalFallsBackWhenSchemaConversionThrows` — inject a `RowType` with an unsupported logical type (e.g. RAW), assert fallback.
- `testShadowedClassReplacesFlinkClass` — load `StreamExecCalc.class` via `Class.forName`, assert its `getProtectionDomain().getCodeSource().getLocation()` points at `auron-flink-planner` (not `flink-table-planner`). Skipped if the test runs without the auron JAR on the classpath ahead of Flink's.

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

Two minor alternatives within the chosen approach:
- **Option α.1**: extract `RexProgramToPlanBuilder.build` into the shadowed `StreamExecCalc` directly (no helper class). Rejected — separating it gives unit tests a clean entry point without instantiating a planner.
- **Option α.2**: return `Optional<PhysicalPlanNode>` from the builder and pass a fallback reason via a separate channel. Rejected — `AuronCalcConversionResult` keeps reason + result paired in one place, simpler for log forensics.

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

1. **Approach A vs. B**: This design picks A (shadow `StreamExecCalc` in Flink's package). Approach B (custom `PlannerFactory` SPI + subclassed `StreamPlanner`) is rejected because Flink 1.18 has no SPI registration for `ExecNodeGraphProcessor` (verified via `javap` against `flink-table-planner_2.12-1.18.1.jar`) and replacing the planner factory creates a `factoryIdentifier()` collision or forces user config. Reviewer concur?
2. **`FAIL_BACK_FLINK_ENGINE_ENABLED` config option**: Boolean, default `true`. When `false`, conversion failure throws `AuronCalcConversionException`. Useful for advanced users who want to surface missing converter coverage at job submission. Reviewer OK with introducing this option?
3. **`FlinkAuronCalcOperator.RESOURCE_ID_PLACEHOLDER` constant**: requires a 1-line addition to the #1857-merged operator class to expose a stable placeholder string the rewriter can pass in. Acceptable? Or hardcode the literal `"placeholder"` string at the rewriter site?
4. **`FlinkNodeConverterFactory` static initializer**: minimal change to register the three built-in converters (#1859) on class load so production callers don't need to register them manually. Alternative is to have the shadowed `StreamExecCalc.translateToPlanInternal` register on first call (more defensive but uglier). Preference?
5. **`AuronCalcConversionResult` sum type**: a 20-line value class with `success(plan)` and `fallback(reason)` constructors. Alternative is `Optional<PhysicalPlanNode>` + a logged WARN at the call site. The sum type keeps the fallback reason paired with the result for log forensics. Preference?
6. **Test location for the shadowed class**: tests for `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc` must live in the same package (Java protected-access rules), even though it's the Auron module's test sources. OK with this packaging?
