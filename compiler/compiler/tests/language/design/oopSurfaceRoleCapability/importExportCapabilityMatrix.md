# Import/Export Capability Matrix

This matrix defines the module system proof required before the OOP surface
role capability package becomes a compiled fixture.

## Positive Cases

1. A public role exported from one module is visible to another module.
2. A public model type exported from one module is visible to another module.
3. A facade module can re-export roles, model types, and surface owners.
4. Imported aliases keep separate canonical identities.
5. Role conformance declared in one module is visible only when exported.
6. Associated type and const requirements are visible through exported roles.
7. Variadic pack constraints work when pack members come from different modules.
8. `Caps.all<Role>()` resolves by canonical symbol identity.
9. `Caps.indexOf<T>()` uses canonical imported type identity.
10. Every OOP surface remains callable after import: `impl`, `view`, `context`, `extract`, and `extend`.
11. Re-exported facade imports behave the same as direct module imports.
12. Import aliases participate in diagnostics and native symbol naming.

## Negative Cases

1. Importing a non-exported type is rejected.
2. Importing a private role is rejected.
3. Importing an internal conformance from another package is rejected.
4. Re-exporting a private type through a public module is rejected.
5. Re-exporting a public role whose associated item leaks a private type is rejected.
6. Using a role conformance that exists but is not exported is rejected.
7. Two unaliased imports with the same local name are rejected.
8. Import alias resolves to the wrong canonical declaration id.
9. `Caps.all<Role>()` succeeds by text name without canonical identity.
10. `Caps.indexOf<T>()` changes when imports are reordered.
11. Surface-qualified lookup bypasses module visibility.
12. Native symbol naming uses an import alias instead of canonical symbol identity.

## Canonical Identity Rule

Every imported symbol is resolved to:

```text
package id
module id
declaration id
generic substitution key
visibility domain
```

Names and aliases are frontend bindings only. They must not become semantic
identity, pack identity, or native symbol identity.

## Required qtlc0 Tables

```text
ModuleSymbolRow
ExportRow
ImportBinding
ReExportRow
ConformanceExportRow
AssociatedItemExportRow
```

All OOP surface conformance rows must reference these module rows instead of
copying text paths.

## Future Commands

After conversion to real positive modules:

```sh
qtlc/build/compiler/driver/qtlc check qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlc/build/compiler/driver/qtlc build qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler --quiet
```
