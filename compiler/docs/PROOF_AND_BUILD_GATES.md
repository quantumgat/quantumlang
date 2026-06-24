# qtlc1 Proof And Build Gates

Status: active proof checklist

## Current Scaffold Proof

Passed:

```text
qtlc0 check qtlang/compiler/driver
qtlc0 check qtlang/compiler/source
qtlc0 check qtlang/compiler/diagnostics
qtlc0 check qtlang/compiler/syntax
qtlc0 check qtlang/compiler/moduleSystem
qtlc0 check qtlang/compiler/nameResolve
qtlc0 check qtlang/compiler/typeSystem
qtlc0 check qtlang/compiler/semantic
qtlc0 check qtlang/compiler/hir
qtlc0 check qtlang/compiler/mir
qtlc0 check qtlang/compiler/optimize
qtlc0 check qtlang/compiler/quantum
qtlc0 check qtlang/compiler/machine
qtlc0 check qtlang/compiler/backend
qtlc0 check qtlang/compiler/backend/native
qtlc0 check qtlang/compiler/backend/quantum
qtlc0 check qtlang/compiler/platform
qtlc0 check qtlang/compiler/buildSystem
qtlc0 check qtlang/compiler/query
qtlc0 check qtlang/compiler/interface
qtlc0 check qtlang/compiler/bootstrap
```

Full package proof:

```text
qtlc0 check qtlang/compiler
  ok files=251

qtlc0 build -j4 qtlang/compiler
  build: ok
  package: qtlc1
  output: qtlang/compiler/build/debug/qtlc1
```

Artifacts:

```text
qtlang/compiler/build/debug/qtlc1
qtlang/compiler/build/debug/interfaces/qtlc1/**/*.qi
qtlang/compiler/build/debug/qtlc1.qcap
qtlang/compiler/build/debug/qtlc1.qidx
qtlang/compiler/build/debug/qtlc1.qnpkg
qtlang/compiler/build/debug/libqtlc1.qna
```

## Implemented Commands

```text
POST-QTLC1-MAGIC-FAST-001
  stable ID model, query keys, and initial query data shells
  module proof: qtlc0 check qtlang/compiler/query, ok files=13
  package proof: qtlc0 check qtlang/compiler, ok files=220
  build proof: qtlc0 build -j4 qtlang/compiler, build ok

POST-QTLC1-SOURCE-STYLE-001
  qtlc1 source uses fn-free Quantum function declarations
  positive fixture proof: qtlc0 check bootstrap_safe_qtlc1_style_ok.qn
  negative fixture proof: qtlc0 check bootstrap_void_return_rule_bad.qn rejects missing return

POST-QTLC1-MAGIC-FAST-002
  BuildDatabase execution API, product lookup shape, validation contracts,
  query execution records, and cache report updates
  module proof: qtlc0 check qtlang/compiler/query, ok files=17
  package proof: qtlc0 check qtlang/compiler, ok files=224
  build proof: qtlc0 build -j4 qtlang/compiler, build ok

POST-QTLC1-SOURCE-STYLE-002
  qtlc1 query data shapes use locked QuantumLang visibility style
  module proof: qtlc0 check qtlang/compiler/query, ok files=17
  package proof: qtlc0 check qtlang/compiler, ok files=224
  build proof: qtlc0 build -j4 qtlang/compiler, build ok

POST-QTLC0-APP-BINARY-ARTIFACT-CONTRACT-001
  app target with [library] stays binary by default
  readable .qi files emit under interfaces/<importRoot>
  legacy interface cleanup does not remove the app binary
  dry-run proof: artifact_mode=binary output=qtlang/compiler/build/debug/qtlc1
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-003
  source fingerprint, token, and green-tree AST query product contracts
  module proof: qtlc0 check qtlang/compiler/query, ok files=20
  package proof: qtlc0 check qtlang/compiler, ok files=227
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/query
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-004
  export surface and symbol-level dependency product contracts
  module proof: qtlc0 check qtlang/compiler/query, ok files=24
  package proof: qtlc0 check qtlang/compiler, ok files=231
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/query
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-005
  type signature, typed body, and typed node query product contracts
  module proof: qtlc0 check qtlang/compiler/query, ok files=27
  package proof: qtlc0 check qtlang/compiler, ok files=234
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/query
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-006
  HIR body and MIR body query product contracts
  module proof: qtlc0 check qtlang/compiler/query, ok files=29
  package proof: qtlc0 check qtlang/compiler, ok files=236
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/query
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-007
  generic argument records and generic instance query product contract
  module proof: qtlc0 check qtlang/compiler/query, ok files=31
  package proof: qtlc0 check qtlang/compiler, ok files=238
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/query
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-008
  native object product, link input, and linked target product contracts
  module proof: qtlc0 check qtlang/compiler/query, ok files=34
  package proof: qtlc0 check qtlang/compiler, ok files=241
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/query
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-009
  qcap/qidx mmap handles and dependency source-avoidance proof contract
  module proof: qtlc0 check qtlang/compiler/interface, ok files=8
  package proof: qtlc0 check qtlang/compiler, ok files=244
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/interface
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-010
  qtlcd hot-build protocol, request, response, and hot-state contracts
  module proof: qtlc0 check qtlang/compiler/buildSystem, ok files=11
  package proof: qtlc0 check qtlang/compiler, ok files=248
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/buildSystem
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-MAGIC-FAST-011
  magic-fast proof scenario budgets, proof result, and changed-product report
  module proof: qtlc0 check qtlang/compiler/buildSystem, ok files=14
  package proof: qtlc0 check qtlang/compiler, ok files=251
  build proof: qtlc0 build -j4 qtlang/compiler, build ok
  interface proof: generated .qi files under interfaces/qtlc1/buildSystem
  executable proof: file build/debug/qtlc1 reports ELF executable; run exits 0

POST-QTLC1-INCREMENTAL-QUERY-ENGINE-001
  incremental query engine plan for no-change/private-body/public-API/target
  invalidation and native-only artifact policy
  module proof: qtlc0 check qtlang/compiler/query, ok files=36
  build-system proof: qtlc0 check qtlang/compiler/buildSystem, ok files=16
  test proof: qtlc0 check qtlang/compiler/tests, ok files=417
  package proof: qtlc0 check qtlang/compiler
  build proof: qtlc0 build -j8 qtlang/compiler and qtlc0 build -j8
  qtlang/compiler/tests
  executable proof: qtlang/compiler/tests/build/debug/qtlc1-tests exits 0
  artifact rule: qtlc1 build never emits generated .c, .cpp, or header source
```

## Do Not Claim Yet

These are not done:

```text
qtlc1 is not a working compiler yet
qtlc1 does not parse source yet
qtlc1 does not type-check source yet
qtlc1 does not lower or emit native code yet
qtlc1 does not build qtlc2 yet
runtime/stdlib/plugins are not part of qtlc1 yet
```

## Per-Slice Gate Rule

Every implementation slice must pass:

```text
qtlc/build/compiler/driver/qtlc check <module-dir>
qtlc/build/compiler/driver/qtlc check qtlang/compiler
```

Only run build after full check passes:

```text
qtlc/build/compiler/driver/qtlc build -j4 qtlang/compiler
```

## Reserved-Name Gate

Reject or rename scaffold files that use keyword-like module names:

```text
type
build
qubit
qreg
qresult
qfunc
qany
qif
qfor
qwhile
qloop
```

Known fixed names:

```text
typeSystem/compilerType.qn
buildSystem/
quantum/types/qubitType.qn
quantum/types/qregType.qn
quantum/types/qresultType.qn
quantum/types/qfuncType.qn
quantum/types/qanyType.qn
quantum/controlFlow/quantumIf.qn
quantum/controlFlow/quantumFor.qn
quantum/controlFlow/quantumWhile.qn
quantum/controlFlow/quantumLoop.qn
```

## Self-Host Gate

The real qtlc1 proof is:

```text
qtlc0 builds qtlc1
qtlc1 checks qtlang/compiler
qtlc1 builds qtlc2
qtlc2 can check qtlang/compiler
qtlc2 can build next compiler artifact
```

Only then can the new compiler be called self-hosted.
