# Research (Gluten + ASF NOTICE prior art) — AURON #2291

Scope: external prior art for (a) Flink class-shadowing and (b) ASF license/NOTICE
for embedded modified third-party (Apache Flink) content. All claims cite a URL.

## Proven Patterns

### 1. Apache Gluten ships shadow `StreamExec*` via A1 (classpath precedence), NOT shading

**Finding (important): Gluten does NOT shade or bundle `flink-table-planner`. It uses the
A1-explicit overlay variant — classpath precedence via `config.sh`.** This means Auron's
A2 (shaded fat jar replacing `flink-table-planner.jar`) has **no direct Flink prior art in
Gluten**. The NOTICE precedent for embedded-modified-Flink-content must come from
`flink-shaded` / Spark, not Gluten.

Evidence:
- gluten-flink module layout: `dev/`, `docs/`, `loader/`, `patches/`, `planner/`,
  `runtime/`, `ut/` plus `pom.xml`, `README.md`.
  https://github.com/apache/incubator-gluten/tree/main/gluten-flink
- `gluten-flink/planner/pom.xml`: declares
  `org.apache.flink:flink-table-planner_${scala.binary.version}` with **`<scope>provided</scope>`**,
  has **no `maven-shade-plugin`**, and produces a **thin jar containing only the shadow
  classes** — flink-table-planner content is excluded.
  https://raw.githubusercontent.com/apache/incubator-gluten/main/gluten-flink/planner/pom.xml
- `gluten-flink/docs/Flink.md` deployment procedure (A1 — additive jars + classpath
  precedence, NOT replacement):
  - Jars go in a **dedicated dir**, not replacing Flink libs: `mkdir -p gluten_lib`, then
    symlink runtime/loader/velox4j jars into `$FLINK_HOME/gluten_lib/`.
  - Activation: **modify `constructFlinkClassPath` in `$FLINK_HOME/bin/config.sh`** to
    **prepend** the gluten jars: `GLUTEN_JAR="$FLINK_HOME/gluten_lib/gluten-flink-loader-1.6.0.jar:..."`
    then `echo "$GLUTEN_JAR""$FLINK_CLASSPATH""$FLINK_DIST"`. Prepending makes gluten
    classes win over stock Flink classes (first-on-classpath wins).
  - Explicit statement of no bundling: *"both Gluten for Flink and Velox4j have not a
    bundled jar including all jars depends on"* — users add dependency jars manually.
  https://raw.githubusercontent.com/apache/incubator-gluten/main/gluten-flink/docs/Flink.md

**Implication for Auron's open question (runtime placement under "A2 replaces A1"):**
Gluten keeps runtime and planner shadows as **separate jars** on the classpath (loader,
runtime, planner are distinct artifacts, all symlinked into `gluten_lib/` and prepended).
Gluten gives no precedent for *bundling runtime into the planner-replacement jar* because
Gluten never replaces the planner jar at all. So Gluten supports the reading that the
**runtime portion can remain a separate `lib/` jar**; only the planner shadows need to be
co-located with the bundled flink-table-planner content (because A2's whole point is
"exactly one class per shadowed FQCN ⇒ structural activation"). This is an inference from
Gluten's structure, not a documented Auron decision — flagged under Unknowns.

### 2. Flink 1.18 planner-loader split — confirms `lib/flink-table-planner.jar` is NOT the default target

Flink 1.18 ships **`flink-table-planner-loader-1.18.x.jar` in `/lib`** (query planner behind
an isolated classpath — you cannot address `org.apache.flink.table.planner` directly) and
**`flink-table-planner_2.12.jar` in `/opt`**.

The documented/supported way to access planner internals is to **swap the loader jar for the
opt jar**: *"you can swap the JARs (copying and pasting `flink-table-planner_2.12.jar` in the
distribution `/lib` folder)."* Constraints/warnings:
- *"you will be constrained to using the Scala version of the Flink distribution that you are using."*
- *"The two planners cannot co-exist at the same time in the classpath. If you load both of
  them in `/lib` your Table Jobs will fail."*
- Future direction: *"In the upcoming Flink versions, we will stop shipping the
  `flink-table-planner_2.12` artifact in the Flink distribution"* and recommends migrating off
  planner internals.

Source: Flink 1.18 docs, "Advanced configuration topics / Anatomy of Table dependencies":
https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/configuration/advanced/

**Critical implication for AURON #2291's deployment target:** Auron's shadow lives in
`org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc` — a *planner internal*.
By default `/lib` contains the **loader** jar (isolated classpath), not `flink-table-planner.jar`.
The shaded-jar-replaces-`lib/flink-table-planner.jar` model presupposes the user has already
performed the loader→opt swap (i.e. `flink-table-planner_2.12.jar` is the thing in `/lib`).
The context (`context.md` line 16) says "User replaces `$FLINK_HOME/lib/flink-table-planner.jar`",
which only exists after that swap. The deployment doc (deferred to #1892) must state this
precondition, and the two-planners-cannot-coexist warning means Auron's shaded jar **is** the
non-loader planner (with the StreamExecCalc override) and the loader jar must be removed.

### 3. ASF NOTICE / LICENSE conventions for bundling modified ALv2 Apache content

The authoritative ASF source is the licensing-howto (the policy pages redirect here):
https://infra.apache.org/licensing-howto.html

**(a) NOTICE is minimal and legally-required-only.**
- *"The `NOTICE` file is reserved for a certain subset of legally required notifications"*;
  *"It is important to keep `NOTICE` as brief and simple as possible, as each addition places
  a burden on downstream consumers."*
- *"**Do not** add anything to `NOTICE` which is not legally required."*

**(b) Bundling another Apache product — what propagates.**
- *"It is not necessary to duplicate the line 'This product includes software developed at the
  Apache Software Foundation...', though the ASF copyright line and any other portions of
  `NOTICE` must be considered for propagation."*
- *"If the dependency supplies a `NOTICE` file, its contents must be analyzed and the relevant
  portions bubbled up into the top-level `NOTICE` file."*
  ⇒ For Auron: analyze Flink's NOTICE and bubble up its required portions (the relevant
  copyright/attribution lines) into Auron's NOTICE. Flink's own root NOTICE is essentially the
  bare ASF boilerplate (`Apache Flink / Copyright 2014-20xx The Apache Software Foundation / ...`),
  so the bubbled-up content is small.
  Flink root NOTICE: https://raw.githubusercontent.com/apache/flink-shaded/master/NOTICE

**(c) Modified third-party files & relocated notices (ALv2 §4(b)).**
- *"Copyright notifications which have been relocated, rather than removed, from source files
  must be preserved in `NOTICE`."* (aligns with ALv2 §4(b) — preserve attribution notices).
  https://infra.apache.org/licensing-howto.html
- Source-header policy: *"Do not add the standard Apache License header to the top of
  third-party source files"* and *"Minor modifications/additions to third-party source files
  should typically be licensed under the same terms as the rest of the third-party source for
  convenience."* The policy does **not** mandate a per-file modification notice for minor
  edits, but ALv2 §4(b) requires "prominent notices stating that You changed the files."
  https://www.apache.org/legal/src-headers.html
  ⇒ For Auron's substituted `StreamExecCalc` (a modified copy of a Flink file): keep Flink's
  original header, do not slap Auron's ALv2 header on it, and carry a "modified by Auron"
  statement (per §4(b)) — see acceptance criterion. The NOTICE-level "this product includes
  modified Apache Flink content" statement satisfies the prominent-notice obligation at the
  distribution level.

**(d) LICENSE side.**
- For bundled ALv2 code with no other-licensed sub-components: *"there is no need to modify
  `LICENSE`"*, though *"for completeness it is useful to list the products and their versions."*
  https://infra.apache.org/licensing-howto.html

### 4. Concrete NOTICE format precedents (templates to follow)

**flink-shaded bundled-jar NOTICE** (`flink-shaded-jackson` META-INF/NOTICE) — the canonical
"this project bundles X under ALv2" format:
```
flink-shaded-jackson
Copyright 2014-2026 The Apache Software Foundation

This project includes software developed at The Apache Software Foundation
(http://www.apache.org/).

This project bundles the following dependencies under the Apache Software
License 2.0 (http://www.apache.org/licenses/LICENSE-2.0.txt):

- com.fasterxml.jackson.core:jackson-annotations:2.20
- com.fasterxml.jackson.core:jackson-core:2.20.1
- ...
```
https://raw.githubusercontent.com/apache/flink-shaded/master/flink-shaded-jackson-parent/flink-shaded-jackson-2/src/main/resources/META-INF/NOTICE

⇒ Auron's shaded-jar NOTICE should follow this exact shape: header + "This project bundles the
following dependencies under the Apache Software License 2.0:" + the list of bundled
flink-table-planner (+ transitive) artifacts with versions. Add the §4(b) modification
statement for the substituted classes (e.g. "This product includes a modified copy of
`org.apache.flink.table.planner...StreamExecCalc` from Apache Flink, modified by Apache Auron.").

**Spark binary NOTICE** (`NOTICE-binary`) — per-component delimited blocks for bundled content:
```
Apache Spark
Copyright 2014 and onwards The Apache Software Foundation.

This product includes software developed at
The Apache Software Foundation (http://www.apache.org/).

=== NOTICE FOR com.clearspring.analytics:streams ===
stream-api
Copyright 2016 AddThis
This product includes software developed by AddThis.
=== END OF NOTICE FOR com.clearspring.analytics:streams ===
```
https://raw.githubusercontent.com/apache/spark/master/NOTICE-binary

⇒ Spark uses `=== NOTICE FOR <ga> === ... === END ===` delimiters when the bundled component
ships its own NOTICE content. The flink-shaded simple-list style is lighter and likely the
better fit for Auron's single-product (Flink-only) bundle; Spark's delimited style is the
precedent if Flink's NOTICE content needs verbatim reproduction.

## API Surface (deployment procedures)

- **Gluten (A1, additive):** `mkdir gluten_lib`; symlink jars in; prepend to classpath in
  `bin/config.sh` `constructFlinkClassPath`. No replacement, no shading.
  (gluten-flink/docs/Flink.md, cited above)
- **Flink planner swap (supported, replacement):** copy `opt/flink-table-planner_2.12.jar` into
  `/lib`, remove `lib/flink-table-planner-loader-*.jar` (two planners cannot coexist).
  (Flink 1.18 advanced-config docs, cited above)
- **Auron A2 (proposed, replacement of the swapped jar):** replace the in-`/lib`
  `flink-table-planner_2.12.jar` (post-swap) with `auron-flink-planner-shaded.jar`. Precondition:
  the loader→opt swap. Deferred deployment doc → #1892.

## Blockers / Incompatibilities

- **Default `/lib` has no `flink-table-planner.jar` to replace** — it has the *loader* jar.
  Replacing "`lib/flink-table-planner.jar`" only makes sense after the user performs Flink's
  documented loader→opt swap. The A2 deployment story is therefore two-step (swap, then replace)
  and must be documented as such. (Flink 1.18 advanced-config docs.) Not a code blocker for the
  shade-plugin work, but a correctness blocker for the deployment doc (#1892) and the smoke
  procedure in the reviewhelper.
- **Two planners cannot coexist** — the shaded jar replaces the planner; the loader jar must be
  absent. A smoke run that leaves the loader jar in `/lib` will fail at job submission.
  (Flink 1.18 advanced-config docs.)

## Unknowns / Risks

- **Runtime-jar placement under "A2 replaces A1" (the context open question, line 31-36) is not
  resolvable from external prior art.** Gluten keeps runtime separate (no planner replacement at
  all), which is suggestive but not authoritative for Auron's bundle-into-the-replacement design.
  Recommend deciding internally: bundling `auron-flink-runtime` (FFI/native loader) into the
  planner-replacement jar mixes two concerns and risks classloader surprises; keeping it a
  separate `lib/` jar matches Gluten's separation. Needs an Auron decision, not external evidence.
- **Exact set of bundled artifacts** for the NOTICE list — the shade plugin will bundle
  `flink-table-planner` *and its non-provided transitive deps*. The NOTICE must enumerate every
  ALv2 artifact actually bundled (mvn dependency:tree on the shaded module), plus call out any
  non-ALv2 (category-B) transitive deps which would also require LICENSE changes. Not yet
  enumerated — depends on the shade config that this PR writes.
- **Per-file §4(b) modification notice wording** — ASF policy requires "prominent notices stating
  that You changed the files" for modified ALv2 files but doesn't prescribe exact text. The
  distribution-level NOTICE statement is the safe, conventional satisfaction; whether reviewers
  also want an in-file banner on `StreamExecCalc` is a judgment call for the Auron PMC/reviewers.

## Dead Ends

- **Gluten as a shading/NOTICE precedent for embedded modified Flink content: dead end.** Gluten
  neither shades nor bundles flink-table-planner (provided scope, thin jar, classpath-precedence
  deployment). It is a valid precedent for the *shadow-class technique* and for *separate-jar
  runtime placement*, but contributes nothing to the A2 fat-jar NOTICE question. Use
  `flink-shaded` (simple bundled-deps list) and Spark `NOTICE-binary` (delimited per-component
  blocks) as the NOTICE precedents instead.
- **flink-shaded root NOTICE: not a useful template** — it is the bare ASF boilerplate
  (`Apache Flink / Copyright ... / This product includes software developed at The ASF`). The
  useful template is the per-module bundled-jar NOTICE (flink-shaded-jackson), cited above.
