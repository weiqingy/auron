# Research — AURON #2291: Ship a shaded `auron-flink-planner` jar replacing the Flink planner

## Rev 2 note (2026-05-29)

`auron-flink-assembly` already bundles runtime + Auron planner + Flink planner content and is the single
deployment jar, so the design now **enhances that assembly jar in place** rather than adding a new module.
None of the research *facts* below change — in particular, the assembly already
contains both Auron's and stock `StreamExecCalc` (the duplicate the `<filters>` exclude resolves), and the
loader-swap deployment finding still applies. Only the *packaging home* moved. See `AURON-2291-DESIGN.md`
Rev 2.

---

## ⚠️ Headline finding (changes the deployment procedure — needs reconciliation)

Flink 1.18's **default distribution does NOT ship `flink-table-planner.jar` in `$FLINK_HOME/lib/`.**
It ships `flink-table-planner-loader-1.18.1.jar` in `lib/`, while the real planner
(`flink-table-planner_2.12-1.18.1.jar`) sits in `opt/` and is loaded through an isolating
`ComponentClassLoader`. Two consequences:

1. The issue text and `AURON-1853-DESIGN.md` Rev 4 wording — *"replace `$FLINK_HOME/lib/flink-table-planner.jar`"* — does not match reality. There is no such jar in `lib/` to replace.
2. A same-FQCN class in a sibling `lib/` jar is loaded by a **different** classloader than the
   loader-internal planner and would never shadow it. And Flink documents *"the two planners cannot
   co-exist in the classpath."*

**A2 is still viable — the procedure is refined, not killed.** The supported path is Flink's own
documented loader→non-loader swap:
> Remove `flink-table-planner-loader-1.18.1.jar` from `lib/`, then place `auron-flink-planner-shaded.jar`
> (= repackaged non-loader `flink-table-planner_2.12` content with Auron's `StreamExecCalc` substituted
> in) into `lib/`.

This aligns with the repo, which already depends on the **non-loader** `flink-table-planner_2.12` at
compile scope (`auron-flink-planner/pom.xml:130-134`). Source: Flink 1.18 Table "advanced config" docs
(loader in `/lib`, planner in `/opt`, "cannot co-exist"); loader internals confirmed via `javap` on the
cached loader jar showing `new ComponentClassLoader(...)`.

➡️ **Requires a DESIGN Rev 5 reconciliation** of the deployment wording (loader→opt swap, not
"replace flink-table-planner.jar"). Carries into the #1892 user docs and the smoke procedure.

## Proven Patterns

- **A1 overlay jar = `auron-flink-assembly`** (`auron-flink-extension/auron-flink-assembly/pom.xml:42-86`).
  A `maven-shade-plugin` uber jar bundling BOTH `auron-flink-runtime` and `auron-flink-planner`
  (`:29-40`). No `<finalName>`/`<relocations>` → produces default `auron-flink-assembly-${version}.jar`,
  replacing the main artifact. Dropped ahead of the planner on the classpath = A1.
- **`auron-flink-planner` depends on `flink-table-planner_2.12` at compile scope**
  (`auron-flink-planner/pom.xml:130-134`, no `<scope>`) — deliberately, so planner content can be
  bundled. This is the artifact A2 re-shades.
- **Shadow class** lives at `auron-flink-planner/src/main/java/org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.java` (PR #2283).
- **Native lib placement**: `libauron.so` ships as a resource of `auron-core` (`auron-core/pom.xml:89-98`),
  rides the `auron-core → auron-flink-runtime` dep chain, loaded via
  `getResourceAsStream(mapLibraryName("auron"))` (`FlinkAuronAdaptor.java:35-46`). Whatever jar carries
  the runtime must carry `libauron.so`. Both "runtime bundled in shaded jar" and "runtime as separate
  lib/ jar" are mechanically supported → **a design decision, not a build constraint.**
- **Spark shading prior art**: `dev/mvn-build-helper/assembly/pom.xml:62-150` relocates only
  *third-party* libs (arrow/protobuf/netty), and **excludes `org.apache.arrow.c.**`** because the C ABI
  needs original FQCNs (`:118-121`). Direct lesson for A2: **must NOT relocate
  `org.apache.flink.table.planner.*`** — the shadow mechanism is FQCN-identity based.
- **Gluten precedent**: Gluten does NOT shade flink-table-planner. `gluten-flink/planner/pom.xml` uses
  `provided` scope, no shade plugin, emits a thin shadow-only jar; deployment is pure A1-explicit
  (`gluten_lib/` + `config.sh` classpath prepend). So Auron's A2 has **no direct Flink prior art** — the
  shadow *technique* is shared, but the *shaded-replacement* packaging is novel; NOTICE precedent must
  come from `flink-shaded` / Spark.

## API Surface (build config — versions, coordinates, plugin options)

- Versions (root `pom.xml`): flink **1.18.1**, scala **2.12**, maven-shade-plugin **3.5.2** (`pom.xml:73`).
- Flink profile id is **`flink-1.18`** (NOT `flink`); build with `-Pflink-1.18`.
- Non-loader `flink-table-planner_2.12-1.18.1.jar`: ~51.8 MB, 11101 entries, contains
  `StreamExecCalc.class` directly, bundles Calcite (~4974 entries) — already a fat jar. Ships
  `META-INF/{NOTICE,LICENSE,DEPENDENCIES}` and `META-INF/services/org.apache.flink.table.factories.Factory`.
- **Class-override mechanism**: maven-shade-plugin 3.5.2 docs do **NOT** document an ordering-based
  winner for overlapping classes. Do not rely on dependency/artifactSet ordering. The deterministic,
  documented path is `<filters>` excluding `StreamExecCalc.class` from the Flink artifact so only Auron's
  copy survives the shade.
- **ASF-compliant transformers** (all first-party shade transformers):
  `ServicesResourceTransformer` (merge `META-INF/services`, incl. the modern `Factory` SPI **and**
  `AuronAdaptorProvider` — dropping the latter breaks native loading), `ApacheNoticeResourceTransformer`,
  `ApacheLicenseResourceTransformer`, `ManifestResourceTransformer`.
- **Verification**: a `JarFile` unit test asserting exactly one `StreamExecCalc.class` = Auron's bytes
  (matches the agreed "structural assertion in CI" scope; prior art `testShadowedClassReplacesFlinkClass`
  at `AURON-1853-DESIGN.md:580`). `BanDuplicateClasses` (extra-enforcer-rules) is an alternative but adds
  a dependency.
- **ASF NOTICE rules** (infra.apache.org/licensing-howto.html): bundling another Apache product →
  analyze its NOTICE, bubble up required portions; relocated copyright notices from *modified* source
  files must be preserved (ALv2 §4(b)); don't stamp Auron's ALv2 header onto modified Flink files; carry
  a "modified by Auron" statement. Lightest template to copy: `flink-shaded-jackson` `META-INF/NOTICE`.

## Blockers / Incompatibilities

- **flink-table-planner-loader** (headline above) — refines the deployment procedure and the
  issue/DESIGN wording. Not a true blocker for A2, but a blocker for the *as-written* "replace
  flink-table-planner.jar" phrasing.
- **CI never builds/tests the assembly today** — `.github/workflows/flink.yml:45` builds only
  `auron-flink-planner -am`. The A2 structural smoke assertion needs a new CI hook (or attach the test to
  the planner-shaded module so the existing reactor picks it up).
- **SPI transformer gap**: today's assembly merges only the legacy `TableFactory` SPI; the runtime's
  `Factory` and `AuronAdaptorProvider` SPIs are not in the transformer. A2's shade must merge the modern
  SPIs or native loading / table factories break.
- **Same-FQCN duplicate** (`StreamExecCalc`: stock vs Auron) inside one fat jar — shade ordering risk;
  the structural test must verify the surviving bytes are Auron's, not just "exactly one present".

## Unknowns / Risks

- **Runtime placement decision (the context.md open question)**: keep `auron-flink-runtime` as a separate
  `lib/` jar, or bundle it into the shaded planner jar? Mechanically both work. Gluten keeps runtime
  separate (suggestive, not authoritative). → **Design decision for Phase 3.** Leaning: separate runtime
  jar (cleaner separation; smaller blast radius; the shaded jar stays a pure planner replacement).
- **Exact bundled-artifact set for the NOTICE**: must enumerate via `mvn dependency:tree` on the
  shaded module to list embedded ALv2 components (Calcite, Janino, etc.) for accurate NOTICE attribution.
- **1.18.1 loader internals** were confirmed against a cached 1.16.1 loader + 1.18 docs (1.18.1 loader
  wasn't in local m2). Mechanism is unchanged across versions, but confirm with `unzip -l` once the build
  downloads the real 1.18.1 loader.
- **Artifact size**: ~30-52 MB shaded jar. Acceptable (flink-dist is ~120 MB) but note for CI artifact
  handling.

## Dead Ends

- **Naive "drop second planner jar in `lib/` alongside the loader"** — fails: different classloader can't
  shadow loader internals, and Flink forbids two planners co-existing. (Evidence: Flink 1.18 docs +
  `ComponentClassLoader` in the loader jar.)
- **Relying on shade dependency/artifactSet ordering to pick Auron's `StreamExecCalc`** — not documented
  behavior in shade 3.5.2; must use explicit `<filters>` exclusion instead.
- **Relocating `org.apache.flink.table.planner.*`** — would break the FQCN-identity shadow mechanism.

## Raw Research Files
- [In-repo build/packaging](research-internal.md)
- [maven-shade + flink-table-planner mechanics](research-framework.md)
- [Gluten + ASF NOTICE prior art](research-external.md)
