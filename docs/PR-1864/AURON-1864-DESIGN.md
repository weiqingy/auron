# Design — AURON #1864: Convert Cast operators to Auron Native operators

Base: master `a0e2b5f7`. Flink 1.18.1 / Calcite bundled (pre-1.35). Wave-1 converter work.

## Problem Statement

The Flink converter already converts explicit `CAST`, but routes it to the **try-cast** native node
(`RexCallConverter.java:195-196` → `buildTryCast` → `PhysicalTryCastNode`), which returns **NULL** on a
bad conversion. Flink SQL `CAST(x AS T)` is **strict** — it must *throw* on overflow / unparseable
input. So native acceleration silently changes query semantics: `CAST('abc' AS INT)` yields NULL
natively where Flink would error. Separately, `isSupported` accepts *every* CAST
(`RexCallConverter.java:133` falls through to `return true`) with no source→target validation, while the
native cast kernels are Spark-tuned — so temporal/complex casts risk silent semantic divergence. There
is also no handling of explicit `TRY_CAST`. This issue makes cast conversion semantically faithful and
safe.

## Approach Candidates

### Approach A: Strict/try split + conservative type gating (CHOSEN)
Route explicit Flink `CAST` → strict `PhysicalCastNode`; explicit `TRY_CAST` (matched by operator
identity) → `PhysicalTryCastNode`; add a source→target type gate so only conversions the native kernel
performs faithfully convert, everything else falls back to Flink's engine. Internal promotion sites stay
on the shared `wrapInTryCast`.
- **Pros**: Flink-faithful error semantics; bounded blast radius (only the explicit-CAST path changes);
  safe-by-construction (unknown conversions fall back, never miscompute); no proto/dep change.
- **Cons**: Diverges from the Spark prior art, which uses try-cast for user CAST (justified below);
  conservative gate may fall back some casts the kernel could actually handle (acceptable — correctness
  over coverage; widen later).
- **Prior art**: native strict path `planner.rs:937-946` (`cast=10`→DataFusion `CastExpr`,
  `safe:false`→errors); Spark gating precedent `NativeConverters.scala:477-486`.

### Approach B: Keep try-cast for everything (status quo)
Leave the merged behavior; optionally add explicit `TRY_CAST` as another try-node.
- **Pros**: zero risk to siblings; smallest diff.
- **Cons**: leaves the correctness bug (strict CAST silently nulls); does not satisfy "faithful cast
  support." Rejected.

### Approach C: Strict for CAST, no type gate
Route CAST→strict, TRY_CAST→try, but emit for *all* type pairs and trust the kernel.
- **Pros**: maximal coverage, simplest predicate.
- **Cons**: strict `CastExpr` *throws* at runtime on a pair the kernel mishandles → a Flink query that
  should succeed instead fails in native code; temporal casts diverge from Flink semantics silently.
  Rejected — unsafe.

## Decision

**Approach A.** Native acceleration must preserve Flink query semantics. Mapping strict `CAST` to a
null-returning node is a latent correctness bug; mapping it to a strict node that can throw on
unsupported pairs is a latent availability bug. The type gate resolves both: faithful conversions go
native (strict or try as the user wrote), everything else falls back. The change is confined to the
explicit-CAST dispatch and a new sibling helper — the shared `wrapInTryCast`/`castIfNecessary` used by
the 6 internal widening-promotion sites (#1859/#1860/#1861) are untouched, so try-cast (correct for
never-failing widenings) is preserved there.

### Justified deviation from Spark prior art
Spark's `NativeConverters.scala:476-506` maps user `Cast` → `setTryCast` (null-on-failure). We map user
`CAST` → strict `setCast`. This is faithful, not contradictory: **Spark's default (non-ANSI) `CAST`
itself returns NULL on failure**, so try-cast is the correct Spark mapping; **Flink's `CAST` throws**,
so strict-cast is the correct Flink mapping. Each side faithfully mirrors its own engine; the proto-node
choice differs because the engine semantics differ. Flink's null-returning form is `TRY_CAST`, which we
map to the try node — exactly Spark's node for its null-returning default.

## Detailed Design

### 1. Dispatch (`RexCallConverter.convert`)
TRY_CAST is `SqlKind.OTHER_FUNCTION` — it cannot be a `switch` case. Add an operator-identity branch
before `switch(kind)` (precedent: LIKE's `instanceof` handling), and repoint `case CAST`:

```java
// before switch(kind), mirroring how TRY_CAST escapes the kind-based switch
if (call.getOperator() == FlinkSqlOperatorTable.TRY_CAST) {
    return buildTryCast(call, context);   // existing helper, now reserved for explicit TRY_CAST
}
switch (kind) {
    ...
    case CAST:
        return buildCast(call, context);  // was buildTryCast — now strict
    ...
}
```

### 2. Strict builder (`RexCallConverter.buildCast`)
Mirrors `buildTryCast` (`:325-328`) but emits the strict node:

```java
private PhysicalExprNode buildCast(RexCall call, ConverterContext context) {
    PhysicalExprNode operand = convert(call.getOperands().get(0), context);
    return FlinkNodeConverterUtils.wrapInCast(operand, call.getType());
}
```

### 3. Strict helper (`FlinkNodeConverterUtils.wrapInCast`)
Byte-identical to `wrapInTryCast` (`:115-121`) except the node setter — same Arrow type stamping
(`FlinkTypeFactory.toLogicalType` → `SchemaConverters.convertToAuronArrowType`):

```java
public static PhysicalExprNode wrapInCast(PhysicalExprNode expr, RelDataType targetType) {
    ArrowType arrowType = toArrowType(targetType);            // same stamping as wrapInTryCast
    return PhysicalExprNode.newBuilder()
        .setCast(PhysicalCastNode.newBuilder().setExpr(expr).setArrowType(arrowType).build())
        .build();
}
```
`wrapInTryCast`/`castIfNecessary` are **unchanged**.

### 4. Support gate (`RexCallConverter.isSupported`)
Two additions: accept TRY_CAST by operator identity (its kind isn't in `SUPPORTED_KINDS`), and gate
CAST/TRY_CAST source→target pairs. Both explicit cast forms share the same type gate.

```java
// accept TRY_CAST before the SUPPORTED_KINDS reject
if (call.getOperator() == FlinkSqlOperatorTable.TRY_CAST) {
    return isCastTypeSupported(call.getOperands().get(0).getType(), call.getType());
}
// inside the kind path, where CAST currently returns true unconditionally:
if (kind == SqlKind.CAST) {
    return isCastTypeSupported(call.getOperands().get(0).getType(), call.getType());
}
```

```java
private static boolean isCastTypeSupported(RelDataType src, RelDataType tgt) {
    if (SqlTypeUtil.isNumeric(src) && SqlTypeUtil.isNumeric(tgt)) return true; // numeric↔numeric (incl decimal)
    if (SqlTypeUtil.isNumeric(src) && SqlTypeUtil.isString(tgt))  return true; // numeric→string
    if (SqlTypeUtil.isString(src)  && SqlTypeUtil.isNumeric(tgt))  return true; // string→numeric (incl →decimal)
    if (SqlTypeUtil.isBoolean(src) && SqlTypeUtil.isString(tgt))  return true; // boolean→string
    if (SqlTypeUtil.isString(src)  && SqlTypeUtil.isBoolean(tgt)) return true; // string→boolean
    return false;                                                              // else → fallback
}
```
`SqlTypeUtil.isNumeric` = exact||approximate numeric ⇒ DATE/TIME/TIMESTAMP excluded automatically;
complex types (ARRAY/MAP/ROW) are neither numeric/string/boolean ⇒ excluded. (`SqlTypeUtil` is already
imported and used in the module.)

### Data flow
```
RexCall CAST(col AS T)      ── kind==CAST ──▶ buildCast ─▶ wrapInCast  ─▶ PhysicalCastNode(strict, errors)
RexCall TRY_CAST(col AS T)  ── op identity ─▶ buildTryCast ─▶ wrapInTryCast ─▶ PhysicalTryCastNode(null)
unsupported src→tgt pair    ── isSupported=false ─▶ whole Calc falls back to Flink engine
internal widening promotion ── (unchanged) ──▶ wrapInTryCast / castIfNecessary  ─▶ PhysicalTryCastNode
```

## Prior Art Comparison

| Aspect | This Design (Flink) | Spark `NativeConverters` | Native kernel |
|--------|---------------------|--------------------------|---------------|
| User CAST | strict `setCast` (throws) | `setTryCast` (Spark default nulls) | `cast=10`→`CastExpr safe:false` |
| User TRY_CAST | `setTryCast` (null) | n/a (Spark CAST already nulls) | `try_cast=15`→`TryCastExpr safe:true` |
| Internal promotion | `setTryCast` (widening, never fails) | `setCast` (decimal overflow wrap) | — |
| Type gating | numeric/string/boolean gate → fallback | emit-all except date/timestamp→UDF wrapper | broad (arrow + Spark arms) |
| Unsupported escape | full-Calc fallback (no Flink UDF path) | Spark-UDF wrapper | — |

## Alignment with AIP-1 (Flink-on-Auron Phase 1)

| Element | Status | Evidence |
|---------|--------|----------|
| Interception at `StreamExecCalc` Rex→native conversion | Aligned | `RexCallConverter` is the established Calc converter; #1864 extends its CAST arm only |
| Reuse existing converter framework (no new dispatch layer) | Aligned | Adds one operator-identity branch + one builder + one helper; mirrors merged #1859/#1860/#1861 shape |
| Native plan via `PhysicalPlanNode` protobuf | Aligned | Reuses existing `PhysicalCastNode`/`PhysicalTryCastNode`; no proto change |
| Fail-back to Flink engine on unsupported nodes | Aligned (extended) | Type gate routes unsupported casts through the same `isSupported=false` fallback path used for unsupported RexCalls (#1853 `FAIL_BACK_FLINK_ENGINE_ENABLED`) |
| Spark converter as second source of truth | Follow actual code (justified deviation) | Spark uses try-cast for user CAST; we use strict because Flink CAST throws while Spark CAST nulls — documented above |

No element conflicts with the AIP direction.

## Dependencies

None new. `flink-table-planner_2.12` already compile-scope (`auron-flink-planner/pom.xml:129-134`);
proto nodes already generated. New imports: `FlinkSqlOperatorTable`, `PhysicalCastNode`, `SqlTypeUtil`
(if not already imported in the touched file).

## Test Strategy

- **Unit (`RexCallConverterTest`)**: update `testConvertCast` to assert strict `hasCast()`; add a
  TRY_CAST→`hasTryCast()` test; add gated-fallback tests (e.g. `CAST(int AS DATE)` → `isSupported`
  false); add a supported-pair sweep (numeric→string, string→numeric, boolean→string,
  string→decimal). One concern per test.
- **End-to-end (`AuronFlinkCalcITCase`)**: SQL `SELECT CAST(int AS DOUBLE)`, `CAST(string AS INT)` over
  the `T1` fixture asserting the native row-set; a `TRY_CAST` query; a query whose cast falls back (e.g.
  to a temporal type) to confirm correct results via fallback. Row-set assertions only (harness can't
  observe native-vs-fallback) — mirrors #1860/#1861.
- **Verify-phase check**: dump `StreamExecCalc` RexNodes for `TRY_CAST(col AS INT)` to confirm the
  operator survives to the converter intact (static risk LOW).

## Out of Scope

- Widening the supported-type matrix to temporal / complex / decimal-precision-narrowing casts (future
  follow-up; conservative gate ships first for correctness).
- Changing shared `wrapInTryCast`/`castIfNecessary` semantics or the 6 internal promotion sites.
- `castIfNecessary`'s SqlTypeName-only comparison (pre-existing; not on the explicit-CAST path).
- Any proto / native-engine change.

## Alternatives Considered

- **Name-string match for TRY_CAST** (`getName().equals("TRY_CAST")`) — rejected; operator-identity
  `==` is exact and the singleton is on the compile classpath, so the string match buys nothing.
- **Hand-rolled type classification** — rejected; reuse Calcite `SqlTypeUtil` predicates already in the
  module.
