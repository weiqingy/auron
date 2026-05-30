# Design — AURON #2291: Ship a shaded `auron-flink-planner` jar that replaces the Flink planner

**Date**: 2026-05-29
**Status**: Rev 2 — awaiting reviewer approval (Rev 2 reuses the existing assembly jar instead of a new module)
**Issue**: https://github.com/apache/auron/issues/2291 (follow-up to PR #2283 / #1853; sub-issue of #1264)
**Source of truth**: `docs/PR-AURON-1853/AURON-1853-DESIGN.md` Rev 4 (deployment-model section) — the relevant
parts are **inlined below** (§"Background — A1 vs A2, inlined from #2283 / Rev 4") so this document is
self-contained; no need to open the #1853 doc.

## Rev 2 Changes (2026-05-29)

Rev 1 proposed a **new `auron-flink-planner-shaded` module** producing a pure planner-replacement jar, plus
**repurposing `auron-flink-assembly` into a runtime-only bundle** (two jars). Rev 2 simplifies this to
**enhancing the existing `auron-flink-assembly` jar in place** (one jar, no new module), because that jar is
already the single Flink deployment artifact.

**Verified against the repo.** `auron-flink-assembly` shades `auron-flink-runtime` + `auron-flink-planner`
(`auron-flink-assembly/pom.xml:29-40`); `auron-flink-planner` depends on `flink-table-planner_2.12` at
**compile scope** (`auron-flink-planner/pom.xml:130-134`); shade bundles transitive compile deps and the
assembly's `artifactSet` excludes only hadoop/slf4j/flink-cep — **not** flink-table-planner. So the assembly
jar **already bundles** Auron runtime + Auron planner + the full Flink planner content, and is already the
single deployment artifact.

**Consequences for this design:**
1. **No new module.** The shaded artifact is the existing `auron-flink-assembly` jar, enhanced in place.
2. **One jar (Q2 resolved).** Runtime stays in the assembly jar alongside the planner — its established
   usage. Rev 1's D2-A (two jars) and D2-B (fold-runtime-into-a-new-module) are both **dropped**; runtime
   placement is no longer an open question.
3. **The latent bug this fixes (why #2291 is not a no-op).** Because the assembly already bundles *both*
   Auron's `StreamExecCalc` and Flink's stock `StreamExecCalc` (transitive) with **no `<filters>` exclude**,
   the surviving copy is decided by maven-shade's class-overlap ordering — **not contractually guaranteed.**
   It works internally today by incidental ordering; this PR makes Auron's copy win *structurally* by
   excluding the stock class. That filter — plus the ASF NOTICE and a structural test — is the real,
   minimal content of #2291.

The rest of the design (the D1 filter mechanics, the Rev 5 loader-swap deployment finding, licensing,
the structural-test approach) is unchanged; only its *home* moves from a new module to the assembly module.

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
> place the `auron-flink-assembly-<version>.jar` into `$FLINK_HOME/lib/`. The assembly jar already contains
> the full non-loader planner content **plus** Auron's runtime, so it is a single drop-in artifact and
> `opt/flink-table-planner_2.12-*.jar` is not needed on the classpath.

This aligns with the repo, which already depends on the **non-loader** `flink-table-planner_2.12` at compile
scope (`auron-flink-planner/pom.xml:130-134`).

## Approach Candidates

The shadow *mechanism* (Auron's class in Flink's package) is fixed, and Rev 2 settled the packaging
(enhance the existing `auron-flink-assembly` jar — one artifact). The one remaining technical decision is
**(D1)** how to make Auron's `StreamExecCalc` win deterministically over the stock copy that is already
bundled in that jar.

### D1 — making Auron's `StreamExecCalc` win inside the assembly jar

The assembly jar already contains *both* Auron's `StreamExecCalc` (from `auron-flink-planner`'s own classes)
and Flink's stock `StreamExecCalc` (transitive from `flink-table-planner_2.12`, compile scope). There is no
class-level filter today, so the survivor is whatever maven-shade writes last for that overlapping entry.

- **Candidate D1-a — rely on shade's class-overlap ordering.** Keep the current state / order dependencies
  so Auron's copy happens to win. **Rejected:** maven-shade-plugin 3.5.2 documents no contractual
  ordering-based winner for overlapping classes; today's behavior is incidental and can flip on a
  dependency-graph or plugin change. This is the same non-determinism A2 exists to remove — just inside one
  jar instead of across two.
- **Candidate D1-b — `<filters>` exclude (chosen).** Exclude
  `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` from the
  `org.apache.flink:flink-table-planner_2.12` artifact so only Auron's copy survives the merge.
  Deterministic and documented; the `<filter><artifact>…<excludes>` shape is already proven in-repo at
  `auron-flink-assembly/pom.xml:65-73`. Stock `StreamExecCalc` has **no inner classes** (verified via
  `unzip -l`), and Auron's source adds none, so a single `<exclude>` is complete.

## Decision

**D1-b** (filter exclude), applied to the existing **`auron-flink-assembly`** module — one jar, no new
module.

Rationale: per Rev 2, the assembly jar is already the single deployment artifact (runtime + Auron planner +
Flink planner content), so reusing it keeps the deployment simple for users and maintenance and avoids
module churn. The `<filters>` exclude converts the
jar's currently-incidental `StreamExecCalc` winner into a *structural* guarantee — exactly one
`StreamExecCalc` (Auron's) — which is the substance of A2. The ASF NOTICE and a structural test certify that
guarantee. Native binary, the EPL-licensed `jts-core`, and the runtime's `Factory` SPI already coexist in
this jar today and are unchanged by this PR, so they are pre-existing assembly concerns, not new risks
introduced here.

## Detailed Design

All changes are confined to the existing `auron-flink-assembly` module. No new module; `auron-flink-planner`
and `auron-flink-runtime` are unchanged.

### `auron-flink-assembly` shade config — add the determinism filter

The module already declares `maven-shade-plugin` (`auron-flink-assembly/pom.xml:42-86`). Add to its existing
`<filters>` block (which today holds only the `META-INF/*.SF|DSA|RSA` excludes) a class-level exclude:

- `<filter>` on artifact `org.apache.flink:flink-table-planner_2.12` excluding
  `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` — so only Auron's copy (from
  `auron-flink-planner`) survives. A single `<exclude>` is complete (no inner classes).
- **No `<relocations>`** — relocating `org.apache.flink.table.planner.*` would break the FQCN-identity
  shadow. (Mirrors why Spark's assembly excludes `org.apache.arrow.c.**`,
  `dev/mvn-build-helper/assembly/pom.xml:118-121`.)
- **Transformers**: keep the existing config and add `ApacheNoticeResourceTransformer` +
  `ApacheLicenseResourceTransformer` so the new `META-INF/{NOTICE,LICENSE}` merge into the jar. The existing
  `AppendingTransformer` for the legacy `TableFactory` SPI stays; the modern `Factory` SPI is already
  preserved by the current build (it is the established internal artifact), so no behavioral change is
  introduced to the runtime's table-factory wiring here.

### Licensing (ASF compliance) — net-new NOTICE/LICENSE in the assembly module

- `src/main/resources/META-INF/NOTICE` (net-new): Auron header + a statement that the jar bundles a
  **modified copy** of Apache Flink `flink-table-planner` (ALv2), bubbling up Flink's NOTICE attribution and
  an ALv2 §4(b) "modified by Apache Auron" note naming the substituted `StreamExecCalc`. Template:
  `flink-shaded-jackson`.
- `src/main/resources/META-INF/LICENSE`: ALv2 verbatim **plus** a block for `jts-core 1.19.0` (**EPL v2.0**,
  embedded inside the upstream planner jar under `org/locationtech/jts`). Other embedded third-party
  (Calcite 1.32.0, Avatica 1.22.0, Guava 31.1, commons-codec/io) are ALv2 → NOTICE list only.
- Root `NOTICE`/`LICENSE`: **no change** — embedding occurs only in the binary assembly jar; per-module
  `META-INF` is the correct home.

### Structural smoke test (build-time)

- New failsafe IT `AssemblyJarStructureIT` in `auron-flink-assembly`, running in the `verify` phase (after
  `package`/shade). Opens `target/auron-flink-assembly-*.jar` via `java.util.jar.JarFile` and asserts:
  1. exactly **one** entry `…/exec/stream/StreamExecCalc.class`;
  2. it is **Auron's** — byte-scan the entry for the UTF-8 constant
     `org/apache/auron/flink/runtime/operator/FlinkAuronCalcOperator` (present in Auron's bytecode,
     `StreamExecCalc.java:27,191-192`); no classloading, no Flink on the test classpath;
  3. a **non-shadowed** `StreamExec*` (e.g. another `StreamExec*` class) is present — proves the planner
     content was actually bundled, not just Auron's one class;
  4. `META-INF/NOTICE` contains both "Apache Auron" and "Apache Flink" — cheap guard against a forgotten
     transformer.
- Requires net-new `maven-failsafe-plugin` wiring in the assembly module (repo has none today).

### CI

- `.github/workflows/flink.yml`: build + **`verify`** the `auron-flink-assembly` module with `-Pflink-1.18`
  so the shade + structural IT execute. Today `flink.yml:45,60` builds/tests only `auron-flink-planner -am`
  with surefire `test`; the assembly is never built in CI, so the structural guarantee is currently
  unverified. (Minimal edit — extend the existing matrix/step to include the assembly module under `verify`.)

## Alignment with DESIGN Rev 4 / AIP (audit)

| Element | Status | Evidence / note |
|---|---|---|
| Shadow mechanism (Auron class in Flink package, FQCN identity) | **Aligned** | Unchanged from #2283 / Rev 4. No relocation. |
| Deployment procedure ("replace flink-table-planner.jar") | **Follow actual code → Rev 5 reconciliation** | Default Flink 1.18 has no `lib/flink-table-planner.jar`; corrected to the documented loader→non-loader swap. Flink 1.18 Table advanced-config docs. |
| A2 = shaded uber-jar replacement | **Aligned** | Rev 4 §"A2"; implemented as the enhanced `auron-flink-assembly` jar (Rev 2). |
| Packaging / runtime placement | **Follow actual code (Rev 2)** | Rev 4 implied a dedicated shaded jar; the in-repo assembly already bundles runtime + planner + Flink planner and is used internally. Reuse it (one jar) rather than add a module. |
| "A2 replaces A1" | **Aligned** | The same assembly jar now activates *structurally* (filter-guaranteed single `StreamExecCalc`) instead of by classpath-traversal ordering. |
| Per-Flink-version re-shade | **Aligned (deferred content)** | Only 1.18 baseline exists; filter/transformer structure is version-agnostic. 1.20/2.0 deferred per scope. |
| Observability machinery (activation log, strict mode) | **Aligned** | Becomes structurally guaranteed under A2; kept as-is (Rev 4 §"under A2 … can be kept or simplified"). No change this PR. |

## Prior Art Comparison

| Aspect | This design (A2) | Gluten `gluten-flink` (A1-explicit) | Spark `auron` assembly |
|---|---|---|---|
| Planner activation | Structural (one class in the assembly jar) | Classpath prepend via `config.sh` | n/a (Spark uses extension API) |
| Shades flink-table-planner | **Yes** (already bundled; this PR makes the override deterministic) | No (provided-scope thin jar) | n/a |
| Relocation of engine planner pkg | No (FQCN identity) | No | Excludes `org.apache.arrow.c.**` for C-ABI identity |
| Packaging | Single assembly jar (runtime + planner + Flink planner) | Separate thin jar + classpath edit | Single assembly |

## Dependencies

No new external dependencies. `maven-shade-plugin` 3.5.2 (already in `pluginManagement`), `maven-failsafe-plugin`
(net-new, standard Apache-parent-managed). `flink-table-planner_2.12:1.18.1` reached transitively through
`auron-flink-planner` (compile scope).

## Test Strategy

- **Build-time structural IT** (above) — the agreed CI smoke assertion.
- **Manual jar-swap procedure** (reviewhelper, not CI): on a real Flink 1.18 home, delete the loader jar,
  drop in the single `auron-flink-assembly` jar, run a Calc SQL, confirm native conversion via the existing
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

## Alternatives Considered

- **D1-a (rely on shade ordering)** — rejected (undocumented shade behavior; the incidental status quo this
  PR replaces).
- **New `auron-flink-planner-shaded` module + two jars (Rev 1's D2-A)** — superseded by Rev 2. Added a module
  and a second `lib/` jar for "planner-jar purity," but the assembly already bundles everything as one
  deployment jar; the extra module and second jar are unnecessary churn.
- **Fold runtime into a new shaded module (Rev 1's D2-B)** — same outcome as reusing the assembly but via a
  new module; reusing the existing assembly is strictly less churn.
- **A1-explicit (Gluten's `config.sh` prepend)** — improves determinism without shading but still asks the
  user to reason about classpath ordering; rejected as the long-term model in Rev 4.
