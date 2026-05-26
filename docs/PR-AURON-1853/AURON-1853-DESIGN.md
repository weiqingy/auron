# Design — AURON-1853: Convert Flink `StreamExecCalc` to Native Calc

**Author**: weiqingy
**Date**: 2026-05-19 (initial) — **Rev 2**: 2026-05-19 (simplified: drop factory + helper classes) — **Rev 3**: 2026-05-21 (log-based observability for fallback; global kill switch deferred) — **Rev 4**: 2026-05-26 (deployment-model discussion post-PR submission)
**Status**: Rev 3 architecture **Approved 2026-05-21** by @Tartarus0zm; Rev 4 deployment guidance awaiting reviewer alignment on PR #2283
**Issue**: https://github.com/apache/auron/issues/1853
**Depends on**: #1856 (converter framework, merged), #1859 (RexNode converters, merged), #1857 (FlinkAuronCalcOperator, PR #2263 merged 2026-05-18)
**Unblocks**: #1860, #1861, #1862, #1863, #1864, #1865

---

## Rev 4 — Deployment-Model Discussion (2026-05-26, Post-PR Submission)

After PR #2283 was submitted, Round 2 review surfaced a question that earlier design rounds had left implicit: **how is Auron deployed into a Flink installation, and what guarantees the JAR-shadowing mechanism actually activates?**

For context: the chosen architecture overrides Flink's built-in `StreamExecCalc` by shipping an Auron class with the same fully-qualified name (`org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc`). The JVM's classloader caches one class per FQCN, so whichever class loads first wins for the lifetime of the JVM. Earlier rounds specified the *mechanism* (the shadow class) but did not specify the *deployment procedure* that ensures Auron's class is the one loaded. This section addresses that gap.

### The Two Options

**A1 — current implementation (JAR overlay).** The user drops `auron-flink-planner.jar` into `$FLINK_HOME/lib/` alongside Flink's existing `flink-table-planner.jar`. Both jars are present at runtime. The JVM classloader resolves `StreamExecCalc` to whichever jar appears first in its directory traversal of `lib/` — typically alphabetical (so `auron-flink-planner.jar` < `flink-table-planner.jar`), but not spec-guaranteed across JVM vendors.

**A2 — shaded uber-jar replacement.** Auron ships `auron-flink-planner-shaded.jar`, a fat jar that contains all of `flink-table-planner`'s content with Auron's `StreamExecCalc` (and any other Auron overrides) substituted in. The user replaces `$FLINK_HOME/lib/flink-table-planner.jar` with this shaded jar. Only one `StreamExecCalc` class exists on the classpath — Auron's — so activation is structural rather than ordering-dependent. Cost: `maven-shade-plugin` configuration (~50 lines `pom.xml`), ~30-40MB artifact (the size of `flink-table-planner`), one-time ASF NOTICE update for embedded Flink content, and per-Flink-version re-shading that slots into the existing build matrix.

A natural-seeming third option — "tell users to remove `flink-table-planner.jar` from `lib/` and keep only Auron's overlay jar" — does not work without shading. Auron's `StreamExecCalc` extends `CommonExecCalc` and transitively depends on other classes that live in `flink-table-planner.jar`; removing that jar `NoClassDefFoundError`s at class-load. To make the user-visible outcome "only Auron's jar in `lib/`" actually function, Auron's jar must contain the planner content itself — which is A2.

### What Triggered This Discussion

Two angles came up in Round 2 review:

- **Robustness angle** — the classpath-ordering assumption in A1 is not enforced; if JVM filesystem ordering changes (across JVM vendors, container image rebuilds, classpath manipulation in launch scripts), Auron's class is silently no longer the resolved one, with no error or warning.
- **Simplification angle** — the deployment story should be a single deterministic jar swap rather than overlay-plus-ordering.

Both angles point to A2.

### Framework-Capability Contrast (Spark vs Flink)

Auron's Spark side does not face this problem because Spark exposes a supported plan-rewrite SPI; Flink 1.18 does not. The asymmetry shapes what is and isn't feasible on the Flink side.

| Dimension | Auron-Spark | Auron-Flink |
|---|---|---|
| Plan-rewrite hook | `SparkSessionExtensions.injectColumnar` — officially supported SPI | None for `ExecNode` substitution on Flink 1.18 |
| Activation mechanism | `spark.sql.extensions=AuronSparkSessionExtension` in `spark-defaults.conf` | Classpath-ordered FQCN shadow of `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc` |
| Classpath-order sensitivity | None — Spark instantiates the named class by reflection | Yes — first `StreamExecCalc` to load wins |
| Activation visibility | Spark prints "extension enabled" in driver log unambiguously | One-shot INFO activation log; `FAIL_BACK_FLINK_ENGINE_ENABLED=false` as strict-mode signal |
| Prior art for the Flink situation | n/a | Apache Gluten's `gluten-flink/` ships 13 shadow `StreamExec*` classes in the same FQCN package — same JAR-shadowing mechanism as A1 |

A true Spark-grade UX on Flink would require either upstreaming a Flink `ExecNode`-replacement SPI (verified absent across 1.18, 1.20, 2.0, and current master — no proposed FLIP in flight) or A2.

### Why Not Use the Exact Same Procedure as gluten-flink?

Fair question — gluten-flink's documented procedure ([`gluten-flink/docs/Flink.md`](https://github.com/apache/incubator-gluten/blob/main/gluten-flink/docs/Flink.md)) does *not* drop their jars directly in `$FLINK_HOME/lib/`. Instead:

1. Place Gluten jars in a dedicated directory `$FLINK_HOME/gluten_lib/`.
2. Edit `$FLINK_HOME/bin/config.sh` to **prepend** `gluten_lib/*` to `FLINK_CLASSPATH`.

This is more deterministic than A1 because the shell-script edit makes the classpath ordering an explicit operator action rather than relying on JVM directory traversal. Call this variant **A1-explicit**:

| Approach | User steps | Activation determinism |
|---|---|---|
| **A1** (current) | 1 (drop jar in `lib/`) | Implicit — JVM directory traversal order |
| **A1-explicit** (gluten-flink's) | 2 (place jars in `auron_lib/`; edit `config.sh`) | Explicit — shell-script `FLINK_CLASSPATH` prepend |
| **A2** | 1 (swap one jar in `lib/`) | Structural — only one `StreamExecCalc` class exists |

A1-explicit is genuinely better than A1 on determinism. The question is whether to stop at A1-explicit or push through to A2:

- **A2 dominates A1-explicit on user steps**: 1 vs 2. A1-explicit requires editing a shell script, which is fragile in containerized/managed Flink deployments where `config.sh` may be regenerated, version-locked, or shared across teams.
- **A2 dominates A1-explicit on determinism**: structural beats explicit-ordering. No later operator action (manual `config.sh` edit, classpath manipulation in launch scripts, base image rebuild) can re-misorder the classpath under A2, because there is no other jar to misorder against.
- **A1-explicit retains a place as a fallback** if A2's scope expansion (shading, artifact size, NOTICE handling) is not acceptable for this PR. It delivers improved determinism with no code or build changes — purely a documentation change. Reviewers may surface this as a third option if A2's cost/benefit is not acceptable.

### UX Comparison Across Systems

| | User steps to enable | Activation determinism |
|---|---|---|
| Auron-Spark | (i) drop jar in `$SPARK_HOME/jars/` (ii) add `spark.sql.extensions=AuronSparkSessionExtension` to `spark-defaults.conf` | Standard Spark SPI — Spark loads the named class explicitly |
| Gluten-Spark | (i) drop jar (ii) add `spark.plugins=org.apache.gluten.GlutenPlugin` | Same Spark SPI |
| Gluten-Flink (= A1-explicit) | (i) place jars in `$FLINK_HOME/gluten_lib/` (ii) edit `bin/config.sh` to prepend `gluten_lib/*` to `FLINK_CLASSPATH` | Classpath ordering — explicit operator action |
| **Auron-Flink A1** | (i) drop jar in `$FLINK_HOME/lib/` | Classpath ordering — implicit (JVM filesystem traversal order) |
| **Auron-Flink A2** | (i) replace `$FLINK_HOME/lib/flink-table-planner.jar` with `auron-flink-planner-shaded.jar` | Structural — exactly one class on the classpath, period |

A2 is the only Flink-side approach that matches Spark's UX clarity — one user action, deterministic outcome. Gluten-Flink's published procedure makes classpath ordering explicit but still asks the user to think about classpath ordering at all, which Spark does not.

### Preferred Direction (Open to Discussion): A2

A2 is the leaning preference. Reasoning:

- Activation is structural, not procedural — users can't accidentally fail to activate Auron; no need to verify via the INFO log; no possibility of silent degradation.
- Auron's Flink support is pre-GA — there is no installed user base today — so the A1-now-A2-later path is purely internal work, no user migration cost. That said, future ExecNode shadows (#1860-#1865) ride on the same deployment model; doing A2 upfront means each subsequent shadow lands against the clean model, while doing A2 later means each interim shadow lands against A1 and gets re-validated on the A2 cutover.
- Cost arguments are bounded: shade configuration is ~50 lines of `pom.xml`; ~30-40MB artifact is acceptable in modern JVM ecosystems (`flink-dist*.jar` is already ~120MB); per-Flink-version re-shading slots into the existing build matrix; license handling is one-time work that Apache projects do regularly (cf. `flink-shaded`, `spark-hadoop-cloud`).
- "Gluten doesn't do it" is real but not decisive — gluten-flink is self-described as experimental, and we don't have to inherit their incomplete state.
- Delivers the simplification angle directly; structurally eliminates the robustness angle as a side effect.

This preference is held lightly.

### Open Question for Reviewers

Is **A2** acceptable as the deployment model, and is it acceptable to land it **in this PR** (scope expansion of ~2-4 days for shade config + NOTICE + smoke test) — or should it be an immediate follow-up after this PR lands A1?

### Items Suggested for Deferral

- **The deployment guide itself** (concrete user-docs PR with `lib/` swap instruction, verification step, troubleshooting) — file as a separate documentation issue under #1264 once the A1-vs-A2 choice is settled.
- **Upstreaming a Flink `ExecNode`-extension SPI** — multi-release Flink-community contribution; out of Auron Phase 1 scope regardless of which option is chosen here.

### Net Impact

The JAR-shadowing architecture (Auron's class in Flink's package) is unchanged in either option — the choice is purely about how the shadow is deployed. Under A2, this PR grows by ~50 lines of `pom.xml` plus a one-time NOTICE update; the observability machinery (one-shot INFO activation log; `FAIL_BACK_FLINK_ENGINE_ENABLED=false` strict mode) becomes less load-bearing because activation is structurally guaranteed (it can be kept or simplified). Under A1, no code or build changes are triggered; the observability machinery is retained as the runtime/strict-mode verification path.

---

## Rev 3 Changes (2026-05-21)

Two adjustments after a second review pass focused on user-visibility:

**1. Log-based observability for fallback events**. Without notification, a user whose job hits a missing-converter case sees only "slow" execution — they can't tell Auron silently fell back. Flink `MetricGroup` counters would be ideal but conversion happens at JobManager planning time before any Auron operator (and its `MetricGroup`) exists, so logging is the natural channel. Adding two log behaviors, both always-on, both at planning time:

- **WARN per fallback**: one line per unique unsupported `RexNode` class per job submission (deduplicated so a job with 50 identical Calcs doesn't spam). Includes the failing Calc's `ExecNode.getId()` and the `RexNode` class name. Lets a user grep for missing-converter coverage and file feature requests.
- **INFO submission summary**: one line per job submission summarizing native-acceleration ratio: `Auron: 3 of 5 Calc operators native-accelerated; 2 fell back (unsupported: RexFieldAccess, RexOver)`. Tells the user at a glance how much of their job got accelerated.

Both behaviors are independent of `FAIL_BACK_FLINK_ENGINE_ENABLED` (which controls hard-fail vs. fallback). Logs fire on every fallback in default-mode jobs. See updated §"Failure-handling behavior".

**2. Global kill switch deferred to future work** (out of scope for #1853). Rationale: if a user doesn't have Auron in their deployment, they wouldn't place `auron-flink-planner` ahead of `flink-table-planner` on the classpath — the shadowing simply wouldn't activate. The "I have Auron deployed but want to disable it" use case is not a Calc-specific concern and can land in a separate PR if and when demand emerges.

**3. Round-2 follow-ups confirmed** (no design change): `auron.failback.flink.engine.enabled` config option retained as the opt-in CI/dev strict-mode flag (Rev 2 design unchanged), placeholder resource ID hardcoded at the rewriter site, `FlinkNodeConverterFactory` static initializer for built-in converters, `StreamExecCalcTest` packaged at `org.apache.flink.table.planner.plan.nodes.exec.stream` in our test sources.

**Net delta from Rev 2**: same file count and structure; ~10 lines of additional logging inside the shadowed `StreamExecCalc` and a small helper for the per-submission summary. No structural changes; no new dependencies.

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
| `true` | failure (helper returned `Optional.empty()`) | **WARN log** (deduplicated per submission), return `super.translateToPlanInternal(planner, config)` (Flink's stock Calc) |
| `true` | thrown exception (caught inside helper's outer `Throwable` net) | Helper returns `Optional.empty()`, then same fallback as above |
| `false` | success | Construct `FlinkAuronCalcOperator`, return Auron-backed `OneInputTransformation` (unchanged) |
| `false` | failure | Throw `IllegalStateException("Auron Calc conversion failed for node N and fallback is disabled")` — fail fast |
| `false` | thrown exception (caught inside helper) | Same — helper returns `Optional.empty()` first, then throw `IllegalStateException` at the call site |

The shadowed `StreamExecCalc.translateToPlanInternal` reads the option only when conversion fails (success path doesn't pay for the lookup). Using stock `IllegalStateException` rather than a custom subclass — there's no caller that catches by type, and Flink's `translateToPlanInternal` doesn't declare any checked exception. See the code sketch in File 1 above.

### Observability (Rev 3)

Without notification, a user whose job hits a missing-converter case sees only "slow" execution. Flink `MetricGroup` counters would be ideal but conversion happens at JobManager planning time before any Auron operator (and its `MetricGroup`) exists; logging is the natural channel.

**Per-fallback WARN log** — emitted inside the shadowed `StreamExecCalc.translateToPlanInternal` when the helper returns `Optional.empty()` and `FAIL_BACK_FLINK_ENGINE_ENABLED=true`:

```
WARN Auron StreamExecCalc fallback (node 17): unsupported RexNode org.apache.calcite.rex.RexFieldAccess; using Flink CodeGen Calc.
```

**Deduplication**: only log the first occurrence of each unique `(unsupportedRexNodeClass)` tuple within a single job submission. Implementation: a planner-scoped `Set<Class<? extends RexNode>>` shared via a static `ThreadLocal` cleared at submission start; on each WARN, check-and-add. A job with 50 identical Calcs producing the same `RexFieldAccess` error logs only once.

**Per-submission summary** — emitted once at the end of submission by the *first* Auron `StreamExecCalc` whose `translateToPlanInternal` runs, OR by a shutdown hook on the planner thread (TBD in implementation):

```
INFO Auron Flink Phase 1: 3 of 5 Calc operators native-accelerated; 2 fell back (unsupported RexNode classes: RexFieldAccess, RexOver).
```

The implementation will pick whichever hook is cleanest; the contract is that **at most one summary line per submission** appears, and only when at least one `StreamExecCalc` was processed.

**Test additions** for the Rev 3 observability:
- `testFallbackEmitsWarnLogOnce` (in `StreamExecCalcTest`): submit two `StreamExecCalc` instances with the same unsupported RexNode class, assert only one WARN line emitted (use a captured log appender).
- `testFallbackEmitsDistinctWarnLogsForDistinctRexClasses` (in `StreamExecCalcTest`): submit two with different unsupported RexNode classes, assert two WARN lines emitted.
- `testSubmissionSummaryLog` (E2E in `AuronCalcRewriteITCase`): submit a job mixing supported + unsupported Calcs, assert exactly one INFO summary line with the expected counts.

**Test additions** for `FAIL_BACK_FLINK_ENGINE_ENABLED`:
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
| Global kill switch (`auron.flink.enabled` toggle to disable Auron without rebuilding) | Future PR. Rationale: a deployment without Auron simply wouldn't place `auron-flink-planner` ahead of `flink-table-planner` on the classpath; the shadow is inert. The "Auron deployed but disable temporarily" use case has no demonstrated demand yet and is not Calc-specific. |
| Flink `MetricGroup` counters for native-acceleration ratio | Future PR. Requires a runtime hook (currently we only have planning-time visibility); could surface in Grafana/Prometheus once we have it. |

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

## Resolved Items (review history)

All review items from the prior rounds are now settled:

**Round 1** (Rev 2):
- ✅ **Q1** — Dropped `FlinkAuronCalcOperatorFactory`. Operator constructed inline; Flink wraps in `SimpleOperatorFactory` automatically.
- ✅ **Q2** — Dropped `RexProgramToPlanBuilder` and `AuronCalcConversionResult`. Plan-build inlined into shadowed `StreamExecCalc`; fallback signal is `Optional<PhysicalPlanNode>`.

**Round 2** (Rev 3):
- ✅ **Notification** — log-based observability added: per-fallback WARN (deduplicated) + per-submission INFO summary. Always-on, no user opt-in.
- ✅ **Granularity** — confirmed per-Calc fallback is the right level. Per-operator-type config deferred to whichever sub-issue needs it.
- ✅ **Global kill switch** — deferred to a future PR. A deployment without Auron simply wouldn't place `auron-flink-planner` ahead of `flink-table-planner` on the classpath, so the shadow is inert by default. Not a Calc-specific concern.

**Round-2 follow-ups confirmed in Rev 3** (no design change from Rev 2):
- ✅ `FAIL_BACK_FLINK_ENGINE_ENABLED` config option retained as the opt-in CI/dev strict-mode flag.
- ✅ Placeholder resource ID hardcoded at the rewriter site (no `RESOURCE_ID_PLACEHOLDER` constant added to #1857).
- ✅ Static initializer in `FlinkNodeConverterFactory` for built-in converters.
- ✅ `StreamExecCalcTest` packaged at `org.apache.flink.table.planner.plan.nodes.exec.stream` in our test sources.

**Status**: design approved 2026-05-21 by @Tartarus0zm. Proceeding to SPEC + PLAN, then implementation.
