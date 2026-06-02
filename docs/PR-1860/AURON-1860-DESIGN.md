# AURON #1860 — Convert Logical Operators to Auron Native Operators (Design)

**Issue**: [#1860](https://github.com/apache/auron/issues/1860) · **Track**: Flink Phase 1 (AIP-1)
**Status**: design for review (no code yet)

## Summary

Extend the Flink → Auron expression-conversion framework so a `Calc` containing logical/conditional
expressions runs natively instead of falling back to Flink's CodeGen. Concretely, add converters for
**AND, OR, NOT, IS NULL, IS NOT NULL, CASE WHEN** (Flink `IF(c,a,b)` is planned as CASE, so it is
covered) from Calcite `RexCall` to Auron `PhysicalExprNode` protobuf.

This is a clean Wave-1 converter PR following the merged math-operator PR (#1859). No native-engine
change, no proto change, no new dependency.

## Background

The Calc pipeline (#1853) rewrites Flink's `StreamExecCalc` to a native plan by converting each
`RexNode` through `FlinkNodeConverterFactory` (#1856). Today only math operators convert (#1859); any
Calc with a logical expression fails conversion and falls back. After this PR, those Calcs run natively.

```
StreamExecCalc (#1853 shadow)
   └─ FlinkNodeConverterFactory (#1856)
        ├─ RexInputRef / RexLiteral converters
        └─ RexCallConverter ──► math ops (#1859, merged)
                                logical ops (#1860, this PR)
   ──► PhysicalExprNode protobuf ──► native engine (DataFusion)
```

## Approach

`RexCallConverter` is the framework's single converter for `RexCall` (the factory dispatches by
`RexNode` subclass and rejects a second `RexCall` converter). Logical operators are therefore added to
**that same converter** — extending its supported-`SqlKind` set and dispatch switch with one helper per
operator. This matches the existing math-operator structure exactly and keeps the change to one
production class plus its test.

A sibling class was considered and rejected: it would require refactoring the factory to a per-`SqlKind`
sub-dispatch, enlarging the blast radius onto the merged #1859 path and the two in-flight Wave-1 PRs
(#1861, #1864) for no functional gain.

## Per-operator design

| Operator | Native node | Notes |
|---|---|---|
| AND / OR | `PhysicalBinaryExprNode` op `"And"` / `"Or"` | Calcite AND/OR are **n-ary** (`a AND b AND c` is one call). Fold left-deep into nested binaries `((a AND b) AND c)`. Operands are already BOOLEAN — no cast. |
| NOT | `PhysicalNot` | Generic single-child wrapper. `Not(=)→NotEq` belongs to comparison (#1861), not here. |
| IS NULL / IS NOT NULL | `PhysicalIsNull` / `PhysicalIsNotNull` | Single child, any input type. |
| CASE WHEN | `PhysicalCaseNode` + `PhysicalWhenThen` | Searched form `[when1,then1,…,else]`. **Each then-branch and the else are cast to the CASE result type** (see below). |

### Why CASE casts every branch

The native CASE decoder hands the branch expressions directly to DataFusion's `CaseExpr::try_new`
without inserting casts, and DataFusion requires all `then` branches and the `else` to share one type.
So the converter casts each branch to the call's result type (a no-op when types already match). This
mirrors the Spark converter's CASE handling.

### AND/OR encoding choice

Plain binary `"And"`/`"Or"` is used rather than the short-circuit nodes
(`PhysicalSCAndExprNode`/`PhysicalSCOrExprNode`). On the Spark side, short-circuit is opt-in (config /
Hive-UDF driven), not the default; there is no Flink driver for it yet. The short-circuit path remains
available for a future PR if a need arises.

## Alignment with AIP-1

| Element | Status |
|---|---|
| Plugs into `FlinkNodeConverterFactory` (#1856), emits `PhysicalExprNode` consumed by the native engine | Aligned |
| Unsupported expressions fall back cleanly (factory returns empty → Flink CodeGen Calc) | Aligned |
| n-ary fold for AND/OR | Flink-side adaptation (Calcite is n-ary where Catalyst is binary); no AIP conflict |
| Plain binary AND/OR instead of short-circuit nodes | Justified deviation — short-circuit is config-gated on Spark with no Flink driver yet; reversible |

No element conflicts with AIP-1.

## Scope

**In scope**: AND, OR, NOT, IS NULL, IS NOT NULL, CASE WHEN converters in `RexCallConverter`, plus unit
tests.

**Out of scope**: short-circuit AND/OR nodes; comparison operators (#1861); cast operators (#1864);
`IS TRUE/FALSE` and bitwise; any native-engine (Rust) change.

## Test strategy

Unit tests are added to the existing `RexCallConverterTest`, building each `RexCall` with `RexBuilder` +
`SqlStdOperatorTable` and asserting on the produced protobuf:

- AND with 2 operands → binary `"And"`.
- AND with 3 operands → left-deep nesting (`getL()` is itself a binary `"And"`).
- OR → binary `"Or"`.
- NOT → `not_expr`.
- IS NULL / IS NOT NULL → respective nodes.
- CASE where branches already match the result type → `case_` with one `when_then` + `else`, no cast.
- CASE where a branch type differs from the result type → asserts the branch is wrapped in a cast (the
  load-bearing path).

## Risks

- **CASE branch typing** — the one correctness-critical detail; covered by the mismatch test above.
