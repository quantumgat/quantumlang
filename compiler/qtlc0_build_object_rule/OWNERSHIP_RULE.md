# QuantumLang Object Argument Rule

Locked final source rule:

```qn
arenaFromBatch(batch)       // default read borrow, no move
arenaFromBatch(&batch)      // explicit read borrow, same meaning
arenaFromBatch(^batch)      // move ownership, batch unusable after
arenaFromBatch(&!batch)     // mutable borrow
let pointer: *Batch = &batch
```

Reserved syntax:

```qn
pointer.valid // validated pointer target scope
...value   // spread / variadic expansion
```

Type-side intent:

```qn
readBatch(batch: Batch) -> Arena      // read borrow by default for non-copy objects
takeBatch(batch: ^Batch) -> Arena     // owns/moves
editBatch(batch: &!Batch) -> Arena    // mutable borrow
rawBatch(batch: *Batch) -> Arena      // raw pointer
```

Safety rule:

- Plain `value` must not silently move a non-copy object.
- Ownership transfer must be visible with `^value`.
- Mutable access must be visible with `&!value`.
- Pointer intent is visible in the target type: `*T` or `*const T`.
- Compiler diagnostics must point at the move site and the later invalid use.
- Native lowering must stay direct: no hidden clone, no runtime ownership map, no dynamic dispatch.

Current qtlc0 status:

- qtlc0 currently still treats non-copy call arguments as moves.
- This project locks the final desired rule and builds the safe/read-use shape.
- qtlc0 parses and enforces `^`, `&!`, target-directed pointers, and const targets.






  POST-QTLC0-EXPLICIT-OWNERSHIP-SYMBOLS-001
    value       -> read borrow/default for non-copy objects
    &value      -> explicit read borrow
    ^value      -> explicit move
    &!value     -> mutable borrow
    *T          -> mutable pointer target
    *const T    -> read-only pointer target
    pointer.valid -> validated direct target access
    ...value    -> spread/variadic stays separate

  After that, this should compile:

  let table = Table {
    storage: arenaFromBatch(batch)
    name: batch.name
    count: batch.count
  }
