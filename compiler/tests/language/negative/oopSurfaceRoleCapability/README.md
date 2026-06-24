# OOP Surface Role Capability Negative Fixtures

This directory owns real invalid `.qn` files for the OOP surface role capability
proof. These files must never be imported by a positive package.

Each fixture should fail for one precise language rule:

- surface role requirements
- visibility and export rules
- variadic pack bounds
- associated type and const contracts
- call-resolution ambiguity
- native lowering constraints
- module import, re-export, alias, and conformance visibility

When expected-failure support is available, every fixture should be checked as a
single-file negative case. A fixture that succeeds is a compiler bug; a fixture
that fails for an unrelated parser crash is also a compiler bug.

