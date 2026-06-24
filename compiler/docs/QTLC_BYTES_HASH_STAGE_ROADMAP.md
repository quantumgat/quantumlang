# qtlc Bytes and Hash Stage Roadmap

## Boundary

The bytes and hash foundations are split by compiler stage:

```text
qtlc1  deterministic self-hosted seed needed to build qtlc2
qtlc2  complete compiler and artifact-format implementation
qtlc3  production optimization, security, and platform hardening
```

qtlc1 must not absorb final runtime, package, networking, signing, or general
stdlib work. Every qtlc1 command must directly improve deterministic checking,
query storage, artifact encoding, native object production, or qtlc2 output.

## qtlc1 Seed Commands

```text
POST-QTLC1-BYTES-SEED-001
  status: implemented
  complete checked ByteView, ByteMutView, ByteBuffer, and ByteCursor seed APIs
  support fixed integers, wide integers, floats, varints, strings, byte framing
  keep bounds failures typed and preserve zero-copy reads where ownership allows

POST-QTLC1-HASH-OWNED-DIGEST-002
  status: implemented
  store Hash128, Hash256, and Hash512 in owned bytes
  reject invalid digest lengths with HashError
  provide byte views and serialization without borrowing temporary cursor state

POST-QTLC1-HASH-BLAKE3-XXH3-003
  status: implemented
  implement BLAKE3-256 for deterministic compiler/cache products
  implement XXH3-64/128 for non-security table and local fingerprint use
  expose incremental domain-separated Hasher state
  pin official BLAKE3 1.5.5 and xxHash 0.8.3 portable seed sources
  keep qtlc1 free from system hash-library runtime dependencies
  buffer incremental writes through ByteBuffer in the bootstrap seed
  replace the seed bridge with QuantumLang-native algorithms in qtlc2

POST-QTLC1-HASH-STABLE-COMPILER-004
  status: implemented
  hash compiler scalars, names, IDs, Option, Result, ordered collections,
  declarations, syntax, types, HIR, MIR, object units, and query products
  never hash pointer values, allocation order, capacity, or object padding

POST-QTLC1-HASH-TYPED-DOMAINS-005
  add SourceHash, SyntaxHash, SemanticHash, ActionHash, ContentHash, QueryHash,
  ProductHash, AbiHash, ObjectHash, LinkHash, PackageHash, and ToolchainHash
  reject accidental cross-domain substitution at type checking

POST-QTLC1-CACHE-HASH-MIGRATION-006
  replace string-only CacheHash256 with canonical typed Hash256 carriers
  retain lowercase hexadecimal only for paths, diagnostics, and interchange

POST-QTLC0-NATIVE-BYTES-HASH-HOT-PATH-007
  lower byte reads/writes, endian swaps, bounds checks, BLAKE3, and XXH3 directly
  prove no reflection, dynamic dispatch, C/CPP emission, or per-value allocation
```

## qtlc2 Complete Compiler Commands

```text
POST-QTLC2-BYTES-COMPLETE-001
  complete mutable views, patching, cursor marks/forks, large and chunked streams

POST-QTLC2-ARTIFACT-CODECS-002
  implement deterministic codecs for qdb, qi, qcap, qidx, qcm, qco, qca, and qcx

POST-QTLC2-HASH-ALGORITHM-FAMILY-003
  add SHA-2, SHA-3, Keccak, SHAKE, SipHash, CRC32C, CRC64, and compatibility gates

POST-QTLC2-HASH-MERKLE-004
  add Merkle nodes, trees, proofs, chunk manifests, and partial product validation

POST-QTLC2-HASH-PARALLEL-HARDWARE-005
  add deterministic tree hashing and target capability selection

POST-QTLC2-HASH-PACKAGE-TOOLCHAIN-006
  hash package graphs, installed toolchains, capsules, indexes, and native archives
```

## qtlc3 Production Commands

```text
POST-QTLC3-BYTES-SIMD-ZEROCOPY-001
  add SIMD codecs, memory mapping, scatter/gather I/O, and provider-safe zero copy

POST-QTLC3-HASH-HARDWARE-DISPATCH-002
  select optimized implementations by CPU/device capability without changing output

POST-QTLC3-HASH-SECURITY-HARDENING-003
  add constant-time paths, key lifecycle, secure erasure, collision defenses,
  corruption quarantine, fuzzing, and adversarial input validation

POST-QTLC3-ARTIFACT-SIGNING-004
  sign and verify packages, toolchains, plugins, providers, and quantum artifacts

POST-QTLC3-REPRODUCIBILITY-CERTIFICATION-005
  certify byte-identical products across supported hosts and execution schedules
```

## Storage Rule

qtlc1 uses owned `bytes` for fixed digests because its current `Array<T,N>` is
a compiler data-contract carrier, not physical inline storage. qtlc2 may lower
the same public digest surface to inline `Array<byte,N>` after fixed-array ABI
and native layout are complete. The public source API must not change.

## No-Padding Safety Rule

QuantumLang source never observes, initializes, compares, serializes, or hashes
implicit target ABI padding.

```text
field load/store       may use compiler-known target offsets internally
object construction    initializes every declared field and tracks validity
object copy            copies declared fields or a compiler-proven full layout
equality               compares declared semantic fields
ByteCodec<T>           writes declared fields in canonical codec order
StableHash<T>          hashes declared fields and stable type/field identities
cache identity         hashes canonical products, never process memory images
```

Raw memory representation is available only through an explicit compiler-core
layout operation with target, size, alignment, initialized-byte, and provenance
proofs. Ordinary users never need to manage padding, zero it manually, or accept
undefined padding bytes in output. Debug and release builds must produce the
same canonical serialization and hash.
