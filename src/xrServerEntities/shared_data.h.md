# src/xrServerEntities/shared_data.h

> Sharing one loaded block of configuration across every instance that names it, keyed by whatever identifies the template.

**Needs** — [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`InfoPortion.h`](../xrGame/InfoPortion.h.md) · [`PhraseDialog.h`](../xrGame/PhraseDialog.h.md) · [`encyclopedia_article.h`](../xrGame/encyclopedia_article.h.md) · [`character_info.h`](character_info.h.md) · [`specific_character.cpp`](specific_character.cpp.md) · [`specific_character.h`](specific_character.h.md)
**Tier floor** — T2: a process-wide keyed cache with reference-counted lifetime; nothing here needs manual layout.

## Purpose

Hundreds of entities in a level are instances of a handful of templates. Every one of them
would otherwise parse its own copy of the same configuration section at construction:
identical numbers, read identically, thrown away identically. This file is the generic
answer — a per-template block of parsed data, created once on first demand, shared by
reference, and released when the last instance that wanted it goes away.

It is generic over both the *shape* of the shared block and the *key* that identifies it, so
one type shares by class identifier and another by section name without either of them
owning a cache.

## State

Three cooperating pieces.

```text
RECORD SharedStore OF (Data, Key)     # one per (Data, Key) combination, process-wide
  blocks   : map<Key, Data>           # owned
  refcount : int                      # instances currently interested
  self_delete : bool                  # whether the store frees itself at zero

RECORD SharedBlock                    # the base every shared block extends
  loaded : bool                       # has this block been filled yet

RECORD SharedUser OF (Data, Key)      # mixed into the instance type
  block : optional<Data>              # borrowed from the store; never owned
```

**Invariants** — a block in the store is never destroyed while the store lives, only at
store teardown. The `loaded` flag is what makes first-demand filling work: a block exists as
soon as it is asked for, but is empty until someone fills it, and exactly one instance does
the filling.

## the store

**Contract** — a process-wide, lazily created cache keyed by the template key. Asking for a
key that is absent **creates an empty block and inserts it**, so the answer is never
nothing; the caller then discovers from the block's `loaded` flag whether it must fill it.
Teardown destroys every block.

**Invariants** — the reference count tracks how many instances hold the store, not how many
hold any particular block. A block is never individually released.

## the user mixin

**Contract** — mixed into an instance type. Construction takes a reference on the store;
destruction releases it. The instance holds a borrowed pointer to its block and must never
free it.

**`load_shared(key, section)`** is the whole idea in four lines:

```text
FUNCTION load_shared(user, key, section)
  user.block = store.get_or_create(key)
  IF NOT user.block.loaded
    user.fill_from_configuration(section)   # supplied by the concrete type
    user.block.loaded = TRUE
```

**Invariants** — `fill_from_configuration` runs **exactly once per key** for the process's
lifetime, and it runs on whichever instance happened to be constructed first. That instance
must therefore fill the block from the *section alone* and put nothing instance-specific in
it. This is the invariant the whole file exists to provide and the one a rebuild must
preserve; violating it gives every later instance the first one's private state.

**The two-phase form** — `start_load_shared(key)` answers whether filling is needed and
`finish_load_shared()` marks it done — exists for types whose configuration arrives in
pieces from several places rather than from one section. Same invariant, opened up.

## Notes

**Lifetime is the fragile part, and the file knows it.** A block's storage outlives every
instance that uses it up to the moment the last one releases the store; if the store is
configured to free itself at zero, a level transition that drops every instance of a
template and then re-creates them re-parses the configuration. If it is configured *not* to,
the blocks live until an explicit teardown and someone must perform it. Both modes ship, and
the choice is made by the concrete type. A rebuild is better served by a single policy —
keep the cache for the process's life, since the data is small and the templates are
finite — and the only thing it must not do is free a block while an instance still points
at it.

**Sharing is per (data shape, key type) pair**, not per instance type. Two unrelated types
that share the same block shape and key type share one cache and can collide on a key. No
collision exists in the shipped code, but nothing prevents one; a rebuild should key the
cache by the consuming type as well.

**The debug build logs the reference count at teardown** and asserts that it reached zero
and that self-deletion was off. That is the only diagnosis available for a shared block
outliving its users, which is worth keeping in some form.
