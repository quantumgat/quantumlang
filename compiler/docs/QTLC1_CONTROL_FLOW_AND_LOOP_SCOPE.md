# qtlc1 Control Flow And Loop Scope

Status: scope lock before frontend/control-flow implementation

qtlc1 must support enough control flow to build qtlc2. This includes normal
compiler control flow, loop typing, break/continue behavior, minimal parallel
loop facts, and first-class quantum control-flow facts.

qtlc1 must not implement the final runtime scheduler, async runtime, stdlib
parallel API, or quantum execution runtime.

## Bootstrap Sources Checked

qtlc0 already has real parser/lexer surface for:

```text
qtlc/compiler/lexer/include/qtlc/lexer/token_kind.h
  if, else, match, loop, while, for, break, continue
  qif, qfor, qfunc, qubit, qreg, qresult
  par, shard, window, group, decode, probe, merge, pipeline, search, stabilize
  controlled, adjoint, inverse

qtlc/compiler/parser/src/statements/control_stmt_parser.cpp
  if, if-let, qif, controlled, adjoint/inverse, loop, while, while-let,
  for, qfor, match, return, break, continue, labels

qtlc/compiler/parser/src/statements/parallel_stmt_parser.cpp
  par for, shard for, window, group, domain for, decode, probe, merge,
  pipeline, search, stabilize

qtlc/compiler/parser/src/statements/expr_stmt_parser.cpp
  statement dispatch for normal control flow, quantum control flow,
  data/domain loops, parallel loops, require, yield, unsafe blocks

qtlc/tests/fixtures/control_flow_e2e_ok.qn
  normal for/while/loop, labels, break/continue, qfor/qif, par for

qtlc/tests/fixtures/while_ergonomics_forms_ok.qn
  while, while-let, loop break values, labeled break/continue, qwhile design

qtlc/tests/fixtures/for_ergonomics_100_forms_ok.qn
  broad final for-loop design surface

qtlc/tests/fixtures/quantum_qif_qfor_profiles_ok.qn
  qif/qfor profile proof
```

## qtlc1 Rule

qtlc1 owns compiler facts for control flow:

```text
parse tree shape
control-flow graph
reachability
definite return
break/continue target resolution
break value typing
loop result typing
pattern binding flow
effect/capability flow
incremental query keys per body/control-flow region
HIR/MIR lowering facts
```

qtlc1 does not own:

```text
async executor runtime
thread pool runtime
parallel stdlib implementation
quantum simulator
cloud quantum provider SDK
database/page/binary loop engines
final high-level loop API ecosystem
```

## Normal Control Flow Required For qtlc1

qtlc1 must support these forms because compiler source naturally needs them:

```text
if condition { ... }
if let pattern = value { ... }
else
else if
match value { pattern => expr }
return
return value
require condition
```

qtlc1 must type-check:

```text
condition is bool
if-let pattern binding
match arm pattern compatibility
match result type unification when used as expression
return type compatibility
definite return for non-void functions
```

## Normal Loop Required For qtlc1

qtlc1 must support the practical loop subset needed for qtlc2 compiler source:

```text
loop { ... }
while condition { ... }
while let pattern = value { ... }
while pattern = value { ... }
for item in iterable { ... }
for index in start..end { ... }
for index in start..=end { ... }
break
break value
continue
label: loop/while/for
break label
continue label
```

qtlc1 must type-check:

```text
loop result type from break values
break value compatibility
continue has no value
label target exists and is a loop
for iterable element type
range bound integer compatibility
while condition bool compatibility
while-let binding scope
unreachable code after unconditional break/continue/return
```

qtlc1 should not implement the full final loop surface yet:

```text
for await
while timeout
while poll
for concurrent
for hot
domain database/file/binary/page engine loops
query-builder loop literals
progress/throttle/debounce loops
full collection comprehensions
```

Those are qtlc2/qtlc3 runtime/stdlib/package work unless qtlc2 compiler source
proves it needs one.

## Parallel Loop Scope

qtlc1 must have two separate parallel concepts:

```text
compiler scheduler parallelism
  internal qtlc1 build engine behavior; required for superfast qtlc2 builds

language parallel loop facts
  qtlc1 compiler facts for qtlc2 source if qtlc2 uses par/shard constructs
```

The qtlc1 build engine must be parallel even if qtlc1 source never uses
`par for`. The scheduler owns parallel file lexing/parsing, signature scanning,
body checking, HIR/MIR lowering, object emission, and quantum transpile facts.

Minimal qtlc1 language support:

```text
par for item in iterable { ... }
par for item in iterable(schedule: static, chunk: 4) { ... }
shard for item in iterable using local: T { ... } merge + into target
parallel 8 { ... }
```

Locked language model:

```text
par for
  independent iterations with an implicit join

shard for
  independent iterations with shard-local mutation and deterministic merge

atomic<T>
  intentional shared scalar state; safe sequentially-consistent ordering by
  default, with explicit advanced memory ordering

normal shared mutable capture
  compile error

parallel N { ... }
  explicit worker region

while and loop
  sequential inside each worker
```

`par while` and `par loop` are not language forms. Their iteration boundaries
depend on changing state and do not define safe partitioning. Queue-driven
parallel work uses an explicit worker region:

```qn
parallel 8 {
  while let job = jobs.receive() {
    process(job)
  }
}
```

This keeps one normal implementation for `while` and `loop`, and one shared
parallel-region model for partitioning, captures, reductions, cancellation,
backpressure, and joining.

qtlc1 must type-check facts only:

```text
parallel body cannot mutate captured shared state
parallel body may mutate shard-local state
reducer must be deterministic/associative when reduction is requested
parallel loop has implicit join
parallel loop query products are partitionable by stable body/region ID
atomic shared mutation is explicit and memory-order valid
parallel worker count is positive
while/loop bodies inside workers remain sequential
```

qtlc1 must not implement the final stdlib parallel runtime.

## Quantum Control Flow Required For qtlc1

qtlc1 must support quantum syntax as compiler facts, not execution runtime.

Required qtlc1 quantum control-flow surface:

```text
qif condition { ... }
qfor q in qreg { ... }
qfor index, q in qreg { ... }
qfor q in qreg controlled by flag { ... }
qwhile condition { ... }
qloop { ... }
controlled control { ... }
adjoint { ... }
inverse { ... }
```

Current qtlc0 note:

```text
qtlc0 lexer/parser has qif and qfor keywords today.
qwhile and qloop appear in fixtures/design docs but are not qtlc0 keywords yet.
```

Because qtlc1 is built by qtlc0, new qtlc1 compiler source must not use
`qwhile` or `qloop` until qtlc0 can parse them or until qtlc1 is compiling
qtlc2 source. qtlc1's parser/type system should still include qwhile/qloop as
target-language syntax for qtlc2 if qtlc2 source needs quantum loops.

Quantum control-flow facts qtlc1 must check:

```text
qif condition is quantum/classical measurement-compatible
qif path-sensitive qubit release/reset rules
qfor iterable is qreg/qreg<N>
qfor binding is a borrowed/linear qubit resource
qfor cannot clone or leak qubits
controlled qfor requires adaptive quantum profile when needed
qwhile requires restricted quantum profile
qloop requires bounded/terminating or target-supported profile
measurement effects are recorded
reset/release obligations are satisfied on all paths
```

qtlc1 must lower these to facts:

```text
QuantumControlFlowFact
QuantumResourceFlowFact
QuantumEffectFlowFact
QuantumProfileRequirement
QMIR/QTIR control-flow region
```

qtlc1 must not implement:

```text
quantum runtime execution
simulator runtime
provider job submission
shot histogram runtime
hardware scheduling runtime
```

## Domain Loop Scope

qtlc0 has parser support for many specialized loops:

```text
window
group
decode
probe
merge
pipeline
search
stabilize
for page
for lanes
for bits
for bitrun
cas loop
```

qtlc1 should not implement all of these as product language features. For
qtlc1, they are allowed only when qtlc2 compiler source needs them as compiler
implementation tools.

Default qtlc1 decision:

```text
normal loop + minimal par/shard + qif/qfor/qwhile/qloop facts are in scope
domain loop engines are out of scope until qtlc2/qtlc3
```

## Implementation Commands

```text
POST-QTLC1-CFLOW-001
  add normal control-flow tokens/AST facts for if/else/match/return/require

POST-QTLC1-CFLOW-002
  add loop/while/while-let/for/range/labeled break/continue AST facts

POST-QTLC1-CFLOW-003
  add control-flow graph, reachability, loop result, and definite-return facts

POST-QTLC1-CFLOW-004
  add minimal par-for/shard-for compiler facts and deterministic-capture checks

POST-QTLC1-CFLOW-005
  lower normal and minimal parallel control flow into HIR/MIR products

POST-QTLC1-QCONTROL-001
  add qif/qfor/qwhile/qloop token, AST, and parser facts

POST-QTLC1-QCONTROL-002
  add qif path-sensitive resource/effect flow facts

POST-QTLC1-QCONTROL-003
  add qfor qreg iteration and no-clone/no-leak facts

POST-QTLC1-QCONTROL-004
  add qwhile/qloop restricted-profile and boundedness facts

POST-QTLC1-QCONTROL-005
  lower quantum control flow into QMIR/QTIR products
```

Each command must leave `qtlang/compiler` qtlc0-checkable and must prove cache
keys/replay products for the affected body/control-flow region.
