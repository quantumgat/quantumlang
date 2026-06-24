# Quantum-First Compiler Plan

Status: architecture contract

QuantumLang must treat quantum as a first-class language pillar, not as a
small backend option.

## Correct Layering

```text
quantum/
  owns quantum language semantics

backend/quantum/
  emits quantum artifacts and provider handoff

machine/
  owns CPU and QPU target capability facts
```

Do not put all quantum logic in `backend/quantum`.

## Reserved Quantum Type Words

These are reserved language type words:

```text
qubit
qreg
qresult
qfunc
qany
```

Compiler metadata names must stay distinct:

```text
QubitType
QregType
QresultType
QfuncType
QanyType
```

Files use metadata names:

```text
quantum/types/qubitType.qn
quantum/types/qregType.qn
quantum/types/qresultType.qn
quantum/types/qfuncType.qn
quantum/types/qanyType.qn
```

## Required Quantum Language Model

qtlc1 must model:

```text
qubit linear resource identity
qreg<N> shape and allocation identity
qresult measurement result identity
qfunc quantum callable identity
qany dynamic quantum value identity
qif path-sensitive quantum control flow
qfor qreg iteration
qwhile restricted-profile quantum loops
qloop bounded/target-supported quantum loops
no-clone rules
reset/release rules
measurement collapse effects
entanglement effects
purity rules for qfunc
provider/profile constraints
target gate/capability constraints
QIR mapping
OpenQASM mapping
shot/histogram execution plan
noise/simulator profile
```

## Quantum Directory Ownership

```text
quantum/types
  quantum type metadata and reserved-word identity

quantum/resource
  ownership, lifetime, allocation, reset, release, no-clone

quantum/effect
  quantum effects, purity, measurement, entanglement

quantum/op
  gate, gate set, intrinsic, operation, measurement, barrier, control

quantum/circuit
  circuit model, builder, topology, schedule, parameters

quantum/controlFlow
  quantumIf, quantumFor, quantumWhile, quantumLoop, profile restrictions

quantum/verify
  verifier passes before backend handoff

quantum/lower
  AST/HIR/MIR to quantum IR facts, QMIR, QTIR

quantum/diagnostic
  quantum-specific diagnostics and errors
```

## Backend Quantum Ownership

```text
backend/quantum/qirEmitter.qn
  QIR textual/binary emission

backend/quantum/openQasmEmitter.qn
  OpenQASM emission

backend/quantum/provider.qn
  provider boundary identity

backend/quantum/providerProfile.qn
  cloud/local/simulator provider capabilities

backend/quantum/targetProfile.qn
  selected target profile

backend/quantum/transpile.qn
  transpilation plan

backend/quantum/shotPlan.qn
  shot execution plan

backend/quantum/noiseModel.qn
  noise model facts

backend/quantum/simulatorPlan.qn
  simulator execution facts

backend/quantum/artifact.qn
  emitted artifact metadata
```

## Machine Quantum Ownership

```text
machine/quantumTarget.qn
  QPU target identity

machine/quantumCapability.qn
  max qubits, supported gates, measurement modes, provider limits

machine/quantumTopology.qn
  coupling map/connectivity
```

## First Quantum Milestone

The first implementation should not execute quantum programs yet. It should
prove the compiler owns the type/resource/effect facts.

Minimum proof:

```text
parse qubit/qreg/qresult/qfunc/qany spellings
parse qif/qfor/qwhile/qloop target-language spellings
create QuantumType facts
reject cloning qubit
require reset/release policy
record measurement effect
emit target unsupported diagnostic for unavailable gate
```

qtlc0 currently tokenizes/parses `qif` and `qfor`. qtlc1 should reserve
`qwhile` and `qloop` in the target-language parser, but qtlc1 compiler source
must not use those spellings until qtlc0 can parse them or until qtlc1 is
building qtlc2 source.
