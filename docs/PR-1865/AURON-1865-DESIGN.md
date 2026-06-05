# Design — AURON #1865: Support merge native operators (Flink source + Calc)

## Problem Statement

In the current Flink integration, a `SELECT … FROM kafka WHERE …` plan runs as two independent
native operators: the native Kafka source (`KafkaScan`) converts its columnar Arrow output to
`RowData` and emits it, then the shadowed `StreamExecCalc` operator converts that `RowData` back to
Arrow (FFI export), runs `Project[Filter]` natively, and converts back to `RowData`. The chain pays
**three** row↔column conversions where one would suffice. #1865 fuses the source and the Calc into a
single native plan tree (`Project[Filter?[KafkaScan]]`) so the data stays columnar end-to-end and is
converted to `RowData` exactly once, at the chain tail.

Issue: https://github.com/apache/auron/issues/1865. Today every native Calc is wrapped with
row→columnar (R2C) and columnar→row (C2R) adapters independently, so a `source → calc` chain pays a
conversion at every operator boundary. The goal is whole-stage merging: fuse a contiguous run of native
operators rooted at the native source into a single native plan, keeping conversions only at the
boundary between Auron-native and Flink-native operators — i.e. convert downward from the source head,
leaving one column→row conversion at the chain tail. This mirrors the plan-merge already implemented on
Auron's Spark side. It is more involved than single-operator conversion and must handle several cases
(watermarks, partial fusion, fallback). OSS Auron has none of this on the Flink side yet.

## Approach Candidates

### Approach A — Plan-time, source-rooted whole-stage merge via `SupportsAuronNative` (CHOSEN)
Build on the existing native-operator abstraction `SupportsAuronNative` (`auron-flink-runtime/.../
operator/SupportsAuronNative.java`): both `AuronKafkaSourceFunction` (`implements FlinkAuronFunction`)
and `FlinkAuronCalcOperator` (`implements FlinkAuronOperator`) already expose `getPhysicalPlanNodes()` /
`getOutputType()` / `getAuronOperatorId()`. In `StreamExecCalc.translateToPlanInternal`, translate the
upstream, detect that its operator/function is `SupportsAuronNative` (i.e. its input is already a native
op), pull its `getPhysicalPlanNodes()` (the source's `KafkaScan`), and use it as the leaf of the Calc's
`Project[Filter]` sub-plan in place of the `FFIReader` placeholder. The fused plan runs in a
**source-type native operator** (`FlinkAuronStreamSource`, see Detailed Design), and the Calc returns
that transformation directly so **no Calc operator is created**. A Calc that cannot fuse falls back to the
stock Flink Calc (codegen) — **not** a standalone FFIReader native island (see Decision: strict gating).
- **Pros**: reuses the existing `SupportsAuronNative` abstraction (no bespoke accessor); mirrors the
  Spark side's proven `isNative(child)` + `.setInput(childPlan)` model; source-rooted (from the source
  head); native engine needs zero change; the non-fusible fallback (stock Flink Calc) is trivial.
- **Cons**: needs a source operator that can run a fused plan (`FlinkAuronStreamSource` +
  `LegacySourceTransformationTranslator` rewrite), not yet in code; watermark case must be gated.
- **Prior art**: `spark-extension/.../AuronConvertStrategy.scala` (bottom-up mark,
  convert-if-child-native); `NativeFilterBase/NativeProjectBase` (`.setInput(childPlan)`); Flink
  `PushWatermarkIntoTableSourceScanAcrossCalcRule` (keeps filter above watermark).

### Approach B — Runtime merge handshake
Keep per-operator planning; at runtime the source registers its native plan in `JniBridge` and the
Calc operator splices it in place of its `FFIReader` leaf.
- **Pros**: less planner surgery.
- **Cons**: a one-input operator cannot host a self-driving `KafkaScan` leaf (no upstream to pull from),
  so the scan must live in a source operator anyway; watermark/checkpoint re-homing is harder; farther
  from the Spark precedent. **Rejected.**

### Approach C — Calc keeps its operator, swaps FFIReader→KafkaScan leaf
- **Cons**: a `FlinkAuronCalcOperator` (one-input) whose native leaf is `KafkaScan` would have the
  operator's FFI-export input AND a self-reading Kafka leaf in the same process — contradictory data
  sources. **Rejected** (investigation confirmed incompatible leaves).

## Decision

**Approach A**, with **strict gating**: a Calc converts to native **only when it can fuse into a native
upstream chain**. The **initial scope of this PR** fuses `Project[Filter]` into the source only when the
source has no event-time watermark; event-time-watermark cases are tracked as a follow-up issue (#2315,
see Out of Scope).

A Calc is converted to native (fused into its source) **iff all hold**:
1. The upstream operator/function is `SupportsAuronNative` (today: the Auron native Kafka source,
   `AuronKafkaSourceFunction implements FlinkAuronFunction`).
2. The Calc fully converts to native (every Rex supported).
3. **The source has no event-time watermark strategy** (`watermarkStrategy == null`: processing-time
   or no-watermark jobs). When an event-time watermark IS present, the filter must stay above the
   watermark generator (canonical Flink `…AcrossCalc` behavior), so this PR cannot fuse. Full event-time
   fusion (native watermark relocation, generated from the pre-filter scan timestamp) requires
   native-engine watermark support and is tracked separately as #2315.

**In all other cases the Calc stays a stock Flink Calc (codegen).** The standalone FFIReader native-Calc
path — which #1853 currently uses whenever every Rex is supported, regardless of input — is **dropped**.
The reasoning: a Calc that cannot fuse runs as a lone native island that pays a row→columnar conversion in
and a columnar→row conversion out for no net win, and can even be slower than Flink's codegen Calc. Leaving
that island in place risks new users measuring Flink-on-Auron on a non-fusible Calc, seeing disappointing
performance, and writing off the integration. So conversion is reserved for the fusible case, where the
chain stays columnar end-to-end and pays a single columnar→row conversion at the tail. This changes #1853's
standalone behavior and the tests that assert it (see Test Strategy); the FFIReader code becomes unreachable
from the Calc decision (its removal vs. retention as dead plumbing is a PLAN-time detail).

**Operator-id / metrics**: the merged operator keeps the **source's** `auron_operator_id` (offset-commit
keys derive from it) and folds the Calc's metric subtree under that operator's metric group.

Rationale for the watermark gate: the Kafka source runs `WatermarkGenerator.onEvent` per **emitted**
record (`AuronKafkaSourceFunction.java:338`). A native `Filter` below the generator would hide
dropped-partition records from it, stalling that partition's watermark and deadlocking downstream
event-time windows. Every mainstream engine keeps the filter above the watermark
(`research-watermark-ordering.md`). Gating on `watermarkStrategy == null` is the precise, provably
safe subset; it still delivers the conversion win for the large class of stateless streaming Calc
jobs and reuses the exact same mechanism the event-time follow-up will extend.

## Detailed Design

### Components
The existing `SupportsAuronNative` interface (`auron-flink-runtime/.../operator/SupportsAuronNative.java`):
`List<PhysicalPlanNode> getPhysicalPlanNodes()`, `RowType getOutputType()`,
`String getAuronOperatorId()`, `MetricNode getMetricNode()`.

1. **`FlinkAuronStreamSource` (new) + `LegacySourceTransformationTranslator` rewrite** — source-operator
   infra not yet in the code. `FlinkAuronStreamSource extends StreamSource implements FlinkAuronOperator`,
   so the source *operator* (not just its function) is a `SupportsAuronNative` reachable at the
   Transformation level and able to run a fused plan. The rewritten `LegacySourceTransformationTranslator`
   swaps the stock `StreamSource` for `FlinkAuronStreamSource` when the user function is a
   `FlinkAuronFunction`. This is the enabler the whole-stage merge needs; #1865 introduces it (the native
   Kafka source from #1847/#2060–#2062 today wires a plain `StreamSource`).
2. **Merge detection in `StreamExecCalc.translateToPlanInternal`** —
   ```
   Transformation<RowData> upstream = (Transformation<RowData>) getInputEdges().get(0).translateToPlan(planner);
   SupportsAuronNative nativeSrc = asNativeSource(upstream);   // operator/function instanceof SupportsAuronNative
   if (nativeSrc != null && watermarkSafe(nativeSrc) && auronPlan.isPresent()) {  // auronPlan = Project[Filter]
       PhysicalPlanNode srcLeaf = nativeSrc.getPhysicalPlanNodes().get(0);        // the KafkaScan
       PhysicalPlanNode fused = rebaseLeafOntoSource(auronPlan.get(), srcLeaf);   // FFIReader leaf -> KafkaScan
       return runMergedInSource(upstream, fused, outputRowType);                  // Calc operator eliminated
   }
   return <stock Flink Calc>;                                                    // non-fusible fallback
   ```
   `rebaseLeafOntoSource` does more than swap the leaf — it also **reconciles the input schema** (see
   the next subsection). The source's `KafkaScan` emits the Calc's logical columns **prefixed by three
   metadata columns** (`partition`, `offset`, `timestamp`), whereas the Calc's `Filter`/`Project`
   expressions were converted against the logical-only schema. So the swap must shift the Calc's input
   column references past the metadata prefix and keep the metadata columns in the fused output.
   `runMergedInSource` reconfigures the `FlinkAuronStreamSource`/`AuronKafkaSourceFunction` to run the
   fused plan and emit the projected output type.

### Input-schema reconciliation (metadata columns)
The native `KafkaScan` output schema is **not** the Calc's logical input schema. The source prepends three
metadata columns — `[partition (INT), offset (BIGINT), timestamp (BIGINT), …logical columns]`
(`AuronKafkaSourceFunction.java:304-308`) — and the source's `run()` loop reads them back by their fixed
positions: `FlinkArrowReader.create(batch, type, 3)` exposes the logical columns at `0,1,…` and the
metadata at negative indices `getInt(-3)` / `getLong(-2)` / `getLong(-1)` for partition / offset / Kafka
timestamp (`AuronKafkaSourceFunction.java:328-333, 357-364`). Those drive offset-commit and
`collectWithTimestamp`.

The Calc, by contrast, converts its `Filter`/`Project` `RexInputRef`s against the **logical-only** input
schema (`StreamExecCalc.tryBuildAuronPlan` builds the `ConverterContext` from `inputRowType` and the
`FFIReader` schema from `convertToAuronSchema(inputRowType, false)` — no metadata; `StreamExecCalc.java:240,269`).
So `RexInputRef(i)` becomes native `Column` ordinal `i`, 0-based over the logical columns. A bare
`FFIReader → KafkaScan` leaf swap would therefore leave every Calc column reference off by the metadata
count (`col@0` would read `partition`, not the first logical column).

`rebaseLeafOntoSource` reconciles this in three steps, with `M = 3` (the metadata-column count):
1. **Shift** every input `Column` ordinal in the Calc's `Filter` and `Project` expressions by `+M`, so a
   reference to logical column `i` reads physical column `i + M`.
2. **Prepend `M` passthrough identity columns** (`partition@0`, `offset@1`, `timestamp@2`) to the merged
   `Projection`, so the fused output stays `[partition, offset, timestamp, …projected logical columns]`.
3. **Declare the merged output RowType** as that metadata-prefixed type, so the existing `run()` loop —
   `FlinkArrowReader.create(merged, 3)` plus the negative-index reads — works unchanged and continues to
   feed offset-commit and `collectWithTimestamp` (and, in the #2315 follow-up, the watermark generator,
   which needs the preserved timestamp column).

The shift can be realized either by re-running the Calc conversion with a metadata-aware
`ConverterContext` that offsets `RexInputRef`→`Column` by `M` (no proto surgery), or by rewriting the
ordinals on the already-built `PhysicalExprNode` tree; the choice is a PLAN-time detail. Either way the
native engine is unchanged — `Project[Filter[KafkaScan]]` with shifted ordinals and a metadata-prefixed
projection is plain DataFusion.
3. **Merged execution in the source** — `AuronKafkaSourceFunction` accepts the fused native sub-plan +
   projected output RowType (it already builds & runs a `PhysicalPlanNode` and exposes it via
   `getPhysicalPlanNodes()`); in `run()` it reads the fused output and emits projected `RowData` once.
   Offset-commit/checkpoint untouched (scan-level, independent of plan shape). `getOutputType()` /
   `getMetricNode()` reflect the merged plan.
4. **Native engine** — no change. `Project[Filter[KafkaScan]]` is already expressible and runs as one
   DataFusion pipeline (`auron.proto`, `planner.rs`, `kafka_scan_exec.rs`).

### Data flow (fused, no-watermark case)
```
KafkaScan(Arrow [meta3, logical…]) → Filter(refs +3) → Project([meta3 passthrough, projected logical])
  → [one] Arrow→RowData → read meta via -3/-2/-1 → collectWithTimestamp(ts=kafka), commit offset
```
vs. today: `KafkaScan(Arrow)→RowData→[FFI]Arrow→Filter→Project→Arrow→RowData`. The fused output keeps the
`[partition, offset, timestamp, …]` prefix so the source's existing per-record bookkeeping is untouched.

### Known implementation risks (resolve in Execute)
- **Metadata-column reconciliation** (core, see *Input-schema reconciliation* above): the fused plan must
  shift the Calc's input column refs by the metadata count and keep `[partition, offset, timestamp]` as a
  passthrough prefix in the merged projection, so the source's negative-index reads (`getInt(-3)` /
  `getLong(-2)` / `getLong(-1)`) and `FlinkArrowReader.create(merged, 3)` still work. Verify the
  metadata-aware ordinal shift against unit + IT plan-shape assertions.
- **Operator-id / metrics unification**: keep the source's `auron_operator_id` (offset-commit keys
  derive from it) and fold the Calc's metric subtree under the merged operator's metric group.
- **Eligibility for partial fusion**: if the Calc has a filter AND an event-time watermark is present →
  no fuse (this PR; full event-time support is #2315). Projection-only-with-watermark is also out of this
  PR (folds into #2315).

## Prior Art Comparison

Gluten's Flink backend does **not** fuse operators yet — its `OperatorChainSliceGraphGenerator` notes
*"we don't coalesce operators into the same velox plan at present,"* giving each operator a standalone
plan with an `EmptyNode` placeholder leaf (the same shape as the standalone FFIReader-leaf Calc #1853
introduced, which this design's strict gating replaces with either fusion or a stock Flink Calc). The
canonical fusion reference is therefore the whole-stage-transform pattern from Gluten's **Spark** backend
(`WholeStageTransformer`), which Auron's own Spark plan-merge already mirrors — so this design ports the
Auron-Spark model.

| Aspect | This design (Flink #1865) | Auron Spark | Flink stock planner | Gluten-Flink |
|---|---|---|---|---|
| Merge trigger | Calc detects native source input | `isNative(child)` bottom-up | n/a (no native fusion) | none (each op standalone) |
| Plan nesting | `Project[Filter[KafkaScan]]` one `PhysicalPlanNode` | parent `.setInput(childPlan)` | n/a | `EmptyNode` placeholder per op |
| Boundary conversion | one Arrow→RowData at tail | one at native-subtree edge | per operator | per operator |
| Watermark vs filter | filter NOT pushed below watermark (gate) | EventTimeWatermark separate node | `…AcrossCalc` keeps filter above | watermark separate Flink op |
| Operator elimination | Calc returns merged source transformation | native ops collapse in one RDD | n/a | n/a |

## Dependencies
No new Maven/Cargo deps. No proto/Rust change. `flink-table-planner` already compile-scope.

## Test Strategy
- **Unit (planner module)**: merge-eligibility detection — native source + no watermark + convertible
  Calc → merged transformation (no Calc operator), plan-shape assertion `Project[Filter[KafkaScan]]`;
  each negative case (non-native source input, watermark present, non-convertible Calc) → **stock Flink
  Calc** (no native conversion, no FFIReader island).
- **#1853 test reconciliation**: strict gating drops the standalone FFIReader Calc, so existing #1853
  tests that assert FFIReader conversion change. Those whose Calc input is the native Kafka source with no
  watermark now assert the **fused** plan (`Project[Filter[KafkaScan]]`, no separate Calc operator) — row
  sets unchanged. Those feeding a Calc from a non-native / mock input now assert the **Flink-Calc**
  fallback. Audit `StreamExecCalcTest`, `AuronFlinkCalcITCase`, `AuronCalcRewriteITCase` during Execute.
- **Integration (`AuronKafkaSource*` IT)**: end-to-end `SELECT proj… FROM kafka WHERE pred` on the
  no-watermark path — assert row-set correctness, single operator in the job graph, offsets commit.
  A watermark-present IT asserts the non-fused path still produces correct results and the watermark
  advances (the Calc runs as a stock Flink Calc above the native source).
- **No native tests** — native engine unchanged.

## Out of Scope (follow-up issues)
| Item | Why deferred |
|---|---|
| Event-time watermark + filter fusion (native watermark relocation from pre-filter scan ts) | Hard correctness case; needs native-engine watermark support; tracked as #2315 |
| Projection-only pushdown under event-time watermark | Folds into the event-time watermark work (#2315) |
| Fusing Calc with non-Kafka sources / other native operators (agg, join) | Phase 2/3 scope of #1264 |
| Multi-input / join fusion | Phase 3 |

## Alternatives Considered
- **Runtime handshake (B)** and **FFIReader→KafkaScan leaf swap in the Calc operator (C)** — rejected
  above (incompatible leaves / harder coordination).
- **Graph-level ExecNode merge pass** — no hook exists in Auron or stock Flink at the ExecNodeGraph
  level usable here; detection inside `StreamExecCalc.translateToPlanInternal` (which runs after the
  source in bottom-up order) is the natural, lower-risk location.
- **Shadow `StreamExecTableSourceScan` with a bespoke plan accessor** — unnecessary; the existing
  `SupportsAuronNative` interface already exposes the source's native plan, so the merge reads it
  directly rather than introducing another shadowed source ExecNode.

## Scope & conventions
The keystones are in place: #1853 (native Calc shadow) is merged, and the native Kafka source
(#1847/#2060–#2062) already runs `KafkaScan` natively and exposes it via `SupportsAuronNative`. The
rewritten `LegacySourceTransformationTranslator` is a shadowed Flink class, so it follows the established
shadow-deployment convention (`FlinkAuronExecNode` marker + `META-INF/auron/shadowed-flink-execnodes.txt`
registry + assembly `<exclude>` + `AssemblyJarStructureIT`).

## Design-review resolution

All design-review questions are resolved; the decisions above reflect the agreed direction.

- **Event-time watermark handling (Rev 2)** — fuse only when the source has no event-time watermark
  (`watermarkStrategy == null`); event-time fusion needs native-engine watermark support and is tracked
  separately as **#2315**. Projection-only-under-watermark folds into #2315.
- **Input-schema reconciliation (Rev 2)** — the merge is not a bare leaf swap: the `KafkaScan` output
  prepends three metadata columns (`partition`, `offset`, `timestamp`), so the Calc's input column
  references are shifted by the metadata count and the metadata columns are kept as a passthrough prefix
  in the merged projection (see *Input-schema reconciliation*), keeping the source's per-record
  offset-commit and `collectWithTimestamp` bookkeeping unchanged.
- **Operator-id unification (Rev 3)** — confirmed: the merged operator keeps the **source's**
  `auron_operator_id` and folds the Calc's metrics under it.
- **Strict gating (Rev 3)** — confirmed: a Calc converts to native **only when it can fuse into a native
  upstream chain**; otherwise it stays a stock Flink Calc. The standalone FFIReader native-Calc path
  (#1853's current default) is dropped, so a non-fusible Calc never runs as a lone native island (which
  would pay R2C+C2R for no net win and risk misrepresenting Flink-on-Auron's performance to new users).
  This changes #1853's standalone behavior and tests — see Test Strategy.
- **Function-driven detection; `FlinkAuronStreamSource` + translator shadow dropped (Rev 4)** — an
  implementation-phase verification established that the merge does not need a source-operator type or a
  `LegacySourceTransformationTranslator` shadow. Transformation translators run only after the full
  `Transformation` DAG is built, so when the Calc translates, its upstream still holds the stock
  `StreamSource` and the `AuronKafkaSourceFunction` is reachable via `getOperator().getUserFunction()`.
  No code consumes an operator-level `SupportsAuronNative` identity (the source self-integrates its plan in
  `open()` and metrics in `run()`). The merge therefore detects and hands the fused plan off through the
  source **function** directly. Operator-level identity (`FlinkAuronStreamSource`) is deferred to the
  future `ExecNodeGraphProcessor`-based merge (Phase 2/3 of #1264) that would actually read it. Detailed
  Design component 1 (the source-operator infra) is superseded by this function-driven path.

## Rev 6 — Graph-level fusion via `ExecNodeGraphProcessor` (supersedes the in-`StreamExecCalc` merge)

Design review converged on moving the fusion off the per-node `StreamExecCalc` and onto a graph-level
pass. The in-`StreamExecCalc` approach decides a whole-graph property — "does this source have exactly
one consumer?" — from a node that structurally cannot see the graph (Flink's exec graph is
one-directional), so it inferred sole-consumership from session reuse config. That proxy is both too
conservative (nothing fuses under default reuse) and unsound for compiled plans (`COMPILE PLAN` freezes
source sharing a later session's reuse config no longer reflects). Rev 6 moves the decision to where the
graph is actually visible.

### Architecture

A new `ExecNodeGraphProcessor` is installed by shadowing `StreamPlanner` (the same classpath-overlay
convention already used for `StreamExecCalc`; `StreamPlanner.getExecNodeGraphProcessors()` is empty in
stream mode, so nothing is displaced). `PlannerBase.translateToExecNodeGraph` runs the processor over the
whole `ExecNodeGraph` *before* Transformation translation. The processor:

1. Builds a reverse fan-out map in one DFS from `ExecNodeGraph.getRootNodes()` over
   `ExecNode.getInputEdges()` / `ExecEdge.getSource()`, keyed by stable `ExecNode.getId()`, giving a
   **provable** per-source consumer count. The count is **per consuming edge with visited-node dedup
   across all roots** — a node reachable from multiple roots is counted once per distinct consumer edge,
   not once per traversal path, so a genuinely shared source can never be miscounted as `count == 1`.
2. Recognizes a native Auron source from a stock `StreamExecTableSourceScan` via
   `getTableSourceSpec().getScanTableSource(flinkContext, typeFactory)`, then keys on a **marker
   interface** the source implements (e.g. `FlinkAuronDynamicTableSource`), **not** the concrete
   `AuronKafkaDynamicTableSource` class — this preserves the marker-based detection that merged in
   `99403ef2` so any future native source is a fusion target. The `DynamicTableSource` is lazy-cached and
   reused at translation; `FlinkContext`/`FlinkTypeFactory` come from `ProcessorContext.getPlanner()`.
3. For each `source → Calc` chain where the source is native, **sole-consumer** (count == 1), unwatermarked,
   and the Calc is fully convertible, builds the merged native plan (reusing `tryBuildAuronPlan` + the
   metadata-passthrough splice) and **stages it on the marker source instance**. This needs **two new
   carry points**, both easily-omitted and both silent-no-fusion if forgotten: (a) a non-transient
   staged-plan field carried in `AuronKafkaDynamicTableSource.copy()` (the same trap that dropped
   `watermarkStrategy` until `bcd110bd`), and (b) an explicit forward in `getScanRuntimeProvider`
   (`:84-118`), which today forwards only `watermarkStrategy`/`mockData` — it must also pass the staged
   plan into the function's `setMergedCalcPlan`. A test asserts the function actually received the plan.

**Annotate-and-skip, not node removal.** `ExecNodeBase.outputType` is `private final` with no setter, so a
processor cannot re-type or remove the scan/Calc node in place. Instead the processor only *decides and
stages*; the shadowed `StreamExecCalc` keeps a **thin mechanical** branch that fires when its source
already carries a staged plan — it re-types the upstream `Transformation` to the projected output
(`Transformation.setOutputType`, a Transformation-level call) and returns it, emitting no standalone
operator for the fused Calc. All detection and the sole-consumer decision live in the processor; the Calc
no longer reaches across to *decide*, only completes a decision made graph-level (forced by ExecNode
output-type immutability).

### Fusion is additive (supersedes the Rev-3 standalone-drop)

Rev 3 dropped the standalone FFIReader native-Calc path (#1853) so a non-fusible convertible Calc fell
back to Flink codegen. Rev 6 reverses that: **fusion is purely additive on top of #1853.** A convertible
Calc that the processor fuses runs as one native plan; *every other* convertible Calc still emits the
standalone native Calc (#1853). A Calc falls back to Flink codegen only when it is not convertible (honoring
`FAIL_BACK_FLINK_ENGINE_ENABLED`). This removes the default-config regression (under graph-level counting,
sole-consumer chains now fuse even under default reuse), restores strict-mode native-or-throw, and keeps
`FlinkAuronCalcOperator` live (the #2328 removal premise — that the operator is orphaned — no longer holds).

### Scope split

- **#1865 (this PR):** the graph-level fusion mechanism + **single-consumer** `source → Calc` fusion
  (correct under default config).
- **#2329:** **multi-consumer** fusion — one shared source feeding several Calcs — built on the same
  processor (when consumer count > 1, Rev 6 simply does not fuse).

### Reuse / remove map

- **Reused unchanged:** `tryBuildAuronPlan`, the metadata-passthrough splice + by-name column resolution,
  the `FlinkAuronFunction` marker (identity hook), `AuronKafkaSourceFunction.setMergedCalcPlan` /
  no-watermark emit fix, non-transient merged fields.
- **Removed from `StreamExecCalc`:** the reach-across detection (`asNativeFusibleSource` + the
  `LegacySourceTransformation` unwrap), the session-config gate (`isSourceFusionSafe`), and the
  decision logic — all move into the processor.
- **Added:** the shadowed `StreamPlanner` + the `ExecNodeGraphProcessor`; the staged-plan field +
  `copy()` carry on `AuronKafkaDynamicTableSource`.

### Review findings folded in

- Strict-mode native-or-throw + javadoc — restored by the additive decision (a convertible Calc is always
  native; the class javadoc is reconciled to "native when convertible — fused or standalone; Flink codegen
  only when not convertible").
- Compiled-plan unsoundness — resolved, but via *de-fusion*, not fused serialization. Annotate-and-skip
  stages the plan on a runtime `DynamicTableSource` instance, which is **not** part of the serialized
  ExecNode graph, so `COMPILE PLAN` writes the stock (unfused) graph. Plan *restore* (`translatePlan`)
  bypasses `ExecNodeGraphProcessor`s entirely, so a compiled plan simply **does not fuse** on restore —
  correct results, fusion perf reverts. Crucially this means a session reuse-config change can never
  re-permit fusion into a shared source (the corruption `:271` raised) because restore fuses *nothing*.
  The honest tradeoff: fusion is a fresh-plan-only optimization; `COMPILE PLAN` workflows run unfused.
- Metadata count single-source-of-truth — derive the reader offset from `KAFKA_AURON_META_FIELDS.size()`
  instead of the hardcoded `3`.
- Reserved-meta-name collisions — checked at the processor gate (so a colliding name falls back at plan
  time instead of crash-looping at `open()`), and on input names as well as output names.
- Watermark-gate test sensitivity — add a watermark-dependent (windowed) or operator-topology assertion so
  the test fails if the gate is deleted.

### Alignment with #1264 / Flink track

`#1264` Phase 1 is whole-stage native operator merging for Flink. A graph-level `ExecNodeGraphProcessor`
that fuses adjacent native operators is the general substrate that the later operator phases (Agg, Join)
reuse — the same direction the Spark side already realizes through plan-merge. Rev 6 keeps the host-agnostic
contracts intact (metadata-by-name, meta-cols-first, source-owned `auron_operator_id`, no fusion over a
watermarked source → #2315).

### Alignment with AIP-1

An element-by-element audit against AIP-1 (PoC + "Introduce Flink integration of native engine") found Rev 6
**aligned, no conflict requiring an AIP amendment**:

- *Whole-stage operator merging* — an explicit AIP-1 Phase-1 (Calc) capability (PoC p3; Intro p14); the AIP
  diagram collapses per-boundary R2C/C2R into one tail conversion (PoC p1). Rev 6 realizes it directly.
- *Fusion mechanism* — **AIP-silent.** The AIP prescribes only "rewrite `StreamExecCalc`" for making a
  *single* Calc native (PoC p5–6); it does not prescribe how a *chain* fuses. The `ExecNodeGraphProcessor`
  fills an unspecified gap (silence ≠ conflict).
- *Additive fallback* — **consistent, and more faithful than Rev 3.** AIP fallback is narrow: Flink only
  "for operators we do not yet support," `FAIL_BACK_FLINK_ENGINE_ENABLED` default `true` (PoC p1–2, p8).
  Additive maps 1:1; Rev 3's perf-heuristic fallback was the off-AIP element.
- *Phasing* — AIP phases by operator class (Calc→Agg→Join). The #1865(single-consumer)/#2329(multi-consumer)
  split is finer sequencing *within* Phase-1 Calc — consistent; both PRs stay Calc-only.
- *Justified deviation* — Rev 4/6 drops `FlinkAuronStreamSource` + the `LegacySourceTransformationTranslator`
  rewrite the AIP lists (PoC p8; Intro p13). Forced by framework constraint (`ExecNodeBase.outputType` is
  `private final`; translators run after the Transformation DAG is built) with no consumer of the
  operator-level identity. A doc-reconciliation note for the AIP authors, not a design blocker.

Identity/metrics/watermark/COMPILE-PLAN behaviors are AIP-silent (no contract contradicted).

### #2328 re-scope (consequence of the additive decision)

#2328 was filed to remove the now-orphaned `FlinkAuronCalcOperator`. The additive decision keeps the
standalone native Calc live, so that operator is **not** orphaned — #2328's removal premise dissolves. Action:
re-scope/close #2328, and fold its remaining valid item — deduplicating the two near-identical FFIReader-leaf
walkers (`spliceScanIntoLeaf` ≈ `FlinkAuronCalcOperator.injectFfiReaderLeaf`), which now both ship live — into
this PR's Execute so the walkers can't drift.

### Lone-native-island tradeoff (additive, explicit — not silent)

Additive re-accepts what Rev 3 deliberately removed: a convertible-but-non-fusible Calc runs as a
standalone native island paying R2C at input + C2R at output, which is not guaranteed to beat Flink
codegen. This is **not a new regression** — it is exactly #1853's already-merged default behavior; Rev 6
simply stops #1865 from silently reverting it, consistent with the AIP's "Flink only for unsupported
operators" fallback philosophy. Recorded here so it's a ratified tradeoff, not an unstated one.

### Commit-0 verification gate (must pass before building the processor)

The cached-`DynamicTableSource`-instance reuse (the scan's translation must dereference the *same* spec
instance the processor staged onto) is **load-bearing for the whole handoff** and only bytecode-inferred.
A ~20-line runtime spike gates the design: if the scan rebuilds a fresh source, the staged plan never
reaches the function and every fusion silently no-ops. If the spike fails, the fallback is to shadow
`StreamExecTableSourceScan` so it carries the staged plan itself — a materially larger change, so we
confirm before committing to the lighter path.

### Watermark — hard `open()`-time guard (failure here is corruption, not slowdown)

Whether watermark push-down has run by processor time is the one inferred item whose failure *corrupts*
(fusing a watermarked source strips per-record event-time). So the processor's no-fuse-over-watermark
check is **not** the only line of defence: `AuronKafkaSourceFunction.applyMergedCalcPlan` re-checks the
same condition at `open()` and **fails fast** — it throws `IllegalStateException` if a merged plan and a
watermark ever coexist, rather than silently applying the staged plan. The throw (not a graceful
un-fuse) is the only safe action: by `open()` time the planner has already committed to fusion — the
downstream Calc was re-typed to the projected output and emitted no standalone operator — so the source
cannot fall back to an unfused plan without breaking the downstream type contract. The processor gate is
the optimization that keeps this from ever triggering; the `open()` guard is the correctness backstop
for a planner-gate bypass, a future source type, or a watermark-push-down timing edge.
