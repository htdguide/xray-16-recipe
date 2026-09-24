# src/xrGame/sound_collection_storage_inline.h

> Reaches the single sound-collection storage, creating it on first use.

**Needs** — [`sound_collection_storage.h`](sound_collection_storage.h.md)
**Used by** — [`sound_collection_storage.h`](sound_collection_storage.h.md)
**Tier floor** — T2: lazily creates a process-lifetime cache

## Purpose

One accessor, separated from its declaration for C++ compilation reasons. What it decides
is worth a line: the storage is **created on first use and never created by anyone else**.
There is no startup step that builds it, so a rebuild must not assume one exists before
the first creature registers a sound kind.

## State

```text
# a single nullable process-wide slot holding the storage
```

## `sound_collection_storage`

**Contract** — returns the storage, creating it if the slot is empty. Never fails. Not
thread-safe: the check-then-create has no interlock, which is safe only because every
caller is on the simulation thread. A rebuild with any audio loading on a worker must
guard it.

**Notes** — nothing in this file destroys the storage or clears the slot. Teardown is
somebody else's — the process-level shutdown that owns the game module. That asymmetry
(lazy create, explicit destroy elsewhere) is the pattern for the chapter's process-wide
caches, and it is the reason the storage's destructor has to be defensive about what is
still loaded.
