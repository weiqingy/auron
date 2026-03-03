# Review Log: Commit 2 — String, Binary, Decimal Wrappers

## Files Created (3)
- `vectors/ArrowVarCharColumnVector.java` — VarCharVector → BytesColumnVector
- `vectors/ArrowVarBinaryColumnVector.java` — VarBinaryVector → BytesColumnVector
- `vectors/ArrowDecimalColumnVector.java` — DecimalVector → DecimalColumnVector

## Reviewer Findings

| Check | Result | Notes |
|-------|--------|-------|
| Style | PASS | Consistent with commit 1 pattern |
| License header | PASS | Exact match |
| Javadoc | PASS | Class, constructor, all public methods, private helper |
| Null handling | PASS | Preconditions.checkNotNull |
| Unsafe casts | PASS | None |
| Bugs: VarChar/VarBinary getBytes | PASS | vector.get(i) returns byte[], Bytes constructor correct |
| Bugs: Decimal compact path | PASS | Sign extension correct for precision <= 18 |
| Improvements | 3 informational notes, 0 blocking |

Informational notes (not blocking):
- VarChar/VarBinary `vector.get(i)` allocates a new byte[] per call (inherent to Arrow API, matches Flink upstream)
- VarChar and VarBinary are near-identical; shared base class rejected per no-overengineering rule
- `fromLittleEndianBytes` could use ByteBuffer but manual loop avoids allocation

## Implementer Fixes (2 compile errors)

**Fix 1**: `DecimalData.MAX_COMPACT_PRECISION` is package-private in Flink.
- Replaced with local `private static final int MAX_COMPACT_PRECISION = 18`

**Fix 2**: `DecimalVector.get(i)` returns `ArrowBuf`, not `byte[]`.
- Changed to read directly from `vector.getDataBuffer()` at offset `i * 16`
- Uses `ArrowBuf.getLong(offset)` which reads 8 bytes little-endian natively
- Added `import org.apache.arrow.memory.ArrowBuf`

## Validator Results
- **Initial compile**: FAILED (2 errors in ArrowDecimalColumnVector.java)
- **After fixes**: BUILD SUCCESS (11 Java sources, 1.188s)
- **Checkstyle**: BUILD SUCCESS — 0 violations

## Team-lead Sign-off
Gate PASSED after implementer fixes. Proceeding to Commit 3.
