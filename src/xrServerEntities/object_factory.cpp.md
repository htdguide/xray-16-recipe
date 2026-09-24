# src/xrServerEntities/object_factory.cpp

> The registry's lifecycle: fill the table at construction, publish it to scripts at init, destroy the entries at teardown.

**Needs** — [`object_factory.h`](object_factory.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — reached through its declarations in [`object_factory.h`](object_factory.h.md); callers name that, not this file.
**Tier floor** — T2: ownership and ordering of a table of constructors.

## Purpose

Three decisions, and nothing else, live here: the registry is **global and singular**, its
**native entries are built before anything can ask for one**, and its **script entries are
built later**, after the script virtual machine exists.

## State

```text
RECORD ObjectFactory
  entries  : list<RegistryEntry>   # each maps one class identifier to two constructors
  sorted   : bool                  # false after any insertion; see actualize()
```

**Invariants** — the entry list is *not* sorted while registration is in progress; every
lookup first brings it back into order. No entry is ever removed except at teardown, so an
index into the sorted list is stable for the life of the process — which matters, because
that index is the number the script layer knows a class by.

## `construct`

**Contract** — builds the whole native table by running the registration list in
[`object_factory_register.cpp`](object_factory_register.cpp.md), and marks the table
unsorted. Allocates one entry per class. Does not touch the script engine, so it is safe
to construct before Lua exists.

## `init`

**Contract** — the second phase, run once the script engine is available: brings up the
script side so that script-declared classes can add themselves. Separate from construction
because the registry is needed by code that runs before scripts (the configuration reader
resolves a section's `class` key through it), and because the dedicated server has no
script engine at all.

## `destroy`

**Contract** — releases every entry. Ordering matters only in that the script entries hold
references into the Lua state, so the registry must die before the script engine does; the
lifetime hook in [`object_factory_inline.h`](object_factory_inline.h.md) arranges that by
destroying the whole registry when the script engine resets.

## Notes

**The registry is reachable as a process-wide global.** That is the service-locator pattern
this codebase uses everywhere; a rebuild should pass it in. What must survive is that there
is exactly *one* table and that its ordering is stable, because the script-visible class
numbering is derived from position in it.
