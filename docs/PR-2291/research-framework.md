# Research (maven-shade + flink-table-planner) — AURON #2291

Scope: maven-shade-plugin class-override mechanics + Flink artifact facts for producing
`auron-flink-planner-shaded.jar` that replaces `$FLINK_HOME/lib/`'s table-planner jar.

All claims below are evidence-backed. Verified facts use the local m2 repo
(`~/.m2/repository/org/apache/flink`), the repo poms, disassembled Flink classes, and the
official Flink 1.18 docs. Plugin behavior is cited from the maven-shade-plugin 3.5.2 docs.

---

## Proven Patterns

### Confirmed coordinates and versions (from the repo)
- `flink.version` = **1.18.1** — `pom.xml:1355`.
- `scala.binary.version` = **2.12** — `auron-flink-planner/pom.xml:33`.
- `maven.plugin.shade.version` = **3.5.2** — `pom.xml:73`.
- `auron-flink-planner` depends on the **non-loader** planner artifact at **compile** scope:
  `org.apache.flink:flink-table-planner_2.12:1.18.1` — `auron-flink-planner/pom.xml:130-134`.
  (No dependency on `flink-table-planner-loader` anywhere in the repo.)
- The shadow class is `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc`,
  source at
  `auron-flink-planner/src/main/java/org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.java`.
- An existing shade config already lives in `auron-flink-assembly/pom.xml:42-85` (the A1 overlay
  uber-jar). It bundles `auron-flink-runtime` + `auron-flink-planner`, excludes hadoop/slf4j/cep,
  and uses one `AppendingTransformer` for `META-INF/services/org.apache.flink.table.factories.TableFactory`.
  It does NOT currently shade in `flink-table-planner` content — Flink deps in the planner pom are
  mostly `provided`, except the planner artifact itself which is compile-scope.

### What the non-loader planner jar contains (verified by unzip)
`flink-table-planner_2.12-1.18.1.jar` (the `/opt` jar):
- Size **~51.8 MB**, **11101 entries**.
- Contains `StreamExecCalc.class` directly at
  `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` (4185 bytes).
- **Self-contained**: bundles its transitive closure relocated/embedded, including calcite
  (4974 entries under `org/apache/calcite/...`, e.g. `RelBuilder.class`). It is a fat jar already.
- Ships ASF metadata: `META-INF/NOTICE` (2635 B), `META-INF/LICENSE` (11358 B),
  `META-INF/DEPENDENCIES` (16138 B), `META-INF/LICENSE.txt`.
- Ships SPI: `META-INF/services/org.apache.flink.table.factories.Factory` (972 B).
  (Note: this is the modern `Factory` SPI, not the legacy `TableFactory` the assembly currently
  appends — see Unknowns.)

This is exactly the jar A2 wants to re-package with Auron's `StreamExecCalc` substituted.

---

## API Surface (plugin config options, transformers, artifact coordinates)

### maven-shade-plugin 3.5.2 — class-override semantics
- The **documented** mechanism to make one artifact's copy of a class win is `<filters>`: for each
  filter you name an `<artifact>` and a set of `<includes>`/`<excludes>` file patterns; includes are
  applied before excludes. Source: shade-mojo.html (`filters` parameter). Quote: *"Allows you to
  specify an artifact ... and a set of include/exclude file patterns for filtering which contents of
  the archive are added to the shaded jar."*
- **There is NO documented "which duplicate wins" / dependency-ordering rule** for overlapping
  classes. The shade-mojo docs do not state that `<dependencies>` order or `<artifactSet>` order
  determines the winner, and do not document a warning for overlapping classes. Verified against
  shade-mojo.html — do not rely on ordering as a contract.
  => **Deterministic approach:** exclude `StreamExecCalc.class` from the *Flink* artifact via a
  `<filter>` so only Auron's copy remains. Sketch:
  ```xml
  <filters>
    <filter>
      <artifact>org.apache.flink:flink-table-planner_2.12</artifact>
      <excludes>
        <exclude>org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class</exclude>
        <!-- add one exclude per future shadow FQCN -->
      </excludes>
    </filter>
  </filters>
  ```
  Auron's `StreamExecCalc` comes from `auron-flink-planner` (its own classes), the Flink artifact
  contributes everything except the excluded class, so the union has exactly one `StreamExecCalc`.
  This is structural and ordering-independent — matches the DESIGN Rev 4 A2 intent.

### Relevant transformers (org.apache.maven.plugins.shade.resource)
For an ASF-compliant, drop-in replacement jar:
- `ServicesResourceTransformer` — merges `META-INF/services/*` SPI files. The non-loader planner jar
  ships `META-INF/services/org.apache.flink.table.factories.Factory`; if Auron also registers a
  factory this must merge rather than clobber. (The current assembly uses `AppendingTransformer`
  hard-coded to the legacy `TableFactory` path — `ServicesResourceTransformer` is the general form.)
- `ApacheNoticeResourceTransformer` — merges/produces an ASF-style `META-INF/NOTICE`. Required because
  the shaded jar now redistributes Flink's NOTICE content alongside Auron's (cf. flink-shaded modules).
- `ApacheLicenseResourceTransformer` — normalizes/dedupes `META-INF/LICENSE`. Flink's jar already
  carries `LICENSE` + `LICENSE.txt`.
- `ManifestResourceTransformer` — sets a clean `MANIFEST.MF` (e.g. `Multi-Release`, implementation
  title/version) instead of inheriting an arbitrary one from input jars.
All four are first-party transformers shipped with maven-shade-plugin (no extra dependency).

### Reproducible / deterministic jar
- maven-shade-plugin honors `outputTimestamp` / `project.build.outputTimestamp` for reproducible
  builds (zip entry timestamps). Worth setting for a redistributable ASF artifact.
- `createDependencyReducedPom` should be considered (likely `false` here — this is a terminal
  redistributable, not a library others depend on).
- The current assembly already filters `META-INF/*.SF|*.DSA|*.RSA` (signature files) — keep that;
  re-bundling signed jar signatures into a fat jar breaks verification.

### Verifying the result (post-build assertion)
Three viable options; the context's "build-time structural assertion" maps cleanly to (a) or (b):
- (a) **Unit test reading the jar via `java.util.jar.JarFile`** in the new shaded module: assert
  exactly one entry `org/.../StreamExecCalc.class`, and assert it is Auron's (e.g. compare bytes to
  the `auron-flink-planner` build output, or load it and check
  `Class#getProtectionDomain().getCodeSource()` / a sentinel marker). Most direct, runs in CI.
  Prior art: DESIGN doc already describes `testShadowedClassReplacesFlinkClass`
  (`AURON-1853-DESIGN.md:580`).
- (b) **maven-enforcer-plugin** — there is no built-in "single class" rule, but the `enforcer`
  `BanDuplicateClasses` rule (from `extra-enforcer-rules`) can fail the build on duplicate FQCNs.
  Confirm the extra-enforcer-rules dependency exists in the build before relying on it (Unknown).
- (c) shade's own duplicate reporting — shade logs overlapping classes at build time, but this is a
  log line, not a hard failure, and (per the docs check above) the overlap-warning behavior is not a
  documented contract. Do NOT rely on it as the gate.

Recommend (a) — explicit, deterministic, version-independent, and matches the agreed
"structural assertion provable in CI" scope.

---

## Blockers / Incompatibilities (flink-table-planner-loader is the headline)

### HEADLINE: Flink 1.18's `$FLINK_HOME/lib/` ships the LOADER jar, NOT the jar A2 assumes

**Verified from the official Flink 1.18 docs** (Table Planner configuration, advanced):
- *"By default, the Flink distribution ships with `flink-table-planner-loader` in the `/lib` folder,
  while `flink-table-planner_2.12` resides in `/opt`."*
- *"The two planner JARs contain the same code, but they are packaged differently."*
- *"The two planners cannot co-exist at the same time in the classpath. If you load both of them in
  `/lib` your Table Jobs will fail."*
- Official swap procedure: *(1) Remove `flink-table-planner-loader-1.18.1.jar` from `/lib`;
  (2) Copy `flink-table-planner_2.12-1.18.1.jar` from `/opt` to `/lib`.* With the warning: *"you
  will be constrained to using the Scala version of the Flink distribution."*

**Verified from the loader jar internals** (local m2 has 1.16.1 loader, structurally identical
mechanism to 1.18; 1.18 loader not cached locally but the mechanism is unchanged across versions):
- `flink-table-planner-loader-1.16.1.jar` (~36 MB, **22 entries**) does NOT contain
  `StreamExecCalc.class` directly. It contains a **nested `flink-table-planner.jar`** (39 MB,
  single top-level entry) plus its own `org/apache/flink/table/planner/loader/*` classes
  (`PlannerModule`, `DelegatePlannerFactory`, `DelegateExecutorFactory`, `BaseDelegateFactory`).
- Disassembly of `PlannerModule.class` (`javap -p -c`) confirms it:
  - reads `flink-table-planner.jar` via `ClassLoader.getResourceAsStream("flink-table-planner.jar")`
    (string constants `flink-table-planner_` and `flink-table-planner.jar` present),
  - instantiates `org.apache.flink.core.classloading.ComponentClassLoader` over it
    (`new ComponentClassLoader([URL], parent, owner-first[], component-first[], Map)`),
  - exposes only `DelegatePlannerFactory`/`DelegateExecutorFactory` through `META-INF/services`,
    delegating planner construction into the isolated submodule classloader.
  - error string confirms intent: *"Flink Table planner could not be found ... add a test dependency
    on the flink-table-planner-loader test-jar."*

**Consequence for A2 as literally described in DESIGN Rev 4:**
The DESIGN/context phrasing — *"the user replaces `$FLINK_HOME/lib/flink-table-planner.jar` with the
shaded jar"* — does **not match the default Flink 1.18 distribution**, because there is no
`flink-table-planner.jar` in `lib/` by default; there is `flink-table-planner-loader-1.18.1.jar`.
Two distinct failure modes:

1. If the shaded jar bundles the **non-loader** planner content (the natural reading, since
   `auron-flink-planner` already depends on `flink-table-planner_2.12`), then dropping it into `lib/`
   **alongside the loader** means BOTH planners are on the classpath in `lib/` → per Flink's own
   warning *"your Table Jobs will fail."* So A2 cannot be a pure "add a jar"; the loader jar must be
   **removed** from `lib/` as part of the procedure.

2. The Auron shadow class lives in the FLAT package namespace (`org.apache.flink.table.planner...`).
   Inside the loader, the real `StreamExecCalc` is loaded by the isolated `ComponentClassLoader` from
   the *nested* `flink-table-planner.jar`, NOT from the application/lib classloader. A same-FQCN class
   sitting in a sibling `lib/` jar would be loaded by a *different* classloader and would **not**
   shadow the one the loader resolves internally. So the shadow mechanism is fundamentally
   incompatible with the loader-jar packaging — it only works against the **flat, non-loader**
   planner on the normal app classpath.

### Viable A2 procedure (the real one)
A2 is still mechanically viable, but the user procedure is the **documented planner-swap**, not a
naive "replace flink-table-planner.jar":
1. Remove `flink-table-planner-loader-1.18.1.jar` from `$FLINK_HOME/lib/`.
2. Place `auron-flink-planner-shaded.jar` (= repackaged non-loader `flink-table-planner_2.12-1.18.1`
   content with Auron's `StreamExecCalc` substituted) into `$FLINK_HOME/lib/`.
3. Do NOT also place the stock `flink-table-planner_2.12` from `/opt` — the shaded jar IS the
   replacement for it (two planners must not co-exist).
This is the loader→non-loader swap that Flink already documents, with Auron's shaded jar standing in
for the `/opt` non-loader jar. The Scala-version constraint Flink warns about applies (the shaded jar
is `_2.12`-specific) — acceptable given the repo is 2.12-only today.

=> **The DESIGN Rev 4 / context wording must be corrected** from "replace lib/flink-table-planner.jar"
to "remove lib/flink-table-planner-loader and install the shaded non-loader replacement." This does
not kill A2; it refines the procedure. Flag to the user as a required DESIGN reconciliation.

---

## Unknowns / Risks

- **Runtime-portion placement (the context's open question).** The current A1 `auron-flink-assembly`
  bundles BOTH `auron-flink-runtime` (operators/FFI/native loader) AND `auron-flink-planner`. The A2
  shaded jar is about *planner* shadows + Flink planner content. Whether `auron-flink-runtime` should
  also be folded into the shaded jar or ship as a separate `lib/` jar is an architecture decision, not
  a shade-mechanics fact — defer to the investigate phase / DESIGN. Mechanically, runtime classes can
  be added to the same shade `artifactSet`; the constraint is only that the planner-content + shadows
  produce exactly one `StreamExecCalc`. Keeping runtime as a separate `lib/` jar is cleaner (the
  shaded jar stays a faithful planner replacement; runtime evolves independently).
- **SPI transformer correctness.** The non-loader jar ships
  `META-INF/services/org.apache.flink.table.factories.Factory`; the current assembly appends the
  legacy `org.apache.flink.table.factories.TableFactory` path. If the shaded jar must preserve Flink's
  `Factory` SPI, use `ServicesResourceTransformer` (general) and verify which service files Auron
  actually contributes vs. must pass through. Needs confirmation against `auron-flink-runtime`/planner
  service files.
- **NOTICE/LICENSE content.** Re-bundling Flink's `META-INF/NOTICE`, `LICENSE`, `DEPENDENCIES` into an
  ASF release artifact requires the NOTICE to attribute the embedded Flink (and its transitively
  bundled calcite/janino/etc.). `ApacheNoticeResourceTransformer` helps, but the final NOTICE text is
  a release-compliance review item, not auto-derivable — cf. context constraint citing flink-shaded /
  spark-hadoop-cloud.
- **1.18.1 loader jar not in local m2** — only 1.16.1 was available to disassemble. The loader
  mechanism (`ComponentClassLoader` + nested `flink-table-planner.jar`) is unchanged 1.16→1.18 (same
  `PlannerModule` design), and the 1.18 docs explicitly confirm the lib/opt split, so confidence is
  high; but a confirming `unzip -l` of the actual 1.18.1 loader jar should be done once it is
  downloaded by the build.
- **BanDuplicateClasses availability** — `extra-enforcer-rules` is not confirmed present in the build;
  if chosen as the verification gate it adds a plugin dependency. The JarFile unit test avoids this.
- **Artifact size** — the shaded jar is ~50 MB (the non-loader planner is already a calcite-bundling
  fat jar). Expected and acceptable per DESIGN Rev 4; just noted.

---

## Dead Ends

### DEAD END: "User adds auron shaded jar next to lib/flink-table-planner-loader.jar"
Cannot work, two independent reasons, both evidenced above:
1. Flink 1.18 docs: *"The two planners cannot co-exist ... your Table Jobs will fail."* The loader
   jar must be removed, not merely supplemented.
2. The loader isolates the real planner inside a `ComponentClassLoader` over a nested
   `flink-table-planner.jar` (verified via `javap` on `PlannerModule.class`); a same-FQCN class in a
   sibling `lib/` jar is loaded by a different classloader and never shadows the loader-internal class.

### DEAD END: "Shade the loader jar itself (re-pack flink-table-planner-loader with Auron's class)"
The loader jar contains the planner only as an opaque nested `flink-table-planner.jar` (single zip
entry, verified: 22 top-level entries, one of which is the nested jar). maven-shade operates on class
entries within an artifact; it would not reach into the nested jar to substitute `StreamExecCalc`
without first un-nesting it. The clean path is to shade the **non-loader** `flink-table-planner_2.12`
content directly (where `StreamExecCalc.class` is a normal top-level entry) and have the user perform
the documented loader→non-loader swap. So this isn't an additional dead end so much as confirmation
that the **non-loader artifact is the only sane shade input** — which aligns with the existing
`auron-flink-planner` dependency (`flink-table-planner_2.12`, compile scope).

### NOT a dead end (clarified): relying on dependency/artifactSet ordering for the override
maven-shade-plugin 3.5.2 docs do **not** document an ordering-based "winner" for overlapping classes,
so ordering is not a contract — but this is solved by `<filters>` excluding `StreamExecCalc.class`
from the Flink artifact (documented behavior). A2's core mechanic is sound once the procedure targets
the non-loader jar.
