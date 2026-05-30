# Design — AURON #2291: Ship a shaded `auron-flink-planner` jar that replaces the Flink planner

**Date**: 2026-05-29
**Status**: Rev 1 — awaiting reviewer approval
**Issue**: https://github.com/apache/auron/issues/2291 (follow-up to PR #2283 / #1853; sub-issue of #1264)
**Source of truth**: `docs/PR-AURON-1853/AURON-1853-DESIGN.md` Rev 4 (deployment-model section) — the relevant
parts are **inlined below** (§"Background — A1 vs A2, inlined from #2283 / Rev 4") so this document is
self-contained; no need to open the #1853 doc.

## Problem Statement

PR #2283 landed the JAR-shadowing mechanism: Auron ships a class at the same FQCN as Flink's
`org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc`, and whichever copy the JVM loads
first wins. Today's deployment (**A1**) drops `auron-flink-assembly.jar` into `$FLINK_HOME/lib/` *alongside*
Flink's planner and relies on classpath-traversal ordering to make Auron's class win — not spec-guaranteed
across JVM vendors or container rebuilds. AURON #2291 replaces that with **A2**: ship a shaded jar that
*structurally* contains exactly one `StreamExecCalc` (Auron's), so activation no longer depends on ordering.
The next ExecNode shadow PRs (#1860/#1861/#1864) should land on A2, not A1.

## Background — A1 vs A2, inlined from #2283 / Rev 4

> The following is reproduced from `AURON-1853-DESIGN.md` Rev 4 (the deployment-model discussion raised in
> PR #2283 Round 2 review) so this design stands alone. It establishes the A1/A2 definitions, why a third
> "just delete the planner jar" option doesn't work, the Spark-vs-Flink capability gap, and why A2 is the
> agreed direction. **One correction applies** — Rev 4's "replace `flink-table-planner.jar`" wording is
> superseded by the loader-swap procedure in the next section (Rev 5); everything else carries forward.

### The shadowing mechanism (context)

The architecture overrides Flink's built-in `StreamExecCalc` by shipping an Auron class with the same FQCN
(`org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc`). The JVM caches one class per FQCN,
so whichever class loads first wins for the JVM's lifetime. #2283 specified the *mechanism* (the shadow
class) but not the *deployment procedure* that guarantees Auron's class is the one loaded. A1 and A2 are the
two procedures.

### The two options

- **A1 — JAR overlay (the current implementation being replaced).** The user drops Auron's jar into
  `$FLINK_HOME/lib/` *alongside* Flink's planner. Both jars are present; the JVM resolves `StreamExecCalc`
  to whichever appears first in its directory traversal of `lib/` — typically alphabetical, **but not
  spec-guaranteed across JVM vendors**. If filesystem ordering changes (JVM vendor, container image rebuild,
  launch-script classpath manipulation), Auron's class is silently no longer resolved — no error, no warning.
- **A2 — shaded uber-jar replacement (this PR).** Auron ships a fat jar containing all of the planner's
  content with Auron's `StreamExecCalc` (and future overrides) substituted in. The user swaps it into
  `lib/`. **Only one `StreamExecCalc` exists on the classpath — Auron's — so activation is structural, not
  ordering-dependent.**

A natural-seeming third option — "tell users to remove the planner jar and keep only Auron's overlay jar" —
**does not work without shading**: Auron's `StreamExecCalc` extends `CommonExecCalc` and transitively
depends on other classes inside the planner jar; removing it `NoClassDefFoundError`s at class-load. To make
"only Auron's jar in `lib/`" actually function, that jar must contain the planner content itself — which is
exactly A2.

### Why A2 (the two angles from Round 2 review)

- **Robustness** — A1's classpath-ordering assumption is unenforced and can silently break. A2 eliminates it
  structurally (there is no second `StreamExecCalc` to misorder against).
- **Simplification** — the deployment story becomes a single deterministic jar swap rather than
  overlay-plus-ordering.

### Spark-vs-Flink capability gap (why this problem is Flink-only)

Auron's Spark side has no such problem because Spark exposes a supported plan-rewrite SPI; Flink 1.18 does
not. A true Spark-grade UX on Flink would require either upstreaming a Flink `ExecNode`-replacement SPI
(verified absent across 1.18 / 1.20 / 2.0 / master — no FLIP in flight) or A2.

| Dimension | Auron-Spark | Auron-Flink |
|---|---|---|
| Plan-rewrite hook | `SparkSessionExtensions.injectColumnar` (supported SPI) | None for `ExecNode` substitution on Flink 1.18 |
| Activation mechanism | `spark.sql.extensions=…` in `spark-defaults.conf` | FQCN shadow of `StreamExecCalc` |
| Classpath-order sensitivity | None (Spark reflects the named class) | **Yes (A1)** — first `StreamExecCalc` to load wins; **None (A2)** — structural |
| Prior art | n/a | Gluten `gluten-flink/` ships 13 shadow `StreamExec*` classes — same mechanism as A1 |

### Why not copy Gluten's exact procedure? (the A1-explicit fallback)

Gluten's documented procedure ([`gluten-flink/docs/Flink.md`](https://github.com/apache/incubator-gluten/blob/main/gluten-flink/docs/Flink.md))
does *not* drop jars in `lib/`. It places them in a dedicated `gluten_lib/` and edits `bin/config.sh` to
**prepend** that dir to `FLINK_CLASSPATH`. Call this **A1-explicit** — more deterministic than A1 (the edit
makes ordering an explicit operator action) but it still asks the user to reason about classpath ordering.

| Approach | User steps | Activation determinism |
|---|---|---|
| **A1** (replaced) | 1 (drop jar in `lib/`) | Implicit — JVM directory traversal order |
| **A1-explicit** (Gluten's) | 2 (place jars in a dir; edit `config.sh`) | Explicit — shell-script classpath prepend |
| **A2** (this PR) | 1 (swap one jar in `lib/`) | Structural — only one `StreamExecCalc` class exists |

A2 dominates A1-explicit on **both** user steps (1 vs 2 — and editing `config.sh` is fragile in
containerized/managed Flink where it's regenerated or version-locked) **and** determinism (no later operator
action can re-misorder the classpath, because there is no other jar to misorder against). A1-explicit
remains a doc-only fallback if A2's scope (shading, artifact size, NOTICE) were judged unacceptable.

### Why land A2 now (pre-GA timing)

Auron's Flink support is pre-GA — no installed users — so A1→A2 is purely internal work, no migration cost.
The future ExecNode shadows (#1860–#1865) ride the same deployment model: doing A2 upfront means each
subsequent shadow lands against the clean model instead of being re-validated on an A2 cutover later. Costs
are bounded — shade config is small, the ~30-52 MB artifact is normal (`flink-dist*.jar` is ~120 MB), and
license handling is one-time work Apache projects do regularly (cf. `flink-shaded`, `spark-hadoop-cloud`).

— end of inlined Rev 4 background —

## Key research finding that reshapes the deployment (DESIGN Rev 5 reconciliation)

The issue and DESIGN Rev 4 both say *"replace `$FLINK_HOME/lib/flink-table-planner.jar`."* **That file does
not exist in a default Flink 1.18 install.** `lib/` ships `flink-table-planner-loader-1.18.1.jar` (a small
bootstrapper) while the real planner — `flink-table-planner_2.12-1.18.1.jar`, the ~52 MB fat jar that
actually contains `StreamExecCalc` — sits in `opt/` and is loaded through an isolating
`ComponentClassLoader`. A sibling jar in `lib/` is loaded by a different classloader and can never shadow
the loader-internal class, and Flink forbids two planners co-existing on the classpath.

**A2 remains viable via Flink's own documented loader→non-loader swap.** This is the corrected deployment
procedure (supersedes the Rev 4 wording — recorded here as the Rev 5 reconciliation, to flow into the
#1892 user docs):

> **Deploy A2 (Flink 1.18):** remove `flink-table-planner-loader-1.18.1.jar` from `$FLINK_HOME/lib/`, then
> place `auron-flink-planner-shaded-<version>.jar` into `$FLINK_HOME/lib/`. (The shaded jar already contains
> the full non-loader planner content, so `opt/flink-table-planner_2.12-*.jar` is not needed on the
> classpath.) Auron's runtime bundle jar is also placed in `lib/` — see the Decision below.

This aligns with the repo, which already depends on the **non-loader** `flink-table-planner_2.12` at compile
scope (`auron-flink-planner/pom.xml:130-134`).

## Approach Candidates

The shadow *mechanism* (Auron's class in Flink's package) is fixed. Two decisions remain: **(D1)** how to
build the shaded planner jar, and **(D2 = Q2)** where Auron's *runtime* (operators, FFI, `libauron.so`)
ships once the planner jar is a clean replacement.

### D1 — making Auron's `StreamExecCalc` win inside one fat jar

- **Candidate D1-a — dependency/artifactSet ordering.** Rely on listing `auron-flink-planner` before the
  Flink artifact so its class is written first. **Rejected:** maven-shade-plugin 3.5.2 documents no
  ordering-based winner for overlapping classes; behavior is not contractual.
- **Candidate D1-b — `<filters>` exclude (chosen).** Exclude
  `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` from the *Flink* artifact so
  only Auron's copy survives the merge. Deterministic and documented; filter shape already proven in-repo
  at `auron-flink-assembly/pom.xml:65-73`. Stock `StreamExecCalc` has **no inner classes** (verified via
  `unzip -l`), and Auron's source adds none, so a single `<exclude>` is complete.

### D2 (Q2) — runtime placement

Both work mechanically. The native `libauron.so` ships as an `auron-core` resource and rides
`auron-core → auron-flink-runtime`; both planning-time (`StreamExecCalc`, JobManager) and execution-time
(`FlinkAuronCalcOperator`, TaskManager) classes must be on the cluster `lib/` classpath.

- **Candidate D2-A — two jars: pure planner-shaded jar + runtime bundle jar (chosen).** The shaded jar =
  Flink planner content + Auron's one `StreamExecCalc` override, nothing else. The runtime — `auron-flink-runtime`
  + `auron-core` + `libauron.so` — ships as its own fat jar (this is the *current* `auron-flink-assembly`
  with its planner dependency removed). User installs two jars in `lib/`.
  - **Pros:** the planner-replacement jar stays a clean, auditable "Flink planner with exactly one class
    swapped" — the structural smoke test means exactly what it says. Native binary + the EPL-licensed
    `jts-core` (bundled inside the upstream planner) and the runtime's `Factory` SPI stay out of the
    planner artifact, so no SPI-merge or native-classifier concerns leak into it. Matches Gluten's
    separation (runtime never bundled into the planner overlay).
  - **Cons:** user drops two jars instead of one (slight UX regression vs A1's single overlay jar). Both
    are unconditional `lib/` placements, so there is no ordering concern for either.
- **Candidate D2-B — one jar: fold runtime into the shaded jar.** Shaded jar depends on both planner and
  runtime; bundles everything + `libauron.so`.
  - **Pros:** single-jar install (preserves A1's one-jar UX).
  - **Cons:** `ServicesResourceTransformer` becomes mandatory (the runtime's
    `META-INF/services/org.apache.flink.table.factories.Factory` would otherwise clobber Flink's three
    `Default*Factory` entries); `libauron.so` needs OS-classifier handling; Arrow may need Spark-style
    relocation; the "Flink planner replacement" artifact now also carries a native binary and an
    EPL-mixed licensing surface, muddying the NOTICE and the structural test's meaning.

## Decision

**D1-b** (filter exclude) + **D2-A** (two jars: pure planner-shaded jar + repurposed runtime bundle jar).

Rationale: A2's whole value is *structural auditability* — "this jar is Flink's planner with exactly our one
class substituted." D2-A preserves that property literally; the structural smoke test then certifies a clean
artifact. The native/licensing/SPI surface stays isolated in the runtime bundle, which already exists as a
build target. The two-jar install is a minor, well-documented cost, and is consistent with — not a
violation of — "A2 replaces A1": A1's *ordering-dependent planner overlay* is gone; the runtime jar shadows
nothing, so its `lib/` placement carries no ordering risk.

If the reviewer weights single-jar UX above artifact purity, D2-B is the fallback and the SPEC/PLAN note the
delta (one module instead of two; `ServicesResourceTransformer` + native-classifier wiring added).

## Detailed Design

### Module structure

```
auron-flink-extension/
├── auron-flink-planner/            (unchanged — source of the StreamExecCalc override + converters)
├── auron-flink-runtime/            (unchanged — operators, FFI, FlinkAuronAdaptor)
├── auron-flink-planner-shaded/     (NEW — D1: planner replacement jar)
└── auron-flink-assembly/           (REPURPOSED — runtime bundle: drop the planner dependency)
```

### `auron-flink-planner-shaded` (new)

- `pom.xml`: parent `auron-flink-extension`; `packaging=jar`; ALv2 header copied from
  `auron-flink-assembly/pom.xml:2-17`. Single dependency: `auron-flink-planner` (compile) — transitively
  pulls Auron's override classes + the full `flink-table-planner_2.12:1.18.1` content.
- `maven-shade-plugin` 3.5.2 config:
  - `<filters>`: from artifact `org.apache.flink:flink-table-planner_2.12`, exclude
    `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` (the one class Auron
    overrides). Also keep the existing `META-INF/*.SF|DSA|RSA` exclusions.
  - **No `<relocations>`** — relocating `org.apache.flink.table.planner.*` would break the FQCN-identity
    shadow. (Mirrors why Spark's assembly excludes `org.apache.arrow.c.**`,
    `dev/mvn-build-helper/assembly/pom.xml:118-121`.)
  - `<transformers>`: `ServicesResourceTransformer` (merge `META-INF/services`, incl. the modern
    `org.apache.flink.table.factories.Factory` SPI), `ApacheNoticeResourceTransformer`,
    `ApacheLicenseResourceTransformer`, `ManifestResourceTransformer`.
  - Artifact name `auron-flink-planner-shaded-<version>.jar` (replaces main artifact; no relocation).
- `src/main/resources/META-INF/NOTICE` + `META-INF/LICENSE` (net-new — see Licensing).

### `auron-flink-assembly` (repurposed → runtime bundle)

- Remove the `auron-flink-planner` dependency (`auron-flink-assembly/pom.xml:35-39`); keep
  `auron-flink-runtime` (which transitively bundles `auron-core` + `libauron.so`). Result: a runtime-only
  fat jar. Keep the existing SPI transformer for the runtime's table factories. (Module dir name kept to
  minimize churn; its `<name>`/`<description>` updated to say "runtime bundle." An optional rename to
  `auron-flink-runtime-bundle` is noted as a follow-up, not done here.)

### Structural smoke test (build-time)

- New failsafe IT `ShadedPlannerJarStructureIT` in `auron-flink-planner-shaded`, running in the `verify`
  phase (after `package`/shade). Opens `target/auron-flink-planner-shaded-*.jar` via `java.util.jar.JarFile`
  and asserts:
  1. exactly **one** entry `…/exec/stream/StreamExecCalc.class`;
  2. it is **Auron's** — byte-scan the entry for the UTF-8 constant `org/apache/auron/flink/runtime/operator/FlinkAuronCalcOperator` (present in Auron's bytecode, `StreamExecCalc.java:27,191-192`); no classloading, no Flink on the test classpath;
  3. a **non-shadowed** `StreamExec*` (e.g. `StreamExecCalcBase`/another) is present — proves the planner
     content was actually bundled, not just Auron's one class;
  4. `META-INF/NOTICE` contains both "Apache Auron" and "Apache Flink" — cheap guard against a forgotten
     transformer.
- Requires net-new `maven-failsafe-plugin` wiring in the shaded module (repo has none today).

### Licensing (ASF compliance)

- `META-INF/NOTICE` (net-new source): Auron header + a statement that the jar bundles a **modified copy** of
  Apache Flink `flink-table-planner` (ALv2), bubbling up Flink's NOTICE attribution and an ALv2 §4(b)
  "modified by Apache Auron" note naming the substituted `StreamExecCalc`. Template: `flink-shaded-jackson`.
- `META-INF/LICENSE`: ALv2 verbatim **plus** a block for `jts-core 1.19.0` (**EPL v2.0**, embedded inside
  the upstream planner jar under `org/locationtech/jts`). Other embedded third-party (Calcite 1.32.0,
  Avatica 1.22.0, Guava 31.1, commons-codec/io) are ALv2 → NOTICE list only.
- Root `NOTICE`/`LICENSE`: **no change** — embedding occurs only in the binary shaded jar; per-module
  `META-INF` is the correct home.

### CI

- `.github/workflows/flink.yml`: add `auron-flink-planner-shaded` to the build matrix; its entry runs
  `mvn verify -Pflink-1.18 …` (not `test`) so the shade + structural IT execute. (Today `flink.yml:45,60`
  builds/tests only `auron-flink-planner -am` with surefire `test`.)

## Alignment with DESIGN Rev 4 / AIP (audit)

| Element | Status | Evidence / note |
|---|---|---|
| Shadow mechanism (Auron class in Flink package, FQCN identity) | **Aligned** | Unchanged from #2283 / Rev 4. No relocation. |
| Deployment procedure ("replace flink-table-planner.jar") | **Follow actual code → Rev 5 reconciliation** | Default Flink 1.18 has no `lib/flink-table-planner.jar`; corrected to the documented loader→non-loader swap. Flink 1.18 Table advanced-config docs. |
| A2 = shaded uber-jar replacement | **Aligned** | Rev 4 §"A2"; this PR implements it. |
| Runtime placement | **Justified deviation (clarified)** | Rev 4 didn't specify; D2-A keeps the shaded jar a pure planner replacement and ships runtime as its own bundle (Gluten-consistent). |
| "A2 replaces A1" | **Aligned** | A1 ordering-overlay retired; planner now structurally replaced. Runtime bundle carries no shadow/ordering concern. |
| Per-Flink-version re-shade | **Aligned (deferred content)** | Only 1.18 baseline exists; filter/transformer structure is version-agnostic. 1.20/2.0 deferred per scope. |
| Observability machinery (activation log, strict mode) | **Aligned** | Becomes structurally guaranteed under A2; kept as-is (Rev 4 §"under A2 … can be kept or simplified"). No change this PR. |

## Prior Art Comparison

| Aspect | This design (A2) | Gluten `gluten-flink` (A1-explicit) | Spark `auron` assembly |
|---|---|---|---|
| Planner activation | Structural (one class in fat jar) | Classpath prepend via `config.sh` | n/a (Spark uses extension API) |
| Shades flink-table-planner | **Yes** (this PR) | No (provided-scope thin jar) | n/a |
| Relocation of engine planner pkg | No (FQCN identity) | No | Excludes `org.apache.arrow.c.**` for C-ABI identity |
| Runtime placement | Separate bundle jar | Separate (never bundled) | Single assembly |

## Dependencies

No new external dependencies. `maven-shade-plugin` 3.5.2 (already in `pluginManagement`), `maven-failsafe-plugin`
(net-new, standard Apache-parent-managed). `flink-table-planner_2.12:1.18.1` reached transitively through
`auron-flink-planner` (compile scope).

## Test Strategy

- **Build-time structural IT** (above) — the agreed CI smoke assertion.
- **Manual jar-swap procedure** (reviewhelper, not CI): on a real Flink 1.18 home, delete the loader jar,
  drop in the two Auron jars, run a Calc SQL, confirm native conversion via the existing
  `"Auron StreamExecCalc shadow active"` activation log (and optionally `FAIL_BACK_FLINK_ENGINE_ENABLED=false`
  strict mode).
- No new unit tests for the planner/runtime classes — those are covered by #2283/#1853's existing suites;
  this PR is build/packaging.

## Out of Scope

| Item | Why | Tracking |
|---|---|---|
| Flink-on-Auron user deployment docs | Separate doc issue | #1892 (carry the Rev 5 loader-swap procedure) |
| Multi-Flink-version re-shading (1.20/2.0) | Those baselines aren't in the build yet | When 1.20/2.0 land |
| Wave-1 ExecNode shadows (#1860/#1861/#1864) per-FQCN excludes | This PR ships the structure; those add their own excludes (+ inner-class re-audit) | #1860/#1861/#1864 |
| Full automated Flink ITCase (jar-swap + SQL in CI) | Heavy/fragile; structural assertion + manual procedure chosen instead | — |
| Optional rename `auron-flink-assembly` → `auron-flink-runtime-bundle` | Churn; not required for correctness | follow-up |

## Alternatives Considered

- **D1-a ordering** — rejected (undocumented shade behavior).
- **D2-B single jar** — viable fallback if reviewer prioritizes one-jar UX; cost is SPI-merge + native
  classifier + mixed licensing in the planner artifact.
- **A1-explicit (Gluten's `config.sh` prepend)** — improves determinism without shading but still asks the
  user to reason about classpath ordering; rejected as the long-term model in Rev 4.
