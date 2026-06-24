# Memory Safety Negative Fixtures

These fixtures are intentionally invalid ownership programs. They must not be
imported by positive tests.

Each file targets one central qtlc1 ownership rule:

- moved owners cannot be reused;
- released owners cannot be borrowed or dereferenced;
- read-only views cannot be written through;
- local pointers cannot escape to global lifetime;
- locks must protect mutable access;
- `Once<T>` cannot be reinitialized by normal user code;
- raw manual `free` / `delete` stays rejected in safe code.

When expected-failure test running is available, each fixture should be checked
alone. A fixture that succeeds is a compiler bug.
