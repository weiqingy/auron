# Investigation (NOTICE/LICENSE + verification) — AURON #2291

Scope: (A) the exact ASF NOTICE/LICENSE content + placement for the shaded
`auron-flink-planner-shaded` jar that embeds a *modified* copy of Apache Flink's
`flink-table-planner` (ALv2) with Auron's `StreamExecCalc` substituted; (B) the build-time
structural smoke test + the manual jar-swap verification procedure for the reviewhelper.

NO CODE — design on paper. Every claim carries a URL or `file:line`.

---

## NOTICE draft (literal text)

### ASF policy basis (what is *required*)

- ALv2 §4(b): a redistributor of a Derivative Work "must cause any modified files to carry
  prominent notices stating that You changed the files." Source: ALv2 text,
  http://www.apache.org/licenses/LICENSE-2.0 (§4(b)).
- ASF licensing-howto on bundling another Apache product:
  - *"If the dependency supplies a `NOTICE` file, its contents must be analyzed and the
    relevant portions bubbled up into the top-level `NOTICE` file."*
  - *"the ASF copyright line and any other portions of `NOTICE` must be considered for
    propagation."*
  - *"Copyright notifications which have been relocated, rather than removed, from source
    files must be preserved in `NOTICE`."*
  - *"Do not add anything to `NOTICE` which is not legally required … keep `NOTICE` as brief
    and simple as possible."*
  Source: https://infra.apache.org/licensing-howto.html
- Source-header policy: *"Do not add the standard Apache License header to the top of
  third-party source files."* Source: https://www.apache.org/legal/src-headers.html
  ⇒ Auron's substituted `StreamExecCalc.java` keeps the standard ASF header (it already does
  — `StreamExecCalc.java:1-16`) because it is itself an ASF-authored file in the `o.a.flink`
  package, not a verbatim third-party file. The §4(b) "modified files carry prominent
  notices" obligation is satisfied at the distribution level by the NOTICE statement below
  (the conventional ASF satisfaction; an in-file banner is a reviewer/PMC judgment call —
  see Risks). Evidence that `StreamExecCalc` is Auron-authored, not a Flink verbatim copy:
  its imports reference Auron-only packages (`org.apache.auron.flink.runtime.operator.*`,
  `org.apache.auron.flink.table.planner.*`) — `StreamExecCalc.java:26-37`.

### Template followed

`flink-shaded-jackson` `META-INF/NOTICE` "this project bundles X under ALv2" simple-list
shape (research-external.md:120-138), chosen over Spark's `=== NOTICE FOR … ===` delimited
style because this is a single-product (Flink-only) bundle and Flink's own root NOTICE is the
bare ASF boilerplate (nothing verbatim to reproduce — research-external.md:95-97, 217-219).

### Literal draft — `auron-flink-extension/auron-flink-planner-shaded/src/main/resources/META-INF/NOTICE`

```
Apache Auron (Incubating)
Copyright 2025-2026 The Apache Software Foundation.

This product includes software developed at
The Apache Software Foundation (https://www.apache.org/).

This artifact bundles a repackaged copy of Apache Flink's flink-table-planner
and its bundled dependencies, licensed under the Apache License, Version 2.0
(http://www.apache.org/licenses/LICENSE-2.0.txt).

Apache Flink
Copyright 2014-2026 The Apache Software Foundation

This artifact replaces the following class from Apache Flink's flink-table-planner
with a modified version developed by Apache Auron:

- org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc

This product bundles the following Apache-License-2.0 dependencies
(http://www.apache.org/licenses/LICENSE-2.0.txt):

- org.apache.flink:flink-table-planner_2.12:1.18.1
  <FILL: remaining bundled ALv2 artifacts from `mvn dependency:tree` on the
   auron-flink-planner-shaded module — e.g. Calcite, Janino/commons-compiler,
   any non-provided flink-table-planner transitives>
```

Notes on the draft, line by line:
- Lines 1-6: Auron's own NOTICE header — identical to the root `NOTICE` (`NOTICE:2-6`) so the
  `ApacheNoticeResourceTransformer` merge is idempotent and the shaded jar's NOTICE leads with
  Auron's identity. (a) satisfied.
- "This artifact bundles a repackaged copy of Apache Flink's flink-table-planner …": (b)
  satisfied — states the jar embeds a modified copy of an ALv2 product.
- "Apache Flink / Copyright 2014-2026 The Apache Software Foundation": (c) satisfied —
  bubbles up Flink's NOTICE attribution. Flink's root NOTICE is the bare ASF copyright line
  (research-external.md:95-97), so this two-line block is the entirety of what propagates.
  Confirm the exact end-year against the actual bundled `flink-table-planner-1.18.1.jar`
  `META-INF/NOTICE` at build time (Flink 1.18.1 was released 2024 → likely
  `Copyright 2014-2024`; the draft uses 2026 as a placeholder — see Risks).
- "This artifact replaces the following class … with a modified version developed by Apache
  Auron": (d) satisfied — the §4(b) "modified by Auron" statement, naming the substituted
  FQCN.
- The bundled-deps list mirrors `flink-shaded-jackson`'s format. The `<FILL>` line is the
  hand-off to the deps investigation (calcite/janino/etc.) — see LICENSE recommendation for
  where the authoritative list belongs.

---

## LICENSE recommendation

**Minimal correct approach: do NOT modify the bundled LICENSE for an ALv2-only bundle.**

- ASF policy: for bundled ALv2 code with no other-licensed sub-components, *"there is no need
  to modify `LICENSE`"*, though *"for completeness it is useful to list the products and their
  versions."* Source: https://infra.apache.org/licensing-howto.html (research-external.md:115-118).
- So `auron-flink-planner-shaded/src/main/resources/META-INF/LICENSE` = a verbatim copy of the
  Apache License 2.0 text (same as root `LICENSE`, which is the plain ALv2 body —
  `LICENSE:2-4`). No per-component additions are *required*.

**Where the bundled-product list goes (the deps-investigation hand-off):**
- The "useful but not required" product+version list is best placed in the **NOTICE** bundled-
  deps block drafted above (the `<FILL>` line), matching `flink-shaded-jackson`'s convention of
  listing bundled ALv2 artifacts in NOTICE rather than LICENSE (research-external.md:131-137).
  Keeping it in NOTICE (not LICENSE) avoids editing the verbatim license text.
- **Conditional LICENSE change — only if a Category-B / non-ALv2 transitive is actually
  bundled.** The shade pulls `flink-table-planner` *and its non-provided transitives*; if the
  deps tree surfaces a non-ALv2 license (e.g. a BSD/MIT/EPL artifact), THAT requires a
  `LICENSE` pointer block per ASF policy. The deps investigation must classify every bundled
  artifact's license; any non-ALv2 entry is a LICENSE (and possibly bundled-license-file)
  obligation, not a NOTICE one. Flag to the deps-investigation agent: *enumerate licenses, not
  just coordinates.*

---

## Root NOTICE/LICENSE impact

Read both root files:
- `NOTICE` (`/Users/wiyang/workspace/auron/NOTICE`) = bare Auron/ASF boilerplate, 7 lines
  (`NOTICE:1-7`). No bundled-component attributions today.
- `LICENSE` (`/Users/wiyang/workspace/auron/LICENSE`) = plain ALv2 text, no appended
  third-party-license sections (`LICENSE:1-30`, 11358 bytes = just the license body).

**Recommendation: no root-level change required for this PR.** Rationale:
- ASF's bubble-up rule targets the artifact that does the bundling. The root `NOTICE`/`LICENSE`
  describe the **source distribution**, which does not embed Flink binaries — the embedding
  happens only in the *binary* shaded jar produced at `package` time. The per-module
  `src/main/resources/META-INF/{NOTICE,LICENSE}` is the correct home for the bundled-Flink
  attribution, and the shade `ApacheNoticeResourceTransformer` / `ApacheLicenseResourceTransformer`
  merge it into the shaded jar (research-external.md:73-76; research.md:65-68).
- The root files carry no precedent of per-bundle attributions (they are the minimal source-tree
  NOTICE/LICENSE), so adding Flink attribution there would diverge from the existing convention
  and contradict the "keep NOTICE brief" policy for a binary-only embedding.
- **Caveat for the deps investigation:** if Auron later ships an official *binary convenience
  distribution* that aggregates this shaded jar, that distribution needs its own
  `NOTICE-binary`/`LICENSE-binary` (Spark precedent, research-external.md:146-160). That is out
  of this PR's scope (binary release tooling is not part of #2291) — note as a follow-up, not a
  blocker.

### Placement (confirmed)

- New module path: `auron-flink-extension/auron-flink-planner-shaded/src/main/resources/META-INF/NOTICE`
  and `.../META-INF/LICENSE`.
- These are merged by the shade plugin's `ApacheNoticeResourceTransformer` and
  `ApacheLicenseResourceTransformer` (both first-party ASF-compliant transformers,
  research.md:65-68). The shaded jar's final `META-INF/NOTICE` = Auron's module NOTICE merged
  with the bundled `flink-table-planner` jar's own `META-INF/NOTICE` (the transformer
  deduplicates the ASF boilerplate). Confirm the merged result by `unzip -p
  target/auron-flink-planner-shaded-*.jar META-INF/NOTICE` during the structural test
  (see below).
- The existing assembly merges only the legacy `TableFactory` SPI and uses **no** notice/license
  transformers today (`auron-flink-assembly/pom.xml:60-64`). The new shaded module MUST add
  `ApacheNoticeResourceTransformer` + `ApacheLicenseResourceTransformer` (gap flagged in
  research.md:65-68, 86-88), or the bundled Flink NOTICE/LICENSE entries collide/get dropped.

---

## Structural test design (distinguishing signal named)

### Prior art read: `testShadowedClassReplacesFlinkClass`

`AURON-1853-DESIGN.md:580` describes the existing classpath-verification test:
> load `StreamExecCalc.class` via `Class.forName`, assert its
> `getProtectionDomain().getCodeSource().getLocation()` points at `auron-flink-planner` (not
> `flink-table-planner`). Skipped if the test runs without the auron JAR ahead of Flink's on
> the classpath.

What it does: a **runtime classloader** check — it asserts that *whatever the JVM resolved* for
the FQCN came from the Auron jar. It depends on classpath ordering at test time and is *skipped*
when ordering isn't set up. It does **not** open the shaded jar, does **not** assert "exactly
one" entry, and cannot detect a duplicate `StreamExecCalc.class` co-residing in one fat jar.

**Verdict: write a NEW test, do not extend.** Different mechanism (jar-file structural
inspection vs. runtime class resolution), different module (lives in the new shaded module's
test scope, runs after `package`), different failure it catches (duplicate / wrong-bytes entry
inside the fat jar). The two are complementary, not overlapping.

### New test: `ShadedPlannerJarStructureTest` (JUnit, Java)

Lives at
`auron-flink-extension/auron-flink-planner-shaded/src/test/java/org/apache/auron/flink/planner/shaded/ShadedPlannerJarStructureTest.java`.

Contract (3 assertions):
1. **Exactly one entry** named
   `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class` in the shaded
   jar. (`JarFile.entries()` → filter by name → assert count == 1. A `JarFile` cannot itself
   hold two entries with the identical name, but a *malformed* shade could; more importantly
   this asserts the entry is *present at all* — guarding against a filter that accidentally
   excluded both copies.)
2. **It is Auron's, not stock Flink's** — read the single entry's bytes and assert the
   distinguishing signal below.
3. **The bundled Flink planner is actually present** — assert a stock planner class that Auron
   does NOT shadow is present, e.g.
   `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecJoin.class` (proves the
   fat jar bundled `flink-table-planner`, not just Auron's thin shadow). Pick any non-shadowed
   `StreamExec*` confirmed to exist in the 1.18.1 planner jar.

### Distinguishing signal (named, concrete)

Auron's `StreamExecCalc` carries marks that stock Flink's class cannot have. Best signal, in
order of robustness:

- **Primary — constant-pool reference to an Auron-only class.** Auron's class references
  `org.apache.auron.flink.runtime.operator.FlinkAuronCalcOperator` (`StreamExecCalc.java:27`,
  instantiated at `:191-192`) and `org/apache/auron/flink/table/planner/UnsupportedFlinkNodeRecorder`
  (`StreamExecCalc.java:28`, called at `:293,:323`). Stock Flink's `StreamExecCalc` references
  none of `org.apache.auron.*`. Test reads the entry bytes and asserts the byte sequence for
  the UTF-8 constant `org/apache/auron/flink/runtime/operator/FlinkAuronCalcOperator`
  (internal slash form) appears in the class file. This is a byte-`indexOf` over the class
  bytes — no classloading, no Flink on the test classpath required. **This is the recommended
  signal** because it works purely structurally on the jar entry.
- **Secondary (confirmatory) — Auron-only member.** Auron's class declares a
  `translateToFlinkCalc(PlannerBase, ExecNodeConfig)` method (`StreamExecCalc.java:213`) and a
  `private static final AtomicBoolean ACTIVATION_LOGGED` field (`StreamExecCalc.java:98`),
  neither present in stock Flink. If a load-the-class variant is preferred,
  `clazz.getDeclaredMethod("translateToFlinkCalc", PlannerBase.class, ExecNodeConfig.class)`
  (no exception thrown) is a positive Auron signal. Avoid the load-the-class path for the
  build smoke test, though — it drags Flink planner internals onto the test classpath and
  re-introduces classpath-ordering fragility. Prefer the byte-`indexOf` primary signal.

Named signal to use: **the UTF-8 constant-pool string
`org/apache/auron/flink/runtime/operator/FlinkAuronCalcOperator` present in the
`StreamExecCalc.class` bytes.**

### Locating the jar + running in the right phase

- Jar path: glob `target/auron-flink-planner-shaded-*.jar` under the module's `${project.build.directory}`.
  Use `System.getProperty("project.build.directory")` injected via surefire `systemPropertyVariables`,
  or resolve relative to the test class's own code source then walk to `target/`. A glob (not a
  hardcoded version) keeps it version-agnostic.
- Phase ordering: the test asserts on the *shaded* jar, which is produced in the `package`
  phase (`shade:shade`, `auron-flink-assembly/pom.xml:78-81`), but unit tests run in `test`
  (before `package`). Resolution: bind the test to the **`verify` phase via
  `maven-failsafe-plugin`** as an integration test (`ShadedPlannerJarStructureIT`), so it runs
  *after* `package`. This is the standard Maven idiom for "assert on the built artifact." A
  plain surefire unit test in `test` would run before the jar exists and fail to find it.
- CI hook: `.github/workflows/flink.yml:45` currently builds only `auron-flink-planner -am` and
  never reaches the assembly (research.md:84-85). The new module must be added to the Flink CI
  reactor so `mvn verify` runs the failsafe IT. Flag: ensure the workflow builds
  `auron-flink-planner-shaded -am` (or the full `-Pflink-1.18` reactor) at `verify`, not just
  `package`.

---

## Manual jar-swap procedure (reviewhelper)

This is a documented manual run for the reviewhelper, NOT a CI test. It uses Flink 1.18's
documented loader→non-loader swap (research-external.md:47-74; Flink 1.18 "Advanced
configuration topics / Anatomy of Table dependencies",
https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/configuration/advanced/).

Precondition: a Flink 1.18.1 distribution (`$FLINK_HOME`). By default `lib/` contains
`flink-table-planner-loader-1.18.1.jar` (isolated classpath), and the real planner sits in
`opt/flink-table-planner_2.12-1.18.1.jar` (research.md headline; research-external.md:47-61).
There is no `lib/flink-table-planner.jar` to replace — the swap is what makes the planner
addressable.

```bash
# 1. Remove Flink's loader jar from lib/ (the loader and a real planner cannot coexist —
#    Flink docs: "The two planners cannot co-exist at the same time in the classpath.")
rm "$FLINK_HOME/lib/flink-table-planner-loader-1.18.1.jar"

# 2. Drop Auron's shaded planner into lib/. This jar IS the non-loader
#    flink-table-planner_2.12 content with Auron's StreamExecCalc substituted in, so it
#    replaces BOTH the loader jar (step 1) and the need to copy opt/ into lib/.
cp auron-flink-extension/auron-flink-planner-shaded/target/auron-flink-planner-shaded-*.jar \
   "$FLINK_HOME/lib/"

# 3. Place the Auron runtime jar (operators + FFI + libauron.so) on the classpath.
#    Under the leaning "separate runtime jar" decision (research.md:94-97), this is a distinct
#    lib/ jar, not bundled into the shaded planner. (Confirm the exact runtime artifact name at
#    design time.)
cp <auron-flink-runtime-jar> "$FLINK_HOME/lib/"

# 4. Start a local cluster and run a Calc SQL via the SQL client.
"$FLINK_HOME/bin/start-cluster.sh"
"$FLINK_HOME/bin/sql-client.sh"
```

In the SQL client, run a projection+filter Calc and confirm native conversion:

```sql
CREATE TABLE src (a INT, b INT) WITH ('connector'='datagen','number-of-rows'='5');
CREATE TABLE sink (c INT) WITH ('connector'='print');
INSERT INTO sink SELECT a + 1 FROM src WHERE a > 0;
```

Confirm Auron engaged (two independent signals, both from `StreamExecCalc.java`):
- **Activation log** — TaskManager/JobManager log shows the one-shot INFO
  `"Auron StreamExecCalc shadow active (loaded from <…auron-flink-planner-shaded…jar>)."`
  (`StreamExecCalc.java:309-316`). The code-source in the line must point at the shaded jar —
  if it points at a Flink jar, ordering is wrong.
- **Strict mode (optional, stronger)** — set `FAIL_BACK_FLINK_ENGINE_ENABLED=false`
  (`StreamExecCalc.java:69-70,180-189`); a supported Calc then runs natively, and any
  unsupported expression throws `IllegalStateException` instead of silently falling back —
  proving the shadow, not stock Flink, is handling the Calc.

Teardown: `"$FLINK_HOME/bin/stop-cluster.sh"`; restore by re-adding the original loader jar
and removing the Auron jars.

---

## Verified Assumptions

- Root `NOTICE` is 7-line Auron/ASF boilerplate with no bundled attributions — `NOTICE:1-7`
  (read).
- Root `LICENSE` is plain ALv2 body, no appended third-party sections — `LICENSE:1-30` (read),
  11358 bytes total.
- Auron's `StreamExecCalc` is Auron-authored (references `org.apache.auron.*`), keeps the
  standard ASF source header, and declares Auron-only members (`ACTIVATION_LOGGED`,
  `translateToFlinkCalc`) — `StreamExecCalc.java:1-16, 26-37, 98, 213`. → distinguishing
  signal is real and byte-detectable.
- Existing assembly shade has NO notice/license transformer and merges only the legacy
  `TableFactory` SPI — `auron-flink-assembly/pom.xml:60-74`. → new module must add the Apache
  Notice/License transformers (and modern SPI merges, per research.md:86-88).
- Prior-art `testShadowedClassReplacesFlinkClass` is a runtime classloader check that is
  skipped without classpath ordering — `AURON-1853-DESIGN.md:580`. → new structural jar test is
  complementary, not a duplicate.
- Flink 1.18 loader→opt swap procedure and "two planners cannot coexist" warning are
  documented — research-external.md:47-74, Flink 1.18 advanced-config docs.
- ASF policy (NOTICE bubble-up, §4(b) modified-files notice, LICENSE-unchanged-for-ALv2-bundle)
  cited from infra.apache.org/licensing-howto.html, ALv2 §4(b), apache.org/legal/src-headers.html
  — research-external.md:76-118.

## Risks / Open items

- **Bundled-deps list is a placeholder.** The NOTICE `<FILL>` line and the conditional LICENSE
  obligation both depend on `mvn dependency:tree` on the new shaded module, which doesn't exist
  yet. Hand-off to the deps-investigation agent: enumerate every bundled artifact **with its
  license** (not just coordinates), so (a) ALv2 ones go in the NOTICE list and (b) any non-ALv2
  one triggers a LICENSE block. Until the module's shade config is written, the exact set is
  unknown.
- **Flink NOTICE copyright end-year.** The draft uses `Copyright 2014-2026`; the actual bundled
  `flink-table-planner-1.18.1.jar` `META-INF/NOTICE` (1.18.1 released 2024) likely says
  `2014-2024`. Reproduce the *bundled jar's* exact line — verify with
  `unzip -p flink-table-planner-1.18.1.jar META-INF/NOTICE` at build time and correct the draft.
- **In-file §4(b) banner is a judgment call.** Distribution-level NOTICE statement is the
  conventional ASF satisfaction (research-external.md:99-113). Whether reviewers/PMC also want
  an in-file "modified by Auron" banner on `StreamExecCalc.java` is not mandated by policy —
  surface to the user/PMC, don't decide unilaterally.
- **`numbers`/transformer gap could swallow the Flink NOTICE.** If the new module forgets the
  `ApacheNoticeResourceTransformer`, the bundled Flink `META-INF/NOTICE` is silently dropped
  during shade (no merge) → the shaded jar would violate the bubble-up rule. The structural
  test SHOULD additionally assert `META-INF/NOTICE` exists in the shaded jar and contains both
  "Apache Auron" and "Apache Flink" — cheap guard against the transformer being omitted.
- **CI does not currently reach `package`/`verify` for the assembly** (research.md:84-85). The
  failsafe IT only runs if the workflow is updated to `mvn verify` the new module — a build-config
  change this PR must include, else the smoke test never runs in CI.
