# Investigation (file mapping + module wiring) — AURON #2291

Scope: precise CREATE/MODIFY map for the A2 shaded planner module. NO CODE. Every claim
anchored to a verified `path:line`. Q2 (runtime placement) is undecided → both options
mapped where they diverge.

---

## Files to Create

| Path | Purpose | Key content notes |
|---|---|---|
| `auron-flink-extension/auron-flink-planner-shaded/pom.xml` | The new shaded module's POM. Produces the fat jar = `flink-table-planner_2.12-1.18.1` content with Auron's same-FQCN `StreamExecCalc` substituted in. | `<parent>` = `auron-flink-extension` (mirror `auron-flink-assembly/pom.xml:20-24`, NOT `auron-parent`). `<artifactId>auron-flink-planner-shaded</artifactId>`, **`<packaging>jar</packaging>`** (default; shade replaces the main artifact like the assembly does — confirmed `auron-flink-assembly/pom.xml` has no `<packaging>` so it defaults to `jar`). Deps: `auron-flink-planner` (compile) — Q2(a); **+ `auron-flink-runtime`** only under Q2(b) "fold runtime in". `maven-shade-plugin` `${maven.plugin.shade.version}` (=3.5.2, `pom.xml:73`) bound to `package`/`shade`. Critical shade config: `<filters>` excluding `StreamExecCalc.class` from the `org.apache.flink:flink-table-planner_2.12` artifact so only Auron's copy survives (research.md L62-64, U1 `research-internal.md:208-220`); NO `<relocations>` of `org.apache.flink.table.planner.*` (B2 `research-internal.md:198-204`); transformers: `ServicesResourceTransformer` + `ApacheNoticeResourceTransformer` + `ApacheLicenseResourceTransformer` + `ManifestResourceTransformer` (research.md L65-68); signature-strip filter `META-INF/*.SF\|DSA\|RSA` (mirror `auron-flink-assembly/pom.xml:65-73`). ALv2 header: **copy lines 2-17 verbatim from `auron-flink-assembly/pom.xml:2-17`** (the `<!-- ~ Licensed ... -->` block). |
| `auron-flink-extension/auron-flink-planner-shaded/src/main/resources/META-INF/NOTICE` | Source NOTICE bubbling up embedded **modified** Flink planner attribution + bundled ALv2 components (Calcite, Janino) per ASF licensing-howto (research.md L73-76). | **Net-new — no in-repo precedent** (`research-internal.md:246-251`). The Apache parent v35 (`pom.xml:21-25`) auto-generates a generic per-module `META-INF/NOTICE` via `maven-remote-resources-plugin` (verified: `auron-flink-planner/target/classes/META-INF/NOTICE` = "Apache Auron Flink Planner 1.18.1 / Copyright 2025 ... developed at ASF" — boilerplate only). For A2 the shade's `ApacheNoticeResourceTransformer` MERGES the bundled Flink/Calcite NOTICEs with this one; a hand-authored source `META-INF/NOTICE` here is the place to add the "modified by Auron" §4(b) statement. Must live under `src/main/resources/META-INF/` so it is a classpath resource the transformer picks up. |
| `auron-flink-extension/auron-flink-planner-shaded/src/main/resources/META-INF/LICENSE` | (Optional, only if bundled content needs license-text additions beyond ALv2 root.) | Apache parent also auto-generates `META-INF/LICENSE` (verified in target dirs). Likely NOT needed as a source file unless a bundled component carries a non-ALv2 license requiring text inclusion — enumerate via `mvn dependency:tree` (research.md L98-99) before deciding. **Flag for design**, do not assume. |
| `auron-flink-extension/auron-flink-planner-shaded/src/test/java/.../StreamExecCalcShadeStructuralTest.java` (exact pkg/name TBD by design) | The "exactly one `StreamExecCalc` = Auron's" structural assertion (context.md decision #2). | See "Structural test location" below — this is the **JarFile** test reading `target/*.jar`. Must run in `verify` phase, AFTER `package` (shade). Asserts: jar entry `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` appears exactly once AND its bytes are Auron's (e.g. contains the `org.apache.auron` import / activation field) not stock Flink's (U1 `research-internal.md:208-220`). NOTE: this is a NEW test pattern — the DESIGN:580 `testShadowedClassReplacesFlinkClass` was **never implemented** (see Verified Assumptions A4). |

---

## Files to Modify

| Path:line | Current content (quoted) | What changes | Why |
|---|---|---|---|
| `auron-flink-extension/pom.xml:32-36` | `<modules>`<br>`  <module>auron-flink-planner</module>`<br>`  <module>auron-flink-runtime</module>`<br>`  <module>auron-flink-assembly</module>`<br>`</modules>` | Add `<module>auron-flink-planner-shaded</module>`. **Order**: it must come AFTER `auron-flink-planner` (and after `auron-flink-runtime` if Q2(b) folds runtime in) because shade bundles those artifacts. Reactor order follows declaration order unless overridden; place it last (after `auron-flink-assembly`, or replacing it — see A2-replaces-A1 audit). | New module must join the Flink aggregator reactor (`research-internal.md:38-43`). Dependency on `auron-flink-planner` forces correct build order regardless, but explicit last-position keeps it readable. |
| `auron-flink-extension/auron-flink-assembly/pom.xml` (whole module) | The A1 overlay uber-jar (`pom.xml:42-86` shade config). | Under "A2 replaces A1": **delete, repoint, or leave** — see A2-replaces-A1 audit table for all three options + breakage. | context.md decision #4: "A2 replaces A1 — the A1 overlay build is removed/repointed." This is a decision for design; investigation enumerates the options. |
| `.github/workflows/flink.yml:45` | `module: [ "auron-flink-extension/auron-flink-planner" ]` | Add the new module to the matrix: `module: [ "auron-flink-extension/auron-flink-planner", "auron-flink-extension/auron-flink-planner-shaded" ]` — OR (minimal) change the single entry's build to also reach the shaded module via `-pl`. | CI today builds ONLY the planner with `-am` (`flink.yml:60`), which pulls upstream deps but NOT the downstream shaded module (`research-internal.md:189-196`, B1). The structural smoke assertion (decision #2) is never exercised unless CI builds the shaded module. |
| `.github/workflows/flink.yml:60` | `run: ./build/mvn -B test -pl ${{ matrix.module }} -am -Pscala-${{ matrix.scalaver }} -Pflink-${{ matrix.flinkver }} -P${{ matrix.sparkver }} -Prelease` | If adding the module to the matrix (option above), this line is reused as-is per matrix entry — BUT the shaded module's structural test runs in `verify`, so the goal must be `verify` not `test` for that entry, OR bind the JarFile test to run in the `integration-test`/`verify` phase via `maven-failsafe-plugin`. Simplest: a second matrix entry running `./build/mvn -B verify -pl auron-flink-extension/auron-flink-planner-shaded -am ...`. | `mvn test` stops before `package`; the shade (and therefore the jar to assert on) only exists after `package`. The test MUST run at/after `verify`. (research.md L83-85; B1.) |

**Not modified (verified absent):** `auron-build.sh` has NO `auron-flink-assembly` or shaded-jar reference — it only emits `-Pflink-$FLINK_VER` (`auron-build.sh:466-467`) and never names a Flink module or assembly jar (grep returned only flink-version plumbing at `:39,62,331-340,466-467,509,527`). So **no `auron-build.sh` edit is required** for the module to build under `-Pflink-1.18`; it builds via the normal reactor. (If a release wants the shaded jar copied to a dist staging dir, that is a separate, currently-nonexistent step — flag for design, not required for this PR.)

---

## A2-replaces-A1 reference audit

Exhaustive grep of `auron-flink-assembly` across `*.xml/*.yml/*.yaml/*.sh/*.md` (excluding `docs/PR-2291`, `docs/review-logs`). **Zero references in build scripts or CI** — the assembly is wired ONLY through the parent `<modules>` list and its own POM. Everything else is documentation.

| Reference | file:line | Kind | Breakage if assembly is removed |
|---|---|---|---|
| `<module>auron-flink-assembly</module>` | `auron-flink-extension/pom.xml:35` | **Live build wiring** | Removing the line drops the module from the reactor — clean (no breakage) **if** the module dir is also deleted/repointed. |
| `<artifactId>auron-flink-assembly</artifactId>` | `auron-flink-extension/auron-flink-assembly/pom.xml:26` | **The module itself** | N/A — this is the artifact being retired. |
| roadmap doc mention | `docs/flink-phase1-roadmap.md:185` | Doc (#1852 module-skeleton history) | None at build time; doc would be stale. Update optional. |
| `auron-flink-assembly`'s shade orders Auron ahead of Flink | `docs/reviewhelper/AURON-1853/02-shadow-stream-exec-calc.md:5` | Doc (merged reviewhelper) | None at build time; historical reviewhelper for a merged PR. Leave. |
| classpath-ordering / out-of-scope test notes | `docs/PR-AURON-1853/AURON-1853-PLAN.md:177,203`; `AURON-1853-starter-prompt.md:86`; `investigate-deps.md:16,281`; `AURON-1853-SPEC.md:22`; `AURON-1853-DESIGN.md:177,209,596` | Docs (merged #1853 artifacts) | None at build time; local-only RIPER artifacts. Leave. |
| starter-prompt / context "read assembly first" | `docs/PR-AURON-2291/AURON-2291-starter-prompt.md:49,58,70,171,180`; `docs/PR-AURON-2291/context.md:52` | This issue's own scaffolding docs | None — guidance for this very investigation. |

**Critical CI fact:** `.github/workflows/flink.yml` does NOT reference `auron-flink-assembly` at all (verified by grep + full read). The A1 overlay jar is **never built or tested in CI today** (`research-internal.md:189-196`). So retiring it has **zero CI breakage**.

### Retirement options (present, do not decide)

- **(a) Delete the assembly module entirely.** Edit `auron-flink-extension/pom.xml:35` (remove the `<module>` line) + delete `auron-flink-extension/auron-flink-assembly/` dir.
  - **Breaks:** nothing in build/CI (no live references outside the `<module>` line). Any external consumer who scripts `mvn ... -pl auron-flink-extension/auron-flink-assembly` or expects `auron-flink-assembly-${ver}.jar` would break — but no such reference exists in-repo. Docs (`flink-phase1-roadmap.md:185`) go stale.
  - **Cleanest under context.md decision #4** ("A2 replaces A1; pre-GA, no installed users → no migration cost").

- **(b) Keep the module dir but repoint it** (e.g. make `auron-flink-assembly` depend on / re-export the shaded jar, or convert it to the runtime-only `auron-flink-runtime` lib jar under Q2(b)).
  - **Breaks:** nothing immediately; risk of two overlapping jars (assembly + planner-shaded) both bundling planner content → the duplicate-`StreamExecCalc` ambiguity moves to deployment. Adds maintenance of a second shade config.
  - Only sensible if Q2 picks "runtime stays a separate lib/ jar" AND you want to reuse the assembly module as that runtime jar.

- **(c) Leave assembly untouched, just add the new module.**
  - **Breaks:** nothing, but **violates context.md decision #4** (A2 is meant to *replace* A1, not coexist). Leaves a dead A1 artifact that is never CI-tested and now diverges from the A2 shaded jar — exactly the drift the decision wants to avoid.
  - Lowest-effort but contradicts the approved scope.

**Recommendation surface for design (not a decision):** (a) aligns with the approved "A2 replaces A1" decision and has zero in-repo breakage; (b) is the path only if Q2 reuses the assembly as the separate runtime jar. Lock at the design gate alongside Q2.

---

## Structural test location

**Requirement:** the test must read the built shaded `target/*.jar` and assert exactly one `StreamExecCalc.class` = Auron's. The jar exists only after `package` (shade), so the test must run in **`integration-test`/`verify`**, never `test` (surefire runs before `package`).

**The DESIGN:580 test does NOT exist as a JarFile test.** Verified:
- `AURON-1853-DESIGN.md:580` describes `testShadowedClassReplacesFlinkClass` using `getProtectionDomain().getCodeSource().getLocation()` — but it was **planned, then dropped**: `AURON-1853-PLAN.md:177` lists "Classpath ordering regression test" under *Tests NOT Writing* ("would require manipulating shade configuration, out of scope"), and `AURON-1853-DESIGN.md:596` repeats this. No such method exists in the codebase (grep for `getProtectionDomain`/`CodeSource`/`testShadowedClassReplaces` in `auron-flink-extension/` returns nothing under `src`).
- What DOES exist: `StreamExecCalcTest.testShadowedClassResolvesViaClasspath` (`auron-flink-extension/auron-flink-planner/src/test/.../StreamExecCalcTest.java:377-380`), asserting only:
  ```java
  Class<?> resolved = Class.forName("org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc");
  assertSame(StreamExecCalc.class, resolved);
  ```
  This is an in-module classpath check (the planner module has only Auron's copy on its test classpath), **NOT** a fat-jar structural assertion. It does not prove the shade picks Auron's bytes when both copies are bundled. **A2 needs a genuinely new test.**

**Where to put the new JarFile test (options, do not decide):**

- **Option 1 — failsafe IT in the shaded module.** Place `StreamExecCalcShadeStructuralTest.java` (or `*IT.java`) in `auron-flink-planner-shaded/src/test/java/...`, add `maven-failsafe-plugin` bound to `integration-test`/`verify`. The test opens `new JarFile(...)` on the module's own `target/auron-flink-planner-shaded-${version}.jar`. **No in-repo failsafe precedent exists** (the whole repo uses surefire `test` only; CI runs `mvn test`). Pro: lives with the module; Con: requires wiring failsafe + locating the jar by glob (use `${project.build.directory}` system property).
- **Option 2 — maven-invoker-plugin.** Heavier; runs a sub-build then asserts. Overkill for a single JarFile check. No in-repo precedent. Not recommended.
- **Option 3 — verify-phase exec/groovy assertion.** A `maven-shade`-companion check via `maven-antrun`/`groovy-maven-plugin` reading the jar. No in-repo precedent; harder to express the "bytes are Auron's" check than a Java test. Not recommended.

**How existing modules run post-package checks:** they **don't** — verified there is no failsafe-plugin, no maven-invoker, no verify-phase test anywhere in the Flink modules; the only verify-phase plugin inherited is `apache-rat-plugin` (`pom.xml:708-728`, bound to `verify`) which checks *source headers*, not bundled jar content. So Option 1 (failsafe IT) is net-new infrastructure but the most idiomatic Java path; flag the failsafe-plugin addition for design.

---

## Verified Assumptions

- **A1.** `auron-flink-assembly` is `packaging=jar` by default (no `<packaging>` element; `auron-flink-assembly/pom.xml:18-27`), shade replaces the main artifact → default name `auron-flink-assembly-${project.version}.jar`. The new shaded module mirrors this (jar packaging, shade, no `<finalName>` unless design wants a friendlier dist name). (`research-internal.md:9-16`.)
- **A2.** `auron-flink-planner` depends on `flink-table-planner_2.12` at **compile** scope (`auron-flink-planner/pom.xml:130-134`, no `<scope>`), so depending on `auron-flink-planner` transitively pulls the stock planner content into the shade — exactly what A2 re-packages. (`research-internal.md:45-57`.)
- **A3.** Shade plugin version resolves via `${maven.plugin.shade.version}` = `3.5.2` (`pom.xml:73`); it is **NOT** in `<pluginManagement>` (grep returned no `maven-shade-plugin` line in root `pom.xml`), so the new module declares the full `<plugin>` block itself (same as `auron-flink-assembly/pom.xml:44-47`).
- **A4.** The DESIGN:580 `testShadowedClassReplacesFlinkClass` JarFile/CodeSource test was **never implemented** (dropped per `AURON-1853-PLAN.md:177`); the only shipped classpath test is the in-module `assertSame` at `StreamExecCalcTest.java:377-380`. A2's structural assertion is net-new.
- **A5.** Per-module `META-INF/NOTICE` + `META-INF/LICENSE` in `target/` are **auto-generated** by the Apache parent v35 (`pom.xml:21-25`) `maven-remote-resources-plugin` (`org.apache:apache-jar-resource-bundle`), NOT source files — verified: no `src/main/resources/META-INF/{NOTICE,LICENSE}` exists anywhere in the repo, and `auron-flink-planner/target/classes/META-INF/NOTICE` contains generic ASF boilerplate. A hand-authored source NOTICE is therefore net-new for A2's modified-Flink attribution.
- **A6.** `auron-build.sh` contains zero Flink-module/assembly names; only `-Pflink-$FLINK_VER` plumbing (`:466-467`). No build-script edit required for the module to compile.
- **A7.** CI `flink.yml` builds only `auron-flink-planner -am` with `mvn test` (`:45,60`); the assembly/shaded jar is never built or tested today (B1).

## Compatibility Risks

- **R1 — duplicate `StreamExecCalc` in the fat jar (U1).** `flink-table-planner_2.12:1.18.1` carries the stock `StreamExecCalc.class`; `auron-flink-planner` carries Auron's. Both arrive as *dependency* classes (the shaded module owns neither), so maven-shade's "project classes win" rule does NOT apply — resolution is dependency-order-dependent and may emit a duplicate warning. The `<filters>` exclusion of `StreamExecCalc.class` from the Flink artifact is the deterministic fix; the structural test must verify the *surviving bytes are Auron's*, not just "one entry". (`research-internal.md:208-220`.)
- **R2 — SPI merge (U2).** If Q2(b) folds the runtime in, the shade MUST merge `META-INF/services/org.apache.auron.jni.AuronAdaptorProvider` and `.../org.apache.flink.table.factories.Factory` (both real source files: `auron-flink-runtime/src/main/resources/META-INF/services/...`, verified present) via `ServicesResourceTransformer`. The legacy assembly only merged `TableFactory` (`auron-flink-assembly/pom.xml:60-64`) — dropping `AuronAdaptorProvider` breaks native loading. Under Q2(a) (runtime separate) this risk lives in the runtime jar, not the planner-shaded jar.
- **R3 — relocation must NOT touch `org.apache.flink.table.planner.*` (B2).** The shadow is FQCN-identity based; relocating it kills activation. Spark's relocation precedent (`dev/mvn-build-helper/assembly/pom.xml:106-143`) applies only to third-party arrow/protobuf/netty, and even there excludes `org.apache.arrow.c.**` — relevant only if A2 bundles arrow + native `.so` under Q2(b).
- **R4 — failsafe is net-new.** Adding `maven-failsafe-plugin` to run the structural IT in `verify` is infrastructure the repo has never used (only surefire `test`). CI must call `mvn verify` (not `test`) for this module or the assertion never runs.

## Edge Cases

- **E1 — jar location in the structural test.** Glob `target/*.jar` is fragile if shade also leaves an `original-*.jar`. Assert on the exact `${project.build.finalName}.jar` via a system property passed to the test, or filter out `original-`/`-sources`/`-javadoc` jars.
- **E2 — `original-*.jar` from shade.** maven-shade writes `original-<artifact>.jar` alongside the shaded one; the structural test (and any `verify` glob) must ignore it.
- **E3 — `<finalName>` vs default.** If design wants the deployed jar named e.g. `flink-table-planner_2.12-1.18.1.jar` (to literally drop into `lib/` per the loader-swap procedure, context.md Q1), a `<finalName>` is needed — but that collides with stock naming. Decide at design; default `auron-flink-planner-shaded-${version}.jar` + a documented rename in the deploy procedure is the lower-risk path.
- **E4 — module order vs assembly removal.** If retirement option (a) deletes `auron-flink-assembly`, ensure the `<module>` line removal and the new `<module>` addition land in the same edit so the reactor stays consistent.

## Constraints

- Java only for Flink modules (CLAUDE.md / context.md L47).
- Flink profile id is `flink-1.18` (NOT bare `flink`); the new CI line must use `-Pflink-1.18` (`research-internal.md:238-244`, U4).
- ALv2 header on every new file: copy the `<!-- ~ Licensed ... -->` block from `auron-flink-assembly/pom.xml:2-17` for POMs; standard Java header for `.java`. RAT (`pom.xml:708-728`) enforces `numUnapprovedLicenses=0` in `verify`.
- NOTICE must reflect embedded **modified** Flink content (ASF §4(b); context.md L49-50) — net-new source file, no in-repo precedent (A5).
- Must NOT relocate `org.apache.flink.table.planner.*` (R3).
- Structural assertion runs in `verify`/`integration-test`, never `test` (timing constraint).
- PR title `[AURON #2291] ...`, squash-merge, user sole author.

## Q2 (runtime placement) — file-map divergence summary

| Concern | Q2(a) runtime = separate `lib/` jar (leaning) | Q2(b) runtime folded into shaded jar |
|---|---|---|
| Shaded module deps | `auron-flink-planner` only | `auron-flink-planner` + `auron-flink-runtime` |
| Native `.so` in shaded jar | No (rides the separate runtime/assembly jar via `auron-core`, `auron-core/pom.xml:89-98`) | Yes (must bundle `auron-core` → `libauron.so`) → needs OS-classifier naming like Spark (`dev/mvn-build-helper/assembly/pom.xml:151-180`) |
| SPI transformers needed | Planner SPIs only | + `AuronAdaptorProvider` + `Factory` merge (R2) |
| Fate of `auron-flink-assembly` | Could be repointed as the runtime jar (retirement option b) | Fully superseded → delete (retirement option a) |
| Arrow/relocation concern | None | Possible arrow bundling → relocation with `org.apache.arrow.c.**` excluded (R3) |

Both are mechanically supported by the existing dep graph (`research-internal.md:122-130`). Decision is the design gate's, with evidence above.
