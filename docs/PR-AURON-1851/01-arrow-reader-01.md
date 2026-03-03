# Changelog: AURON #1851 — Arrow to Flink RowData Reader

## What Changed

Added `FlinkArrowReader` and 16 `ArrowXxxColumnVector` wrapper classes that convert Arrow `VectorSchemaRoot` data into Flink `RowData` via zero-copy columnar access.

### New Files (18 total)

**`org.apache.auron.flink.arrow`**:
- `FlinkArrowReader.java` — Orchestrator: creates wrappers from `VectorSchemaRoot` + `RowType`, provides `read(int)` returning `ColumnarRowData`

**`org.apache.auron.flink.arrow.vectors`** (16 wrappers):
- Primitive: `ArrowBooleanColumnVector`, `ArrowTinyIntColumnVector`, `ArrowSmallIntColumnVector`, `ArrowIntColumnVector`, `ArrowBigIntColumnVector`, `ArrowFloatColumnVector`, `ArrowDoubleColumnVector`
- String/Binary: `ArrowVarCharColumnVector`, `ArrowVarBinaryColumnVector`
- Decimal: `ArrowDecimalColumnVector` (with compact `fromUnscaledLong` optimization for precision ≤ 18)
- Temporal: `ArrowDateColumnVector`, `ArrowTimeColumnVector` (micros→millis), `ArrowTimestampColumnVector` (micros→TimestampData)
- Nested: `ArrowArrayColumnVector`, `ArrowMapColumnVector`, `ArrowRowColumnVector`

**Test**: `FlinkArrowReaderTest.java` — 21 test methods

## Why

Per AIP-1, the Flink integration data path requires converting Arrow vectors returned by the native engine (DataFusion/Rust) back into Flink `RowData` so downstream Flink operators can process results. This implements the reader side of the Arrow ↔ Flink bridge.

## How to Test/Verify

```bash
./build/mvn test -pl auron-flink-extension/auron-flink-runtime -am \
  -Pscala-2.12 -Pflink-1.18 -Pspark-3.5 -DskipBuildNative \
  -Dtest=FlinkArrowReaderTest
```

Expected: 21 tests pass, 0 failures, 0 errors.

## Follow-ups / Known Limitations

- **Round-trip tests** (`FlinkArrowRoundTripTest`): Requires PR #1930 (writer) to merge. Will be added in a separate PR.
- **FFI importer**: `FlinkArrowFFIImporter` (receives Arrow from native via C Data Interface, wraps with `FlinkArrowReader`) is a separate follow-up issue.
- **Dictionary-encoded vectors**: Not supported (not produced by Calc operator). Can be added if future operators require it.
- **NullType**: Intentionally unsupported in the reader (not a data-carrying type).
- **TIME precision**: Reader assumes `TimeMicroVector` (writer normalizes to microseconds). If `FlinkArrowUtils.toArrowType()` time precision mapping changes, reader must be updated to match.

## Review Notes

- Approach A (columnar `ColumnVector` wrappers) chosen per approved design doc `AURON-1851-DESIGN.md`
- Follows Flink's own `flink-python` Arrow reader pattern
- No new Maven dependencies — uses existing `arrow-vector` and `flink-table-common`
- All review logs available in `docs/review-logs/AURON-1851/`
