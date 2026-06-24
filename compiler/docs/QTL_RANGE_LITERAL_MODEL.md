# QuantumLang Range Literal Model

## Canonical Forms

```qn
a..<b   // included start, excluded end
a..=b   // included start, included end
a..     // included start, open end
..b     // open start, excluded end
..=b    // open start, included end
..      // full range
```

`a..b` is accepted only as a compatibility spelling for `a..<b`. New source
uses `..<` so the end-bound rule is visible.

## Types

- Finite ranges use `Range<T>`.
- `a..` uses `RangeFrom<T>`.
- `..b` uses `RangeTo<T>`.
- `..=b` uses `RangeToInclusive<T>`.
- Individual bounds use `RangeBound<T>`.
- The unqualified full literal uses `RangeFull`.
- `Range<T>` has one canonical native ABI: two machine words, `start` at byte
  offset `0` and `count` at byte offset `8`.
- Open-bound metadata is never embedded in finite `Range<T>`.
- `Range<T>()` is the empty bounded range `[0, 0)`.
- `RangeSpec<T>` resolves finite and open range forms against a collection
  length, producing a checked finite `Range<T>`.

## Context Rules

- A bounded integer range may drive `for`, `par for`, and `shard for`.
- An open range cannot drive a `for` loop because it has no finite completion.
- Range patterns support bounded, inclusive, open-start, open-end, and full
  forms.
- Collection indexing with a range produces a borrowed view:
  `Slice<T>`, `BytesView`, or `str`.
- A bounded integer loop lowers to direct compare/increment native control
  flow. It allocates no range object.
- An invalid or mismatched native `Range<T>` layout is a compiler error, not a
  fallback ABI.
