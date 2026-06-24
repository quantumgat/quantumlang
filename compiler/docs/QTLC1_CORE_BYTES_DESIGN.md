# qtlc1 Core Bytes Design

## Scope

`compiler/core/bytes` is the compiler-owned binary foundation used by source
loading, fingerprints, caches, `.qi`, `.qcap`, `.qidx`, object metadata, and
native artifact writing. It does not depend on runtime or stdlib packages.

General formats such as JSON, BSON, MessagePack, compression, encryption, and
network protocols remain separate packages built on this foundation.

## Literal Rules

```qn
let magic: bytes = b"QCAP"
let payload: bytes = [0x51, 0x43, 0x41, 0x50]
let empty: bytes = []
```

An untyped `[]` remains a vector literal. A `bytes` target makes it a compact
byte value. Every element must be representable as `byte`.

## Core Types

- `bytes`: owned immutable byte value.
- `BytesView`: borrowed immutable native view.
- `BytesMutView`: borrowed exclusive mutable native view.
- `ByteView`: checked offset/length view used by compiler code.
- `ByteMutView`: checked mutable offset/length view.
- `ByteBuffer`: growable encoder with one allocation after capacity planning.
- `ByteCursor`: checked linear decoder.
- `ByteRead<T>`: decoded value plus the next cursor.
- `ByteOrder`: little, big, or native endian.
- `ByteError`: typed failure with offset/request/availability facts.
- `ByteCodec<T>`: compile-time codec role.

## Buffer Surface

`ByteBuffer` supports:

- append one byte or a borrowed byte view;
- bool, signed and unsigned integers from 8 through 256 bits;
- `f32` and `f64`;
- unsigned and signed varints;
- explicit/default endian order;
- alignment, reserve, clear, and finish.

Examples:

```qn
let buffer = ByteBuffer.withCapacity(64)
let next = buffer.writeU32(42)?
let network = next.writeU64(99, ByteOrder.BigEndian)?
let payload = network.finish()
```

The value is always required. `writeU32(ByteOrder.BigEndian)` is invalid.

## Cursor Surface

`ByteCursor` supports:

- bounds checks through `require`;
- seek, skip, align, position, and remaining;
- bool, integer, float, varint, and byte-view reads;
- default endian from the cursor or explicit order per read.

```qn
let cursor = ByteCursor(payload)
let version = cursor.readU32()?
let flags = version.cursor.readU16(ByteOrder.BigEndian)?
```

## Codec Surface

```qn
pub role ByteCodec<T> {
  encodedSize(order: ByteOrder) -> usize
  encode(buffer: ByteBuffer, order: ByteOrder) -> Result<ByteBuffer, ByteError>
  decode(cursor: ByteCursor, order: ByteOrder) -> Result<ByteRead<T>, ByteError>
}
```

The bounded extension supplies:

```qn
value.toBytes()
value.toBytes(ByteOrder.BigEndian)
```

There is no reflection or runtime field map. A compiler-generated or explicit
`ByteCodec<T>` implementation lowers fields directly in declaration order.
Schema-aware formats should define their own codec role.

## Native Hot Path

- fixed-width operations lower to direct loads/stores and byte swaps;
- checked cursor offsets remain scalar values and can be range-proven;
- proven bounds checks are eliminated;
- views are zero-copy;
- `encodedSize` enables one allocation;
- codec dispatch is statically resolved;
- no C/CPP source, LLVM, reflection, dynamic dispatch, or named-argument map;
- malformed input returns `Result`, never unchecked memory access.

Object padding is never encoded. `ByteCodec<T>` lowers declared fields directly
and does not copy raw object memory. Target ABI padding remains an internal
layout detail and cannot affect bytes, equality, hashing, or cache identity.

## Implementation Commands

The stage ownership and complete command sequence are defined in
[QTLC_BYTES_HASH_STAGE_ROADMAP.md](QTLC_BYTES_HASH_STAGE_ROADMAP.md).

`POST-QTLC1-CORE-BYTES-FOUNDATION-001`

- create the compiler byte type/API contract;
- lock byte literal rules;
- expose checked buffer/cursor operations;
- expose bounded codec conversion;
- prove core module checking.

`POST-QTLC0-NATIVE-BYTES-HOT-PATH-002`

- bind the core methods to existing native binary intrinsics;
- specialize endian operations;
- add assembly proof for direct loads/stores and byte swaps;
- prove zero-copy views and removed proven bounds checks.

`POST-QTLC1-CORE-BYTES-CODEC-003`

- generate `ByteCodec<T>` implementations from typed declarations;
- use codecs for qtlc1 cache and interface products;
- prove deterministic encode/decode round trips.
