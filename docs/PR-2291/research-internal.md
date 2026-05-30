# Research (in-repo build/packaging) — AURON #2291

Scope: map exactly how the Flink extension (and Spark extension, as prior art) is built
and packaged today, to inform the A2 shaded-`flink-table-planner` jar. Every claim is
anchored to `path:line`.

## Proven Patterns

### P1. The current "A1 overlay jar" is `auron-flink-assembly` — an uber jar bundling BOTH planner + runtime
- `auron-flink-extension/auron-flink-assembly/pom.xml:29-40` declares exactly two
  dependencies: `auron-flink-runtime` and `auron-flink-planner`.
- It runs `maven-shade-plugin` (`auron-flink-assembly/pom.xml:44-84`) bound to `package`
  (`:81`), goal `shade` (`:79`). No `<finalName>`, no `<relocations>`, no
  `<shadedArtifactAttached>`, no `<minimizeJar>` (confirmed: grep for those keys returns
  nothing). So it **replaces the main artifact** with the uber jar at the default name
  **`auron-flink-assembly-${project.version}.jar`** (e.g. `auron-flink-assembly-<ver>.jar`).
- artifactSet excludes (`auron-flink-assembly/pom.xml:49-58`): `org.apache.hadoop:*`,
  `org.slf4j:slf4j-api`, `slf4j-log4j12`, `slf4j-simple`, `slf4j-jdk14`,
  `org.apache.flink:flink-cep`. **Everything else on the compile/runtime classpath of
  the two deps gets bundled** — including `flink-table-planner_2.12` (which is a
  *compile-scope* dep of the planner module; see P3) and the native `.so` (via the
  `auron-core` → runtime chain; see P5).
- ServiceLoader merge: one `AppendingTransformer` for
  `META-INF/services/org.apache.flink.table.factories.TableFactory`
  (`auron-flink-assembly/pom.xml:60-64`). Note: the runtime ships the **newer**
  SPI file `META-INF/services/org.apache.flink.table.factories.Factory`
  (`auron-flink-runtime/src/main/resources/META-INF/services/org.apache.flink.table.factories.Factory`)
  plus `org.apache.auron.jni.AuronAdaptorProvider` — neither is in the transformer list,
  so today only the legacy `TableFactory` SPI is being merge-appended. (Risk noted in U-block.)
- signature stripping filter: `META-INF/*.SF|*.DSA|*.RSA` (`:65-73`).

This `auron-flink-assembly` IS the A1 overlay jar described in DESIGN Rev 4. It is meant
to be dropped on the classpath **ahead of** `flink-table-planner.jar` so the shadowed
`StreamExecCalc` FQCN resolves to Auron's copy (see the shadow class javadoc,
`StreamExecCalc.java:59-78`, which explicitly states "with `auron-flink-planner` ahead
of `flink-table-planner` on the classpath").

### P2. Module wiring
- `auron-flink-extension/pom.xml:32-36` `<modules>`: `auron-flink-planner`,
  `auron-flink-runtime`, `auron-flink-assembly` (packaging `pom`, `:29`).
- Parent of the aggregator is `auron-parent_${scalaVersion}` (`auron-flink-extension/pom.xml:21-26`).
- A new `auron-flink-planner-shaded` module (per context.md decision #3) must be added
  to this `<modules>` list.

### P3. `auron-flink-planner` depends on `flink-table-planner` at **compile scope** (the one Flink dep that gets bundled)
- `auron-flink-planner/pom.xml:130-134`:
  ```xml
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  ```
  **No `<scope>` ⇒ compile** — deliberately so the assembly bundles the planner content.
  This is the single most load-bearing fact for A2: the shaded jar must bundle this same
  `flink-table-planner_2.12:1.18.1` content, with Auron's same-FQCN `StreamExecCalc`
  winning over the stock one.
- All *other* Flink deps in the planner are `provided` or `test`:
  `flink-table-api-java-bridge` provided (`:105-110`), `flink-scala_2.12` provided
  (`:114-119`), `flink-table-runtime` provided (`:137-142`), `flink-table-calcite-bridge`
  provided + optional, with `calcite-core` excluded (`:144-157`), `flink-core` provided
  (`:200-205`). Test-scoped: `flink-table-planner` test-jar (`:160-166`),
  `flink-test-utils` (`:168-183`), api-scala/bridge (`:185-198`), several test-jars
  (`:207-244`), hadoop client (`:246-257`), `flink-connector-files` (`:259-265`).
- Non-Flink compile deps: arrow-c-data, arrow-compression, arrow-memory-unsafe,
  arrow-vector (`auron-flink-planner/pom.xml:274-293`), plus `auron-flink-runtime`
  (`:64-68`). janino/immutables/checker-qual are provided.
- Auron's planner main classes (`auron-flink-planner/src/main/java/...`):
  - The shadow: `org/apache/flink/table/planner/plan/nodes/exec/stream/StreamExecCalc.java`
    (FQCN `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc`).
  - `org/apache/auron/flink/table/planner/UnsupportedFlinkNodeRecorder.java`.
  - converters: `converter/{ConverterContext,FlinkAggCallConverter,FlinkNodeConverter,
    FlinkNodeConverterFactory,FlinkNodeConverterUtils,FlinkRexNodeConverter,RexCallConverter,
    RexInputRefConverter,RexLiteralConverter}.java`.
- `flink.markBundledAsOptional=false` (`auron-flink-planner/pom.xml:30`) and
  `scala.binary.version=2.12` (`:33`) are local properties this module relies on.

### P4. `auron-flink-runtime` holds the operators, FFI, native-loader bridge, Kafka connector
- Main classes (`auron-flink-runtime/src/main/java/...`):
  - operators: `org/apache/auron/flink/runtime/operator/{FlinkAuronCalcOperator,
    FlinkAuronFunction,FlinkAuronOperator,SupportsAuronNative}.java`.
  - FFI / Arrow bridge: `org/apache/auron/flink/arrow/{FlinkArrowFFIExporter,FlinkArrowReader,
    FlinkArrowUtils,FlinkArrowWriter}.java` + `arrow/vectors/*` + `arrow/writers/*`.
  - native-loader bridge: `org/apache/auron/jni/FlinkAuronAdaptor.java` +
    `FlinkAuronAdaptorProvider.java`; SPI file
    `src/main/resources/META-INF/services/org.apache.auron.jni.AuronAdaptorProvider`.
  - Kafka connector: `org/apache/auron/flink/connector/kafka/{AuronKafkaDynamicTableFactory,
    AuronKafkaDynamicTableSource,AuronKafkaSourceFunction,KafkaConstants}.java`; SPI file
    `src/main/resources/META-INF/services/org.apache.flink.table.factories.Factory`.
  - config: `org/apache/auron/flink/configuration/FlinkAuronConfiguration.java`
    (holds `FAIL_BACK_FLINK_ENGINE_ENABLED`, referenced by the shadow at
    `StreamExecCalc.java:69`).
  - `org/apache/auron/flink/metric/FlinkMetricNode.java`,
    `org/apache/auron/flink/table/data/AuronColumnarRowData.java`,
    `org/apache/auron/flink/utils/SchemaConverters.java`.
- Deps (`auron-flink-runtime/pom.xml:32-105`): compile = `auron-core` (`:34-38`),
  `proto` (`:39-43`), arrow-c-data/-memory-unsafe/-vector (`:45-57`), `kafka-clients`
  3.9.2 (`:85-89`). All Flink deps are `provided`: `flink-table-common` (`:58-63`),
  `flink-streaming-java` (`:64-69`), `flink-table-api-java-bridge` (`:71-76`),
  `flink-table-runtime` (`:77-82`).
- **`auron-flink-runtime` carries NO shade plugin** — it is a plain `jar` module.

### P5. The native `.so`/`.dylib` ships via the `auron-core` dependency, NOT a Flink-specific step
- The native lib is added as a **resource** of `auron-core`:
  `auron-core/pom.xml:89-98` declares `<directory>../native-engine/_build/${releaseMode}</directory>`
  including `libauron.so`, `libauron.dylib`, `auron.dll`. So these land inside
  `auron-core-<ver>.jar` at the jar root.
- `auron-flink-runtime` depends on `auron-core` (`auron-flink-runtime/pom.xml:34-38`),
  so the native lib transitively reaches anything that bundles the runtime.
- The native lib is built by `auron-core`'s `build-native` profile
  (`auron-core/pom.xml:117-154`, auto-active unless `-DskipBuildNative`), which runs
  `dev/mvn-build-helper/build-native.sh` (`auron-core/pom.xml:135`; libname `libauron`
  at `build-native.sh:34`, cache dir `native-engine/_build/$profile` at `:47`).
- Loading is uniform JVM-side: both `FlinkAuronAdaptor.loadAuronLib()`
  (`auron-flink-runtime/.../jni/FlinkAuronAdaptor.java:35-46`) and
  `SparkAuronAdaptor.loadAuronLib()`
  (`spark-extension/.../jni/SparkAuronAdaptor.java:42-49`) do
  `getResourceAsStream(System.mapLibraryName("auron"))` → temp file → `System.load(...)`.
  i.e. the `.so` must be a **classpath resource** at runtime. Whatever jar carries the
  runtime + auron-core content must therefore also carry `libauron.so`.

  **Implication for the A2 open question (runtime placement):** because the native lib
  and operators travel with `auron-core`+`auron-flink-runtime`, the runtime portion is a
  *separate concern* from the planner-shadow shading. Two viable shapes for A2:
  (a) keep shipping the runtime as the existing `auron-flink-assembly` uber jar in
  `$FLINK_HOME/lib/` and have the new `*-planner-shaded.jar` bundle ONLY the
  flink-table-planner content + planner shadows/converters; or
  (b) fold runtime + auron-core (+ native `.so`) into the single shaded planner jar.
  The current assembly already does (b)-style bundling (it lists both runtime and
  planner). Decision is for the design phase; the evidence above is the input.

### P6. Spark prior art for shading/relocation — `dev/mvn-build-helper/assembly`
- This is the Spark uber-jar producer: `dev/mvn-build-helper/assembly/pom.xml`,
  artifactId `auron-${shimName}_${scalaVersion}` (`:27`), packaging `jar`.
- It depends on `spark-extension_${scalaVersion}` + `spark-extension-shims-spark_${scalaVersion}`
  + netty-buffer/-common (`:36-57`).
- `maven-shade-plugin` (`:62-150`): `<finalName>auron-${shimName}_${scalaVersion}-${project.version}</finalName>`
  (`:67`), `createDependencyReducedPom=false` (`:76`).
- artifactSet excludes (`:77-90`): commons-*, jackson-*, log4j-*, slf4j-*, zstd-jni,
  scala-library, scala-reflect.
- filters (`:91-105`): strips commons/slf4j classes + `META-INF/*.SF|DSA|RSA` +
  `module-info.class` + `*.semanticdb`.
- **`<relocations>` (`:106-143`)** — the precedent for relocating bundled third-party
  content: `com.google.flatbuffers`, `com.google.protobuf`, `org.apache.arrow`
  (**excluding `org.apache.arrow.c.**`** because the C ABI refers to original FQCNs —
  `:118-121`), `io.netty`, `javax.annotation`, `net.bytebuddy`, `org.checkerframework`,
  `scala.compat`, all under prefix `${auron.shade.packageName}` = `auron` (`:33`).
- Note: this Spark assembly does NOT relocate `org.apache.spark.*` or
  `org.apache.flink.*` — it relocates only *third-party* libs to avoid clashing with the
  host engine's own copies. **The A2 jar must do the opposite for the planner**: it must
  KEEP `org.apache.flink.table.planner.*` at original FQCNs (the whole mechanism depends
  on FQCN identity), so relocation is the wrong tool for the planner content; it is only
  relevant for the arrow/protobuf/netty deps if those are bundled.
- `os-maven-plugin` (`:173-180`) provides `os.detected.classifier` (linux-x86_64,
  osx-x86_64) used to name the released jar
  `auron-${shimName}_..-${releaseMode}-${os.detected.classifier}-${ver}.jar` (`:165-166`)
  via an `exec-maven-plugin` cp step (`:151-171`). This is how a *native-bearing* jar is
  given an OS-classifier name — relevant if A2 bundles the `.so`.

## API Surface  (pom structure, plugin configs, artifact names — file:line)

| Element | Evidence |
|---|---|
| Flink aggregator modules | `auron-flink-extension/pom.xml:32-36` (planner, runtime, assembly) |
| A1 overlay artifact name | default `auron-flink-assembly-${project.version}.jar` (no `<finalName>` in `auron-flink-assembly/pom.xml`) |
| A1 shade plugin | `auron-flink-assembly/pom.xml:44-84`, version `${maven.plugin.shade.version}` = `3.5.2` (`pom.xml:73`) |
| A1 bundled deps | runtime + planner (`auron-flink-assembly/pom.xml:29-40`); excludes hadoop/slf4j/flink-cep (`:49-58`) |
| planner→flink-table-planner scope | **compile** (`auron-flink-planner/pom.xml:130-134`) |
| shadow class FQCN | `org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecCalc` (`StreamExecCalc.java:17,90`) |
| shadow mechanism doc | `StreamExecCalc.java:59-84` (classpath-order FQCN resolution; activation INFO log; fallback WARN) |
| `flink.version` | `1.18.1`, set only in profile `flink-1.18` (`pom.xml:1350-1357`) |
| shade plugin version | `maven.plugin.shade.version` = `3.5.2` (`pom.xml:73`) |
| surefire version | `maven.plugin.surefire.version` = `3.5.3` (`pom.xml:72`) |
| scala | `scalaVersion=2.12`, `scalaLongVersion=2.12.18` (`pom.xml:60-61`); planner local `scala.binary.version=2.12` (`auron-flink-planner/pom.xml:33`) |
| hadoop | `hadoopVersion=3.4.3` (`pom.xml:53`) |
| junit | `junit.jupiter.version=5.13.4` (`pom.xml:88`) |
| native lib resource | `auron-core/pom.xml:89-98` (`libauron.so/.dylib`, `auron.dll`) |
| native build hook | `auron-core/pom.xml:117-154` → `dev/mvn-build-helper/build-native.sh` |
| native loader (Flink) | `FlinkAuronAdaptor.java:35-46` (`getResourceAsStream(mapLibraryName("auron"))`) |
| Spark uber/shade prior art | `dev/mvn-build-helper/assembly/pom.xml:62-150` (relocations `:106-143`) |
| Spark OS-classifier naming | `dev/mvn-build-helper/assembly/pom.xml:151-180` |
| Flink CI workflow | `.github/workflows/flink.yml` (matrix flinkver 1.18, module `auron-flink-extension/auron-flink-planner` `:45`, build cmd `:60`) |
| license/RAT CI | `.github/workflows/license.yml:48-54` (runs `apache-rat-plugin:check` with `-Pflink,flink-1.18` `:54`) |
| root NOTICE | `NOTICE` (6 lines, generic ASF boilerplate only — no third-party attributions yet) |
| root LICENSE | `/Users/wiyang/workspace/auron/LICENSE` |

## Blockers / Incompatibilities

- **B1. CI does NOT build or test the assembly/shaded jar.** `flink.yml:45` builds only
  `-pl auron-flink-extension/auron-flink-planner -am` (the `-am` at `:60` pulls upstream
  deps including runtime+core, but **not** the downstream `auron-flink-assembly`). So
  today's A1 overlay jar is **never exercised in CI**, and any A2 shaded-jar structural
  smoke assertion (context.md decision #2) needs a NEW CI hook (either add the new module
  to the existing `flink.yml` matrix/`-pl`, or attach the assertion to the module's own
  build via a verify-phase plugin so `-am` reachability isn't required). This is the
  concrete place the "build-time structural assertion" must be wired.

- **B2. The shaded planner jar must NOT relocate `org.apache.flink.table.planner.*`.**
  The whole shadow mechanism is FQCN-identity based (`StreamExecCalc.java:59-63`).
  Relocation (the Spark assembly's tool, `dev/mvn-build-helper/assembly/pom.xml:106-143`)
  would break it. The Spark precedent is useful only for *third-party* deps
  (arrow/protobuf/netty), and even there `org.apache.arrow.c.**` is excluded because the
  C ABI needs original names (`:118-121`) — directly relevant if A2 bundles arrow + the
  native `.so`.

## Unknowns / Risks

- **U1. Duplicate `StreamExecCalc` resolution inside one fat jar.** A2's premise
  (context.md L16-18) is "exactly one class per shadowed FQCN ⇒ structural activation."
  But `flink-table-planner_2.12:1.18.1` (bundled at compile scope, P3) contains the
  *stock* `StreamExecCalc.class`, and `auron-flink-planner` contains Auron's. The shade
  plugin must guarantee Auron's wins. maven-shade resolves same-path duplicates by the
  **project's own classes taking precedence over dependency classes**, but when both come
  from *dependencies* (planner-shaded depends on auron-flink-planner AND
  flink-table-planner), ordering is dependency-declaration-order dependent and may emit a
  "we have a duplicate" warning. The structural smoke assertion (decision #2) exists
  precisely to catch this — must verify the surviving `StreamExecCalc.class` bytes are
  Auron's (e.g. presence of the `ACTIVATION_LOGGED` field / `org.apache.auron` imports),
  not just that exactly one entry exists. Needs design-phase verification of shade
  duplicate-handling semantics.

- **U2. SPI / ServiceLoader merge coverage.** Today's assembly transformer only merges
  the legacy `org.apache.flink.table.factories.TableFactory` SPI
  (`auron-flink-assembly/pom.xml:60-64`), but the runtime actually ships the modern
  `org.apache.flink.table.factories.Factory` SPI plus
  `org.apache.auron.jni.AuronAdaptorProvider` (P4). If A2 bundles the runtime, the
  `AuronAdaptorProvider` SPI MUST survive (it is how `loadAuronLib` is wired,
  `AuronAdaptor.java:46-48`). Whether to add a `ServicesResourceTransformer` or
  additional `AppendingTransformer` entries is a design decision; risk is a silently
  dropped SPI → native lib never loads.

- **U3. Runtime placement (the explicit context.md open question).** Evidence (P5) shows
  the native `.so` + operators ride on `auron-core`+`auron-flink-runtime`. Whether A2's
  single shaded jar should also carry these, or whether the runtime ships as a second
  `lib/` jar, is unresolved by the build files alone — it is a deliberate design choice.
  Both shapes are mechanically supported by the existing dependency graph.

- **U4. `-Pflink` profile id does not exist in the root pom.** Only `flink-1.18`
  exists (`pom.xml:1350`; full profile-id list checked — no bare `flink`). Yet CI invokes
  `-Pflink,flink-1.18` (`license.yml:54`) and the captain memory/contribution notes refer
  to `-Pflink,flink-1.18`. Maven silently ignores unknown profile ids (only warns), so
  the build still works via `flink-1.18`. `auron-build.sh:467` correctly emits only
  `-Pflink-$FLINK_VER`. Not a blocker, but any new CI line for the shaded module should
  use `-Pflink-1.18` (the real id) to avoid the dead `flink` token.

- **U5. No NOTICE/LICENSE under `auron-flink-extension` or the assembly today.** Root
  `NOTICE` is bare ASF boilerplate (no embedded-third-party section). The RAT check
  (`license.yml:48-54`) validates *source header* presence, not bundled-content
  attribution. A2 embeds **modified** Flink planner content, so per context.md L41-42 a
  NOTICE/LICENSE addition is required (cf. `flink-shaded` / `spark-hadoop-cloud`
  precedent) — there is no in-repo precedent file to copy from; this is net-new.

## Dead Ends

- Searched for a dedicated native-lib-copy step inside the Flink modules or
  `auron-build.sh` — none exists. The native lib is exclusively wired through
  `auron-core/pom.xml:89-98`; there is no Flink-specific `.so` packaging. (Not a dead end
  for the research, just confirms P5: no separate path to chase.)
- No `<relocations>` anywhere except the Spark assembly
  (`dev/mvn-build-helper/assembly/pom.xml:106-143`); the Flink assembly has none. So
  there is no in-repo Flink relocation precedent — the Spark one is the only model.
