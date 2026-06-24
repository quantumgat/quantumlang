# OOP Surface Role Capability Positive Proof

This is the buildable positive proof package for the OOP surface role capability
work. Keep this package valid at every step; promote design cases into this tree
only after qtlc0 can parse, check, and lower them without weakening qtlc1.

Current focused proof:

- first-class `role` declarations
- `extend` conformance for concrete and generic owners
- role-conforming `impl`, `impl view`, `context`, `extract`, and `extend`
- `impl<T, ...Caps>` conformance with role bounds
- variadic capability pack use through `HttpCtx<T, ...Caps>`
- direct native call proof through `ctx.packValue()` and every OOP surface call

Expected commands:

```sh
qtlc/build/compiler/driver/qtlc check qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlc/build/compiler/driver/qtlc build qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability/build/debug/oop-surface-role-capability-proof
```
