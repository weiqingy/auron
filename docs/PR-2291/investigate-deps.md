# Investigation (shade config + dependencies) — AURON #2291

Scope: exact `maven-shade-plugin` 3.5.2 config for a new `auron-flink-planner-shaded` module that
produces all of `flink-table-planner_2.12-1.18.1` content with Auron's `StreamExecCalc` winning, as a
drop-in for `$FLINK_HOME/lib/` (after removing the loader jar).

All claims are evidence-backed. Plugin options are verified against the shade-mojo 3.5.2 docs
(maven.apache.org/plugins/maven-shade-plugin 3.5.2) and against the existing in-repo shade config at
`auron-flink-extension/auron-flink-assembly/pom.xml:42-86`, which already uses
`maven-shade-plugin:${maven.plugin.shade.version}` (= 3.5.2, `pom.xml:73`). Every option named below
is confirmed present either in that working in-repo config or in the shade-mojo doc.

---

## Proposed shade configuration (annotated XML sketch — every option verified to exist in 3.5.2)

This is a sketch for the new module `auron-flink-extension/auron-flink-planner-shaded/pom.xml`. It
mirrors the structure of the existing assembly shade config (proven to work in this repo) and adds the
class-override filter + ASF transformers.

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-shade-plugin</artifactId>
      <version>${maven.plugin.shade.version}</version>   <!-- 3.5.2, pom.xml:73 -->
      <configuration>

        <!-- Terminal redistributable, not a library others depend on. -->
        <createDependencyReducedPom>false</createDependencyReducedPom>   <!-- shade-mojo param, exists -->

        <!-- Name the output auron-flink-planner-shaded-<version>.jar (see §finalName). -->
        <finalName>${project.artifactId}-${project.version}</finalName>   <!-- shade-mojo param, exists -->

        <artifactSet>
          <includes>
            <!-- Auron override classes (Auron's StreamExecCalc lives here). -->
            <include>org.apache.auron:auron-flink-planner</include>
            <!-- The non-loader Flink planner fat jar to re-package. -->
            <include>org.apache.flink:flink-table-planner_2.12</include>
          </includes>
        </artifactSet>

        <filters>
          <!-- Class-override: drop the stock StreamExecCalc so only Auron's copy survives.
               Single .class entry confirmed (see §inner-class completeness) — no inner classes. -->
          <filter>
            <artifact>org.apache.flink:flink-table-planner_2.12</artifact>
            <excludes>
              <exclude>org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class</exclude>
            </excludes>
          </filter>
          <!-- Strip jar signatures so the fat jar verifies (kept from the existing assembly config,
               auron-flink-assembly/pom.xml:65-73). -->
          <filter>
            <artifact>*:*</artifact>
            <excludes>
              <exclude>META-INF/*.SF</exclude>
              <exclude>META-INF/*.DSA</exclude>
              <exclude>META-INF/*.RSA</exclude>
            </excludes>
          </filter>
        </filters>

        <transformers>
          <!-- Merge META-INF/services/* so the Flink Factory SPI is preserved (see §SPI). -->
          <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
          <!-- ASF NOTICE / LICENSE for a redistributable that re-bundles Flink content. -->
          <transformer implementation="org.apache.maven.plugins.shade.resource.ApacheNoticeResourceTransformer"/>
          <transformer implementation="org.apache.maven.plugins.shade.resource.ApacheLicenseResourceTransformer"/>
          <!-- Clean MANIFEST instead of inheriting an arbitrary input manifest. -->
          <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer"/>
        </transformers>

      </configuration>
      <executions>
        <execution>
          <phase>package</phase>
          <goals><goal>shade</goal></goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Option/transformer existence verification:

| Element | Verified how |
|---|---|
| `<finalName>` | shade-mojo doc param `finalName`; also note the existing assembly relies on the default name (no `<finalName>`), confirming the param is optional. |
| `<createDependencyReducedPom>` | shade-mojo doc param `createDependencyReducedPom` (default `true`). |
| `<artifactSet><includes>` | shade-mojo doc param `artifactSet`; the assembly uses the sibling `<excludes>` form (`auron-flink-assembly/pom.xml:49-58`). Both `includes` and `excludes` are documented. |
| `<filters><filter><artifact>` + `<excludes>` | shade-mojo doc param `filters`; **proven in-repo** at `auron-flink-assembly/pom.xml:65-73` (the `*:*` signature filter uses exactly this shape). |
| `ServicesResourceTransformer` | first-party shade transformer class `org.apache.maven.plugins.shade.resource.ServicesResourceTransformer` — documented in shade-mojo "Resource Transformers". |
| `ApacheNoticeResourceTransformer` | first-party — documented. |
| `ApacheLicenseResourceTransformer` | first-party — documented. |
| `ManifestResourceTransformer` | first-party — documented. |
| `AppendingTransformer` (the one the assembly uses, `pom.xml:60-64`) | first-party — proven in-repo; **not** needed here, see §SPI. |

`<artifact>` glob syntax for filters: the value is `groupId:artifactId[:type[:classifier]]` with `*`
wildcards allowed per segment — confirmed by the in-repo `*:*` usage (`auron-flink-assembly/pom.xml:67`)
which matches every artifact. So `org.apache.flink:flink-table-planner_2.12` is a valid exact-match
artifact selector (no version segment needed; version is not part of the filter coordinate).

> Note on the `${scala.binary.version}` interpolation: the in-repo dep uses
> `flink-table-planner_${scala.binary.version}` (`auron-flink-planner/pom.xml:132`). The filter
> `<artifact>` and `artifactSet` should use the resolved value `flink-table-planner_2.12` (or the same
> property) — `scala.binary.version=2.12` is set at `auron-flink-planner/pom.xml:33`. Whichever form,
> keep planner and filter consistent.

---

## Dependencies (block + scopes; Q2 runtime both options)

### Verified: how `auron-flink-planner` pulls the Flink planner artifact
`auron-flink-extension/auron-flink-planner/pom.xml:130-134` declares
`org.apache.flink:flink-table-planner_${scala.binary.version}:${flink.version}` with **no `<scope>`**
⇒ **compile scope**. (Confirmed: lines 130-134 have no scope element, unlike the surrounding deps
which are explicitly `provided`.) So depending on `auron-flink-planner` transitively pulls the
non-loader planner jar at compile scope into the shaded module's reactor — shade will see it.

### Recommended dependencies block for `auron-flink-planner-shaded`
```xml
<dependencies>
  <dependency>
    <groupId>org.apache.auron</groupId>
    <artifactId>auron-flink-planner</artifactId>
    <version>${project.version}</version>
  </dependency>
</dependencies>
```

Rationale: `auron-flink-planner` already brings (a) Auron's override classes incl. `StreamExecCalc`
and (b) `flink-table-planner_2.12` at compile scope transitively. Declaring the Flink planner artifact
*directly* in the shaded module is redundant and risks version drift. Mirror the existing assembly,
which lists only `auron-flink-runtime` + `auron-flink-planner` and lets the shade `artifactSet` decide
what gets bundled (`auron-flink-assembly/pom.xml:29-40`).

**However** — to make the `<artifactSet><includes>` of `flink-table-planner_2.12` resolvable and
explicit (shade includes operate on the resolved dependency set), the cleanest, least-surprising form
is to ALSO declare the planner artifact directly so the coordinate is unambiguous at the shaded
module level. Both work; recommendation: declare `auron-flink-planner` only and rely on the transitive
compile-scope planner (matches assembly precedent). If the Investigate→Design phase prefers an
explicit coordinate, add `flink-table-planner_2.12` at compile scope directly — it is idempotent
because the transitive one is already compile scope.

### Q2 — `auron-flink-runtime` placement (present both options, do NOT decide)

The runtime carries operators, FFI, the native loader, and two main-resource SPI files
(`META-INF/services/org.apache.flink.table.factories.Factory` →
`org.apache.auron.flink.connector.kafka.AuronKafkaDynamicTableFactory`, and
`.../AuronAdaptorProvider` → `org.apache.auron.jni.FlinkAuronAdaptorProvider`; verified at
`auron-flink-extension/auron-flink-runtime/src/main/resources/META-INF/services/*`). `libauron.so`
rides `auron-core → auron-flink-runtime` (research-internal: `auron-core/pom.xml:89-98`).

| Option | Dep block change | Shade impact | Trade-off |
|---|---|---|---|
| **Q2-A: runtime stays a separate `lib/` jar** (research leaning) | shaded module depends on `auron-flink-planner` only. Runtime ships via its own jar (or the existing assembly). | Shaded jar = pure planner replacement. Only the Flink `Factory` SPI passes through; **no `Factory` SPI merge needed** (Auron's `Factory` entry lives in the separate runtime jar). | Cleaner separation; shaded jar stays a faithful planner stand-in; runtime evolves independently. Two `lib/` jars to deploy. |
| **Q2-B: runtime bundled into the shaded jar** | add `<dependency> auron-flink-runtime` (compile) and `<include>org.apache.auron:auron-flink-runtime</include>` to `artifactSet`. | **`ServicesResourceTransformer` becomes mandatory** to merge the two `Factory` SPI files (Flink's `DefaultExecutorFactory`/`DefaultParserFactory`/`DefaultPlannerFactory` + Auron's `AuronKafkaDynamicTableFactory`). Also must carry the `AuronAdaptorProvider` SPI + `libauron.so`. | One `lib/` jar to deploy; but the shaded jar is no longer a faithful planner-only replacement, and SPI-merge correctness is load-bearing (clobber = native loading / table factory break). |

The `ServicesResourceTransformer` is in the sketch above defensively — it is **required** under Q2-B
and **harmless** under Q2-A (it just passes the single Flink `Factory` SPI through). Design picks the
option; the shade config is written so either works.

---

## StreamExecCalc inner-class / filter completeness  (correctness trap)

**Stock Flink `StreamExecCalc` has NO inner classes.** Verified by `unzip -l` on
`flink-table-planner_2.12-1.18.1.jar` — exactly one matching entry:
```
4185  org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.class
```
No `StreamExecCalc$1.class`, `StreamExecCalc$Foo.class`, etc. (grep for
`nodes/exec/stream/StreamExecCalc` returned the single line above). So the single `<exclude>` of
`StreamExecCalc.class` is **complete** for the override — no inner-class excludes required.

**Auron's `StreamExecCalc` source** (`auron-flink-planner/.../StreamExecCalc.java`, read in full)
defines no nested/anonymous classes: all helper methods are plain instance/static methods
(`logActivationOnce`, `recordFallback`, `tryBuildAuronPlan`, `translateToFlinkCalc`); the only
lambda-shaped constructs are absent (loops + `Optional`, no lambda literals). `AtomicBoolean` and
`Logger` are static fields, not nested types. Therefore Auron's compiled output is a single
`StreamExecCalc.class` as well — no `StreamExecCalc$1.class` to worry about on the Auron side either.
Net: union has exactly one `StreamExecCalc.class` (Auron's), with no orphaned stock inner classes
referencing a removed outer.

> Correctness guard for the future: when additional shadows are added (Wave 1: more `StreamExec*`),
> each new shadow needs its own `<exclude>` line AND a re-check for inner classes via
> `unzip -l <flink-planner.jar> | grep '<FQCN>'`. Several stock `StreamExec*` DO have inner classes
> (not verified here beyond Calc) — the structural test in the verification scope should assert "exactly
> one entry per shadowed FQCN" to catch a forgotten inner-class exclude.

---

## Bundled ALv2 components (for NOTICE)

Two distinct lists — do not confuse them:

### (A) What is ACTUALLY embedded in the planner jar bytes (this is what NOTICE must attribute)
Verified by listing embedded `.class` package roots in `flink-table-planner_2.12-1.18.1.jar`:
- `org/apache/calcite/...` (Calcite core + linq4j + avatica)
- `org/apache/flink/...` (Flink planner + sql-parser + calcite-bridge code)
- `org/locationtech/jts/...` (JTS core)
- `org/apache/flink/calcite/shaded/com/google/common/...` — **Guava is relocated/embedded here**
  (verified: `org/apache/flink/calcite/shaded/com/google/common/base/Ascii.class` etc.)

The planner jar's **own `META-INF/NOTICE`** (read in full) is therefore the authoritative
embedded-content attribution. It declares the bundled ALv2 deps as:
- `com.google.guava:guava:31.1-jre`
- `com.google.guava:failureaccess:1.0.1`
- `org.apache.calcite:calcite-core:1.32.0`
- `org.apache.calcite:calcite-linq4j:1.32.0`
- `org.apache.calcite.avatica:avatica-core:1.22.0`
- `commons-codec:commons-codec:1.15`
- `commons-io:commons-io:2.11.0`

Plus non-ALv2 embedded deps (NOTICE must keep these license callouts too):
- MIT: `org.checkerframework:checker-qual:3.12.0`
- EPL v2.0: `org.locationtech.jts:jts-core:1.19.0`

And the NOTICE's own ASF copyright blocks for: Apache Flink, flink-table-planner, Flink SQL Parser,
Flink Calcite Bridge, Apache Calcite, Apache Calcite Avatica, Apache Commons Codec, Apache Commons IO.

➡️ **NOTICE recommendation:** the shaded jar's `META-INF/NOTICE` must reproduce the planner jar's
NOTICE content (above) — `ApacheNoticeResourceTransformer` aggregates input NOTICE files toward this,
but the final text is a release-compliance review item (per research). Add Auron's own ASF NOTICE
header block AND a "this product includes modified Apache Flink source (StreamExecCalc), modified by
the Apache Auron project" statement (ALv2 §4(b) — modified-file attribution). Do NOT stamp Auron's
ALv2 source header onto the modified Flink file's package.

### (B) The planner jar's `META-INF/DEPENDENCIES` (full BUILD closure — NOT all embedded)
`DEPENDENCIES` (read in full) lists ~70 transitive artifacts including jackson, netty, zookeeper,
scala-compiler, kryo, snappy, asm, javassist, all the flink-* modules, etc. **Most of these are NOT
embedded** in the planner jar bytes (they are `provided`/`flink-dist`-supplied — confirmed: only
calcite/flink/jts/relocated-guava class roots are present). Do **not** copy the whole DEPENDENCIES
list into the NOTICE; that would over-attribute. The embedded set in (A) is the correct NOTICE basis.

> If Q2-B (runtime bundled) is chosen, the NOTICE must ALSO attribute whatever `auron-flink-runtime`
> and `auron-core` embed (arrow, protobuf, `libauron.so` etc.) — out of scope for this planner-only
> investigation; flag for the NOTICE agent under Q2-B.

---

## Verified Assumptions
- `maven.plugin.shade.version` = **3.5.2** — `pom.xml:73` (cited in research, used by the working
  assembly config).
- `auron-flink-planner` → `flink-table-planner_2.12:1.18.1` at **compile scope** —
  `auron-flink-planner/pom.xml:130-134` (no `<scope>`). Verified by direct read.
- Non-loader `flink-table-planner_2.12-1.18.1.jar` present in local m2; **11101 files**, single
  `StreamExecCalc.class` (4185 B), no inner classes. Verified by `unzip -l`.
- Planner jar ships exactly one SPI file: `META-INF/services/org.apache.flink.table.factories.Factory`
  (3 entries: `DefaultExecutorFactory`, `DefaultParserFactory`, `DefaultPlannerFactory`). Verified by
  extract + cat.
- Planner jar embeds class bytes only under `org/apache/calcite`, `org/apache/flink`,
  `org/locationtech/jts` (Guava relocated under `org/apache/flink/calcite/shaded/...`). Verified by
  package-root listing.
- `auron-flink-runtime` ships TWO main-resource SPI files (`Factory` → Kafka factory;
  `AuronAdaptorProvider` → `FlinkAuronAdaptorProvider`). Verified by `find` + read.
- `auron-flink-planner` ships **no** main-resource SPI files (only `src/test/resources/...`). Verified
  by `find` on `src/main`.
- The `<filter><artifact>...<excludes>` shape and `*:*` glob are proven valid in-repo —
  `auron-flink-assembly/pom.xml:65-73`.
- All four named transformers and `AppendingTransformer` are first-party shade transformers
  (`AppendingTransformer` proven in-repo at `auron-flink-assembly/pom.xml:60-64`).

## Compatibility Risks
- **SPI clobber under Q2-B**: if runtime is bundled and the shade does NOT merge `Factory` SPI files
  (e.g. uses a clobbering transformer or none), either Flink's `Default*Factory` entries or Auron's
  `AuronKafkaDynamicTableFactory` is lost → planner construction or Kafka table factory breaks. Mitigation:
  `ServicesResourceTransformer` (in sketch). Note the existing assembly uses `AppendingTransformer`
  bound to the **legacy** `org.apache.flink.table.factories.TableFactory` path
  (`auron-flink-assembly/pom.xml:60-64`); the planner jar ships the **modern** `Factory` SPI, so the
  legacy AppendingTransformer alone would NOT preserve the modern `Factory` SPI. Use
  `ServicesResourceTransformer` (general) instead of / in addition to the legacy AppendingTransformer.
- **Version-property drift**: filter `<artifact>` must track `scala.binary.version` (`2.12`) and the
  same planner coordinate the dep resolves to. A hardcoded `_2.12` is fine while only 1.18/2.12 exists
  (research: defer multi-version re-shading).
- **`createDependencyReducedPom` default is `true`** — for a terminal redistributable, set `false` to
  avoid emitting a reduced pom. Not a correctness risk, but cleaner.
- **Signature re-bundling**: keep the `META-INF/*.SF|*.DSA|*.RSA` exclusion (carried from the
  assembly) — re-bundling signed-jar signatures into a fat jar breaks verification.
- **Future shadows (Wave 1)**: adding `StreamExec*` shadows requires one `<exclude>` per FQCN AND an
  inner-class audit; some stock `StreamExec*` classes DO carry `$N` inner classes (not the case for
  Calc). The single-Calc filter is correct today but not a template that auto-extends.

## Edge Cases
- **`StreamExecCalc$1.class` and friends do not exist** for Calc (stock or Auron) — the single-exclude
  filter is complete. (If they DID exist on the stock side and were left in, they would reference a
  removed outer and fail verification/load — hence the future-shadow guard above.)
- **`LICENSE.txt` vs `LICENSE`**: planner jar ships both `META-INF/LICENSE` (11358 B) and
  `META-INF/LICENSE.txt` (1126 B). `ApacheLicenseResourceTransformer` normalizes `LICENSE`;
  `LICENSE.txt` may pass through untouched — verify the shaded jar's `META-INF` for duplicate/garbled
  license files post-build.
- **`org.apache.flink.sql.parser.utils` appears as a top-level "dir"** in the jar listing (a
  package-named resource path, not a package collision) — harmless, no action.
- **Reproducibility**: shade honors `project.build.outputTimestamp` for deterministic zip timestamps —
  optional but recommended for a redistributable ASF artifact (research-framework noted `outputTimestamp`).

## Verification command evidence (read-only, this session)
- `unzip -l flink-table-planner_2.12-1.18.1.jar` → `11101 files`; single `StreamExecCalc.class`; SPI =
  `META-INF/services/org.apache.flink.table.factories.Factory`; metadata = NOTICE/LICENSE/DEPENDENCIES/
  LICENSE.txt/MANIFEST.
- Extract + cat of `META-INF/NOTICE` and the `Factory` SPI → contents reproduced above.
- Package-root listing → embedded bytes only `org/apache/calcite`, `org/apache/flink`,
  `org/locationtech/jts`; Guava relocated under `org/apache/flink/calcite/shaded/...`.
- `find .../src/main/resources/META-INF/services` → runtime has `Factory` + `AuronAdaptorProvider`;
  planner has none in `src/main`.
- Full read of `auron-flink-planner/pom.xml`, `auron-flink-assembly/pom.xml`, and
  `StreamExecCalc.java`.

## Gaps / not verified this session
- Did NOT run `mvn dependency:tree` (the embedded-bytes inspection + the jar's own NOTICE/DEPENDENCIES
  give a more accurate embedded list than the build tree would, and avoids a slow network build).
- 1.18.1 **loader** jar internals not re-confirmed here (only 1.16.1 loader is in local m2) — already
  covered by research; not needed for the shade config (shade input is the non-loader jar).
- Final NOTICE wording is a release-compliance review item, not auto-derivable — handed to the NOTICE
  agent with the embedded list above.
