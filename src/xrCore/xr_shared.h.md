# src/xrCore/xr_shared.h

> A reference-counted cache keyed by interned string: ask for a resource by name, get the one that already exists or a freshly loaded one, and sweep the unreferenced ones on demand.

**Needs** — [`xrstring.h`](xrstring.h.md) · [`../xrCommon/xr_map.h`](../xrCommon/xr_map.h.md) · [`xrDebug.h`](xrDebug.h.md)

**Used by** — [`xrCore.h`](xrCore.h.md) · [`xr_shared.cpp`](xr_shared.cpp.md)

**Tier floor** — T3: this is a keyed cache with a reference count and a sweep. Nothing about it is layout- or device-facing, and a tier with its own reference counting deletes most of the file.

## Purpose

Several resources in the game are expensive to build, are named in configuration by a section
identifier, and are shared by many instances: every rifle of the same model wants the same
first-person animation set, the same bone indices and the same muzzle offsets. Loading that
once per rifle wastes memory and load time; loading it once per *name* and handing out
references does not.

This file is the generic machinery for that pattern, in three parts: a base for cached
values, the cache itself, and a handle. A rebuild in a tier with reference-counted handles of
its own gets the handle for free and keeps only the cache's two interesting decisions — the
**construct-under-the-key** protocol and the **two sweep modes**.

Worth knowing before investing in it: in the current tree this machinery has **exactly one
user**, the weapon's first-person display data. Whatever generality it was written for did
not arrive. A rebuild should implement the one case directly unless a second appears.

## State

```text
RECORD CachedValue                  # the base every cached resource extends
  reference_count : int

RECORD Cache of V
  entries : map<InternedString, V>  # keyed by interned name; see xrstring.h

RECORD Handle of V
  target : optional<V>              # none when the handle is empty
```

**Invariants**

- The cache key is an [interned string](xrstring.h.md), so a lookup is a pointer comparison
  and not a string comparison. That is the whole reason the cache is affordable to consult
  per object rather than per name — and it means the key must already have been interned by
  the caller.
- **The cache holds every entry it ever created, including ones with a reference count of
  zero.** An entry reaching zero is not destroyed; it waits for a sweep. That is deliberate:
  a weapon dropped and picked up again should not reload its animation set, and the window
  between them is a frame, not a level.
- The cache must be **empty when it is destroyed**. Destroying a non-empty cache is a
  programming error, and the original asserts rather than cleaning up, because a live entry
  at teardown means something still holds a handle to it and freeing it would leave that
  handle dangling.
- A handle either holds a value whose reference count it has incremented, or holds nothing.
  There is no third state.

## `dock` — find or construct under the key

**Contract** — Takes a key and a *constructor callback*, returns the cached value for that
key, creating it if absent. The callback is given the key and a freshly made, otherwise
empty value and fills it in; it reports success or failure. On failure the value is
destroyed and **nothing is cached** — so a resource that fails to load is retried on the
next request rather than remembered as broken. The returned value's reference count is *not*
incremented by this operation; the handle does that.

```text
FUNCTION dock(key, construct) -> optional<V>
  existing = entries.get(key)
  IF existing is present THEN RETURN existing

  fresh = new V with reference_count = 0
  IF construct(key, fresh) THEN
    entries.put(key, fresh)
    RETURN fresh
  destroy fresh
  RETURN none
```

**Invariants**

- The value is created **before** the callback runs and is filled in place, not returned by
  the callback. That inversion is what lets the callback carry its own context — the weapon
  that is asking — without the cache knowing anything about what it caches.
- A failed construction leaves the cache unchanged. The caller receives nothing and must
  handle it; the original returns a null reference and several call sites do not check.
- Not thread-safe. Two threads docking the same absent key both construct, and one of the
  two values is leaked into the map's entry while the other is returned to a caller that will
  decrement a count nobody tracks. Every caller in the tree runs on the main thread during a
  load, which is why this has never been a problem and is exactly the assumption a rebuild
  must make explicit.

## `clean` — the sweep

**Contract** — Two modes, selected by a flag:

- **gentle** — destroy and remove only the entries whose reference count is zero, leaving
  live entries in place. This is the between-levels reclaim.
- **forced** — destroy and remove **every** entry regardless of its reference count. This is
  the shutdown path, and it invalidates every outstanding handle.

**Invariants** — The forced mode is only safe once nothing holds a handle. It exists because
the cache must be empty at destruction (see above) and because at shutdown the holders are
being torn down in an order the cache cannot know. A rebuild should treat it as "assert that
nothing is referenced, then clear" rather than as a legitimate runtime operation.

**Notes** — The gentle mode walks and erases in one pass, which in the original requires
stepping the iterator past the entry before erasing it. That is the *problem it solves* — a
removal must not invalidate the position the walk is holding — and it is a problem every
rebuild has in some form, solved by whatever its collection offers.

## The handle

**Contract** — A value that holds a reference to a cached entry, copyable, with the reference
count maintained across copies and destruction. Its one non-obvious operation is creation:
given a key, a cache and a constructor callback, it docks, takes a reference, and releases
whatever it held before.

**Invariants** — The order inside creation is load-bearing and is the classic
self-assignment guard: **take the new reference first, release the old one second**. Doing it
the other way round destroys the value when a handle is re-created with the same key it
already holds. The same order appears in the copy operation.

**Notes** — **Releasing never destroys anything.** It decrements the count and stops; the
value stays in the cache until a sweep takes it. That is the division of responsibility the
whole design rests on — the handle owns a *count*, the cache owns the *lifetime* — and it is
what makes the "keep entries at zero references" behaviour possible at all. A rebuild that
reaches for its tier's standard reference-counted pointer gets a handle that destroys on the
last release, which is the opposite behaviour, and must then either accept that resources
reload or keep one reference in the cache itself.

The release path also clears the handle's pointer only when the count reaches zero, which
reads like an oversight but is harmless: every path that releases either overwrites the
pointer immediately afterwards or is destroying the handle. A rebuild clears it
unconditionally and loses nothing.

The handle exposes its target as read-only. The one caller's debug helpers cast that away to
poke at a shared value in place, which tells you the read-only accessor is documentation
rather than enforcement — and that the real invariant is "a shared value is immutable once
constructed", which a rebuild should enforce by construction instead.

An empty handle reads as nothing, and the caller is expected to check. Several call sites do
not, which works only because the one resource this caches is always present for a weapon
that exists at all.
