# Design — AURON #1861: Convert Comparison operators to Auron Native operators

## Problem Statement
Flink Calc expressions using comparison operators (`=`, `<>`, `>`, `<`, `>=`, `<=`) and `LIKE` cannot
yet execute on the Auron native engine — the Flink `RexCallConverter` recognizes only arithmetic and
`CAST`, so any Calc containing a comparison falls back to the Flink engine. Comparisons are the
backbone of `WHERE`/`JOIN`/`CASE` predicates, so without them most real queries can't be accelerated.
This is Wave-1 converter issue #1861, sibling to #1860 (logical) and #1864 (cast).

## Approach Candidates

### Approach A — Extend the existing `RexCallConverter` (chosen)
Add the comparison `SqlKind`s + `LIKE` to the existing converter's `SUPPORTED_KINDS` and `convert`
switch, reusing the established operand-promotion helpers.
- **Pros:** One converter keyed on `RexCall.class` (the factory rejects a second `RexCall` converter,
  per #1860 notes); reuses `getCommonTypeForComparison`/`castIfNecessary` already proven for
  arithmetic; mirrors how #1859/#1860 grew the same class; smallest diff; no proto/Rust/dep change.
- **Cons:** Sibling Wave-1 PRs touch the same `SUPPORTED_KINDS`/switch → trivial rebase conflicts.

### Approach B — Separate `ComparisonRexCallConverter`
A new converter class dedicated to comparisons.
- **Pros:** No rebase conflict with siblings.
- **Cons:** The factory keys on `RexCall.class` and rejects a second converter for the same node
  class (#1860) — would require reworking the dispatch model. Over-engineering for 7 operators that
  share the exact binary-expr machinery already in `RexCallConverter`. Rejected.

## Decision
**Approach A.** It matches the in-repo evolution of this class, reuses verified machinery, and keeps
the diff reviewable. The rebase-conflict cost is textual and expected across the Wave-1 series.

## Detailed Design

### Comparison operators → `PhysicalBinaryExprNode`
Six binary comparisons map to a single binary-expr node with a case-sensitive op string (source of
truth: native decoder `lib.rs:70-101`):

| SqlKind | op string |
|---|---|
| `EQUALS` | `"Eq"` |
| `NOT_EQUALS` | `"NotEq"` |
| `GREATER_THAN` | `"Gt"` |
| `LESS_THAN` | `"Lt"` |
| `GREATER_THAN_OR_EQUAL` | `"GtEq"` |
| `LESS_THAN_OR_EQUAL` | `"LtEq"` |

`buildComparison` is `buildBinaryExpr` **without the trailing output-cast** — operands are promoted to
a common type (so the native Arrow `apply_cmp` kernel sees matching types), but the result is already
BOOLEAN and needs no wrap:

```java
private PhysicalExprNode buildComparison(RexCall call, String op, ConverterContext context) {
    RexNode left = call.getOperands().get(0);
    RexNode right = call.getOperands().get(1);
    RelDataType compatibleType = FlinkNodeConverterUtils.getCommonTypeForComparison(
            left.getType(), right.getType(), FlinkNodeConverterUtils.TYPE_FACTORY);
    if (compatibleType == null) {
        throw new IllegalStateException("Incompatible types: "
                + left.getType().getSqlTypeName() + " and " + right.getType().getSqlTypeName());
    }
    PhysicalExprNode leftExpr = FlinkNodeConverterUtils.castIfNecessary(
            convertOperand(left, context), left.getType(), compatibleType);
    PhysicalExprNode rightExpr = FlinkNodeConverterUtils.castIfNecessary(
            convertOperand(right, context), right.getType(), compatibleType);
    return PhysicalExprNode.newBuilder()
            .setBinaryExpr(PhysicalBinaryExprNode.newBuilder()
                    .setL(leftExpr).setR(rightExpr).setOp(op))
            .build();
}
```

**Type-promotion rationale (D1):** the native comparison kernel requires matching operand types and
performs no implicit coercion (`binary.rs:405-415`); Calcite already inserts CASTs for mismatched
operands at validation, so the promotion is a no-op in the common case and a correctness fix
otherwise — never wrong. It also keeps comparison consistent with arithmetic in the same file. (Spark
trusts its analyzer and omits the cast; we deliberately keep the defensive promotion given the native
hard-requirement.)

**Incompatible-type edge case:** when `getCommonTypeForComparison` returns `null`, `buildComparison`
throws `IllegalStateException`, mirroring `buildBinaryExpr`. This path is unreachable for validated
Flink SQL (Calcite rejects incompatible comparisons at validation). A graceful `isSupported=false`
fallback is the one-line alternative if preferred.

### LIKE / NOT LIKE → `PhysicalLikeExprNode`
LIKE is **not** a binary op — it has a dedicated native node (`auron.proto:316-321`, `like_expr=20`):

```java
private PhysicalExprNode buildLike(RexCall call, ConverterContext context) {
    boolean negated = ((SqlLikeOperator) call.getOperator()).isNegated();
    PhysicalExprNode expr = convertOperand(call.getOperands().get(0), context);
    PhysicalExprNode pattern = convertOperand(call.getOperands().get(1), context);
    return PhysicalExprNode.newBuilder()
            .setLikeExpr(PhysicalLikeExprNode.newBuilder()
                    .setNegated(negated)
                    .setCaseInsensitive(false)
                    .setExpr(expr)
                    .setPattern(pattern))
            .build();
}
```

- `NOT LIKE` shares `SqlKind.LIKE`; the `negated` flag is read from `SqlLikeOperator.isNegated()` →
  a single negated node (not an outer `NOT`).
- `case_insensitive` is `false` (Flink SQL `LIKE` is case-sensitive; ILIKE out of scope).
- **ESCAPE fallback:** `isSupported` returns `false` when operand count ≠ 2 (3-operand explicit
  ESCAPE), because the proto has no escape field. The whole Calc then falls back to Flink — safe.

### `isSupported` / `SUPPORTED_KINDS`
Add the 6 comparison kinds + `LIKE` to `SUPPORTED_KINDS`. Comparisons stay **out** of
`BINARY_ARITHMETIC_KINDS` (no numeric-result guard). Add a LIKE guard:
```java
if (kind == SqlKind.LIKE) {
    return call.getOperator() instanceof SqlLikeOperator && call.getOperands().size() == 2;
}
```

### Test ripple
Six `StreamExecCalcTest` sites + one `RexCallConverterTest` site use `SqlStdOperatorTable.EQUALS` as a
*known-unsupported* fallback trigger. They switch to `SqlStdOperatorTable.SIMILAR_TO` (an unsupported
2-operand RexCall) so they keep testing the "unsupported RexCall → fallback" path. `UnregisteredRex`
is **not** used because `testFallbackEmitsDistinctWarnLogsForDistinctRexClasses` contrasts a RexCall
fallback against an UnregisteredRex one.

## Prior Art Comparison

| Aspect | This design | Spark (`NativeConverters`) | #1860 logical | Gluten-flink |
|---|---|---|---|---|
| Comparison node | `PhysicalBinaryExprNode` op `Eq/NotEq/…` | same op strings | same node (And/Or) | Velox fn names (n/a) |
| `<>` | first-class `NOT_EQUALS` → one `NotEq` | `Not(EqualTo)` special-case | — | only `decimal_notequalto` |
| Operand promotion | reuse `getCommonTypeForComparison` | trusts analyzer (no cast) | n/a (boolean operands) | promotes in-converter |
| LIKE | `PhysicalLikeExprNode`, `negated` from `isNegated()` | same node, default-escape assert | — | no LIKE |
| ESCAPE | unsupported → fallback | asserts default `\` | — | — |

## Alignment with AIP-1 (Flink-on-Auron PoC)

| Element | Classification | Evidence |
|---|---|---|
| Conversion interception (Calc shadow → native plan) | **Aligned** | reuses #1853 `StreamExecCalc` shadow + `RexCallConverter`; this PR only adds expr kinds |
| Converter dispatch keyed on `RexCall.class` | **Follow actual code** | factory rejects a 2nd `RexCall` converter; in-repo model evolved past per-operator dispatch (Gluten's axis) — extend the single converter |
| Op-string source of truth = native decoder | **Aligned** | `lib.rs:70-101`; same strings Spark emits |
| No proto/Rust change | **Aligned** | all 7 operators already decode natively (`PhysicalBinaryExprNode`, `PhysicalLikeExprNode`) |
| Operand type promotion in JVM (native has no implicit coercion) | **Justified deviation from Spark** | native `apply_cmp` requires matching types (`binary.rs:405-415`); we promote defensively rather than trust the planner like Spark — correctness + same-file arithmetic consistency |
| Unsupported expr → whole-Calc fallback | **Aligned** | `isSupported=false` for ESCAPE-LIKE / unsupported kinds → #1853 fallback path |

No element conflicts with AIP-1. The single deviation (in-converter promotion vs Spark's
trust-the-planner) is justified by the native hard-requirement and documented above.

## Dependencies
None new. Protobuf (transitive via `auron-flink-runtime` → `proto`) and Calcite (shaded in
`flink-table-planner_2.12`) already on the classpath.

## Test Strategy
Extend `RexCallConverterTest` (unit): one test per comparison op asserting the emitted op string;
operand-promotion test (INT vs BIGINT → TryCast on the narrower side, no outer result cast); LIKE,
NOT LIKE (asserts `negated=true`), and explicit-ESCAPE-LIKE-is-unsupported. Fix the 7 EQUALS-fallback
sites. Existing `StreamExecCalcTest` + `RexCallConverterTest` suites must stay green.

## Out of Scope
- ILIKE / case-insensitive LIKE, `SIMILAR TO`, `RLike`/regex.
- Explicit non-default ESCAPE (falls back).
- `IN` / `BETWEEN` / null-safe `<=>` (`IS NOT DISTINCT FROM`) — not in the issue's listed scope.
- Proto / Rust / native changes.
