# src/xrServerEntities/object_factory_inline.h

> The registry's actual algorithms: lazy creation, sorted insertion with duplicate detection, binary lookup by tag, and the index-in-table number that scripts use to name a class.

**Needs** — [`object_factory.h`](object_factory.h.md) · [`object_item_abstract.h`](object_item_abstract.h.md) · [`xrGame/ai_space.h`](../xrGame/ai_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: sorting and binary search over a table of constructors.

## Purpose

This is where the registry's behaviour lives, even though the name suggests a mechanical
header. Four decisions are made here and nowhere else: *when* the registry comes into
existence, *how* a class identifier maps to an entry, *what happens to a duplicate*, and
*what number a script sees when it asks for a class*.

## `object_factory`

**Contract** — returns the one registry, creating it on first call. Creation also installs
a hook on the script engine's reset event that destroys the registry; the next call then
rebuilds it. Never fails; the first call is expensive (it runs the whole registration
list), every later call is a pointer read.

**Invariants** — the registry never outlives the script engine it published itself to. A
script engine reset invalidates every script-declared entry, so the whole table is thrown
away rather than partially pruned.

```text
FUNCTION registry() -> ObjectFactory
  IF no registry EXISTS
    registry = build_registry()        # runs the native registration list
    registry.init()                    # brings up the script half
    ON script_engine_reset DO destroy registry
  RETURN registry
```

**Notes** — rebuilding the entire table on a script reset is the coarse but correct answer:
a script-declared class holds a reference into a Lua state that no longer exists, and there
is no safe partial recovery. It costs a few milliseconds and happens only when a developer
reloads scripts.

## `add`

**Contract** — appends one entry and marks the table unsorted. In a checked build it first
scans for a duplicate class identifier *and* a duplicate script name, and aborts on either.
Both checks are linear, which is why they are checked-build only: registration inserts a few
hundred entries.

**Invariants** — a class identifier appears at most once; a script class name appears at
most once. This is the invariant that makes the shipped tag values meaningful — two classes
answering to one tag would make a spawn record ambiguous.

## `actualize`

**Contract** — sorts the table by class identifier if any insertion has happened since the
last sort, then clears the dirty mark. Every lookup calls it first. Idempotent and cheap
after the first call.

**Notes** — the deferred sort exists because registration is a long literal list and
insertion order is meaningless; sorting once at the end beats maintaining order through a
few hundred insertions. The important consequence is downstream: after the final sort the
table is in **class-identifier order**, and that order is what the script numbering is built
on, so it is deterministic across runs even though the registration list is written in
readability order.

## `item`

**Contract** — binary search for the entry with a given class identifier. Two forms: the
strict one asserts the tag exists and reports the tag in the message (a missing tag means a
spawn file naming a class this build does not have); the tolerant one answers `none`, which
is how the configuration scanner in
[`object_factory_spawner.cpp`](object_factory_spawner.cpp.md) skips sections whose class is
not compiled in.

```text
FUNCTION lookup(tag : ClassIdentifier, tolerant : bool) -> optional<RegistryEntry>
  actualize()
  e = first entry whose identifier is not less than tag
  IF e is missing OR e.identifier != tag
    IF NOT tolerant THEN FAIL WITH "unknown class identifier " + text_of(tag)
    RETURN none
  RETURN e
```

## `script_clsid`

**Contract** — returns the entry's **position in the sorted table**, as a signed integer.
This is the number the script layer calls a class by: the class-name-to-number enumeration
published in [`object_factory_script.cpp`](object_factory_script.cpp.md) is exactly this
index, and every server record carries its own value in a field.

**Notes** — this is the subtle part of the whole registry. The 64-bit tag is the *data*
identity and is frozen by the shipped levels; the small integer index is the *script*
identity and is frozen only by the registration list being stable within one build. Scripts
compare `object:clsid() == clsid.wpn_ak74`, so both sides of that comparison come from the
same table in the same process, and the number never reaches disk or the wire — which is why
it is allowed to be position-derived. A rebuild that changes the registration list changes
these numbers harmlessly; a rebuild that lets scripts *store* one has created a bug.

## `client_object` and `server_object`

**Contract** — look up the entry and delegate to its constructor pair. `client_object` takes
only the tag; `server_object` also takes the configuration section name, because a server
record is `(class, section)` and cannot be built without both. A tolerant `server_object`
answers `none` for an unknown tag instead of aborting.
