# Investigation — AURON #2291: Shaded `auron-flink-planner` jar replacing the Flink planner

## Rev 2 reconciliation (2026-05-29 — reviewer feedback)

Reviewer (@Tartarus0zm) redirected the design to **reuse the existing `auron-flink-assembly`** (it already
bundles runtime + Auron planner + Flink planner content and is used internally as one jar) instead of
creating a new `auron-flink-planner-shaded` module. **Net effect on this investigation:**
- "Files to Create" — **drop** the new-module pom; the `<filters>` exclude, the net-new `META-INF/{NOTICE,
  LICENSE}`, and the structural IT all move **into `auron-flink-assembly`** (`src/main/resources` + `src/test`).
  The structural IT targets `target/auron-flink-assembly-*.jar`.
- "Files to Modify" — **no `<modules>` change** and **no assembly deletion**; instead edit the assembly's
  existing shade config (add the class-exclude filter + NOTICE/LICENSE transformers + failsafe wiring).
- The "A2-replaces-A1 reference audit" (delete-assembly options) is **moot** — the assembly is kept and
  enhanced, not retired. Its "never built in CI" finding still drives the CI change (build + `verify` it).
- Q2 (runtime placement) is **resolved**: one jar (the assembly). The two-jar analysis below is superseded.

Everything else (filter completeness, distinguishing signal, bundled-deps list, failsafe-in-`verify`, EPL
`jts`) carries forward unchanged. See `AURON-2291-DESIGN.md` Rev 2 for the consolidated decision.

---

## Files to Create

| Path | Purpose | Key content notes |
|---|---|---|
| `auron-flink-extension/auron-flink-planner-shaded/pom.xml` | New module producing the shaded jar | `<parent>` = `auron-flink-extension`; `packaging=jar` (default); ALv2 header copied verbatim from `auron-flink-assembly/pom.xml:2-17`; `maven-shade-plugin` `${maven.plugin.shade.version}` (=3.5.2) declared per-module; load-bearing config = `<filters>` exclude of `StreamExecCalc.class` from the Flink artifact; **NO relocation** of `org.apache.flink.table.planner.*` |
| `auron-flink-extension/auron-flink-planner-shaded/src/main/resources/META-INF/NOTICE` | ASF attribution for embedded modified Flink content | Net-new (no `src/.../META-INF/NOTICE` exists anywhere in repo today); follows `flink-shaded-jackson` template; draft in `investigate-notice-test.md` |
| `auron-flink-extension/auron-flink-planner-shaded/src/main/resources/META-INF/LICENSE` | Bundle LICENSE | ALv2 verbatim; product+version list lives in NOTICE not LICENSE (ALv2-only bundle) |
| `auron-flink-extension/auron-flink-planner-shaded/src/test/java/.../ShadedPlannerJarStructureIT.java` | Build-time structural smoke assertion | Failsafe IT (runs in `verify`, after `package`/shade); opens `target/auron-flink-planner-shaded-*.jar` via `JarFile` |

## Files to Modify

| Path:line | Current | Change | Why |
|---|---|---|---|
| `auron-flink-extension/pom.xml:32-36` | `<modules>` = planner, runtime, assembly | Add `<module>auron-flink-planner-shaded</module>` (last → builds after planner); remove `auron-flink-assembly` if Q-retire = delete | Wire new module; retire A1 per "A2 replaces A1" |
| `.github/workflows/flink.yml:45,60` | builds `auron-flink-planner -am`, runs `mvn test -pl ... -am` | Add shaded module to matrix; its entry runs **`mvn verify`** (not `test`) with `-Pflink-1.18` so package-phase shade + structural IT execute | Assembly/shaded never built in CI today; structural test only runs post-package |
| `auron-flink-extension/auron-flink-assembly/` (whole module) | A1 overlay jar (bundles runtime+planner) | **Delete** (recommended) — see retirement audit | "A2 replaces A1"; zero CI/build breakage |

`auron-build.sh` — **no edit needed**; it only emits `-Pflink-$FLINK_VER` and never names a Flink module.

## A2-replaces-A1 reference audit (zero breakage)

`auron-flink-assembly` is referenced **only** by:
- its own `<module>` line `auron-flink-extension/pom.xml:35`
- its own `pom.xml` + flattened pom
- docs (DESIGN/research artifacts)

It appears **nowhere** in `auron-build.sh` or `.github/workflows/flink.yml`, and is **never built or tested in CI today**. Retirement options:
- **(a) Delete the module entirely** — cleanest, aligns with decision #4, zero breakage. **Recommended.**
- (b) Repoint as the runtime-only jar — only relevant if Q2 keeps runtime separate AND we want a runtime distribution artifact (the `auron-flink-runtime` module's own jar may already suffice).
- (c) Leave it — contradicts approved "A2 replaces A1" scope.

## Verified Assumptions

- `auron-flink-planner` depends on `flink-table-planner_2.12:1.18.1` at **compile scope** (`auron-flink-planner/pom.xml:130-134`, no `<scope>`) — so the shaded module needs only depend on `auron-flink-planner` to transitively get both Auron's override classes and the full planner content.
- Stock Flink `StreamExecCalc` has **exactly one** `.class` entry, **no inner classes** (`unzip -l flink-table-planner_2.12-1.18.1.jar` → single 4185 B entry, no `StreamExecCalc$1.class`). Auron's source defines no nested/anonymous classes or lambdas. → **A single `<exclude>`** of `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` is complete.
- All shade options/transformers needed are first-party in 3.5.2 (`ServicesResourceTransformer`, `ApacheNoticeResourceTransformer`, `ApacheLicenseResourceTransformer`, `ManifestResourceTransformer`); `<filter><artifact>` glob shape proven in-repo at `auron-flink-assembly/pom.xml:65-73`. Nothing hallucinated.
- **Distinguishing signal** for "is it Auron's StreamExecCalc": the UTF-8 constant-pool string `org/apache/auron/flink/runtime/operator/FlinkAuronCalcOperator` in the class bytes (`StreamExecCalc.java:27,191-192`). Detected via byte `indexOf` on the jar entry — no classloading, no Flink on test classpath.
- Per-module `target/META-INF/{NOTICE,LICENSE}` are **auto-generated** by Apache parent v35's `maven-remote-resources-plugin` — NOT source. The modified-Flink NOTICE is net-new source.
- The planned `testShadowedClassReplacesFlinkClass` (`AURON-1853-DESIGN.md:580`) **was dropped** (`AURON-1853-PLAN.md:177`); only `StreamExecCalcTest.testShadowedClassResolvesViaClasspath:377-380` shipped (a simple in-module `Class.forName == StreamExecCalc.class`). The A2 JarFile test is net-new and complementary.

## Bundled ALv2 components (for NOTICE)

From inspecting the planner jar's embedded class bytes + its own `META-INF/NOTICE` (authoritative, not `DEPENDENCIES`): guava 31.1-jre, failureaccess 1.0.1, calcite-core/linq4j 1.32.0, avatica-core 1.22.0, commons-codec 1.15, commons-io 2.11.0 (ALv2); checker-qual 3.12.0 (MIT); **jts-core 1.19.0 (EPL v2.0 — triggers a LICENSE block, non-ALv2)**. Guava is relocated under `org/apache/flink/calcite/shaded/...` inside the jar.

## Compatibility Risks

- **CI uses surefire `test` only** — no failsafe/invoker/verify-phase test anywhere in repo. The structural IT needs **net-new `maven-failsafe-plugin`** wiring + the CI entry must call `mvn verify`.
- **SPI gotcha**: existing assembly's `AppendingTransformer` targets the **legacy** `TableFactory` SPI; the planner jar ships the **modern** `Factory` SPI (`META-INF/services/org.apache.flink.table.factories.Factory`). Use `ServicesResourceTransformer` to preserve it.
- **jts-core is EPL v2.0** (not ALv2) — already bundled inside the upstream planner jar; requires a LICENSE block, not just NOTICE. Confirm it's actually embedded (it is, per `org/locationtech/jts` bytes).
- **Flink NOTICE year**: the bubbled-up Flink copyright end-year should match the real 1.18.1 jar (~2014-2024), not 2026.

## Edge Cases

- Future Wave-1 shadows (#1860/#1861/#1864) need **per-FQCN excludes** + an inner-class re-audit (some stock `StreamExec*` DO carry `$N` classes, unlike `StreamExecCalc`). Out of scope here but the filter structure should be obviously extensible.
- Structural test should also assert a **non-shadowed** `StreamExec*` IS present (proves the planner content was actually bundled, not just Auron's one class) and that `META-INF/NOTICE` contains both "Apache Auron" and "Apache Flink" (cheap guard against a forgotten transformer).

## Constraints

- maven-shade-plugin **3.5.2**, Flink **1.18.1**, Scala **2.12**, profile id **`flink-1.18`** (not bare `flink`).
- Must NOT relocate `org.apache.flink.table.planner.*` (FQCN-identity shadow). Lesson mirrors Spark assembly excluding `org.apache.arrow.c.**` (`dev/mvn-build-helper/assembly/pom.xml:118-121`).
- Java only; ALv2 headers on all new files; Javadoc on public APIs; checkstyle 0 violations.

## Q2 (runtime placement) — both options mapped, NOT decided (design gate)

| Aspect | Q2-A: separate `auron-flink-runtime` lib/ jar (leaning) | Q2-B: runtime folded into shaded jar |
|---|---|---|
| Shaded jar deps | `auron-flink-planner` only | + `auron-flink-runtime` |
| `libauron.so` | stays in runtime jar | must bundle → OS-classifier naming complexity |
| SPI merge | `Factory` passes through, no merge needed | `ServicesResourceTransformer` **mandatory** (runtime's `Factory` SPI → Kafka factory would clobber Flink's 3 `Default*Factory` entries) |
| Arrow relocation | n/a | may need to mirror Spark's `org.apache.arrow.c.**` handling |
| Shaded jar character | pure planner replacement | mixed planner + execution + native binary |
| User install | 2 jars in `lib/` | 1 jar in `lib/` |

`ServicesResourceTransformer` is included defensively in the deps sketch (required under B, harmless under A).

## Raw Investigation Files
- [File mapping + module wiring](investigate-files.md)
- [Shade config + dependencies](investigate-deps.md)
- [NOTICE/LICENSE + verification test](investigate-notice-test.md)
