# src/xrGame/alife_object_registry_inline.h

> The table operations of the alife object registry: insert with a duplicate check, remove, and look up with an optional tolerance for absence.

**Needs** — [`alife_object_registry.h`](alife_object_registry.h.md)
**Used by** — [`alife_object_registry.h`](alife_object_registry.h.md)
**Tier floor** — T3: map operations

## Purpose

These three operations are on every alife hot path, which is why they are always inlined;
a rebuild folds them into the registry type. They are more than accessors, because the
registry's central invariant — one identifier, one object, no duplicates — is enforced
here and nowhere else.

## `add`

**Contract** — inserts an object under its own identifier. Fails hard on any collision.

```text
FUNCTION add(object)
  existing = objects.lookup(object.id)
  IF existing is present
    IF existing is the same object -> FAIL WITH "object already registered"
    ELSE                           -> FAIL WITH "id already taken by another object"
  objects.insert(object.id, object)
```

**Invariants** — both collisions are fatal, and they are reported as *different* failures
on purpose: re-registering the same object is a lifecycle bug in the caller, while a
second object claiming a live identifier means the identifier allocator handed out a
value that was still in use — a far more serious fault, and one that silently corrupts a
save if it survives. A rebuild that collapses them into one error loses the only
diagnostic that distinguishes them.

This is the runtime assertion conformance §6 names first. It is not a debug-only check in
spirit, even where the original compiles it out of a shipping build.

## `remove`

**Contract** — removes the entry for an identifier. Absence is fatal by default; a caller
that knows the object may already be gone passes a flag to tolerate it and gets a silent
no-op. Does **not** destroy the object — ownership passes back to the caller.

**Notes** — the "tolerate missing" flag appears on both `remove` and `object` and means
the same thing in each: *the caller has a legitimate reason to ask about an entity that
may not exist*. Callers without one must not pass it, because a missing entity there is a
real fault. A rebuild expresses this as two functions, or as a lookup returning an
optional, rather than as a boolean parameter.

## `object`

**Contract** — resolves an identifier to the registered object. Returns nothing when
absent and tolerance was requested; fails hard otherwise. This is the single hottest call
in the alife layer — every cross-entity reference goes through it — and the original wraps
it in a profiler zone for exactly that reason.

**Notes** — the lookup is an ordered map keyed by a 16-bit identifier. Nothing about the
design requires ordering; the only place order is observed is the save walk, and there it
is a reproducibility convenience rather than a requirement. A rebuild is free to use a
flat array indexed by the identifier — 65536 slots — and trade a fixed footprint for a
constant-time resolve, which is the obvious improvement this structure is too old to have
made.

## `objects`

**Contract** — the whole table, readable and writable. Handed out so that callers which
must sweep every entity (the save walk, the switch manager, debug tooling) do not each
need a visitor. A rebuild should prefer iteration over a mutable handle to the table.
