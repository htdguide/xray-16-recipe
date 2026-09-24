# src/utils/mp_gpprof_server/profiles_cache.cpp

> A bounded, name-sorted cache of fetched profiles that expires by age and evicts by unpopularity.

**Needs** — [`profiles_cache.h`](profiles_cache.h.md) · [`profile_data_types.h`](profile_data_types.h.md) · [`threads.h`](threads.h.md)

**Used by** — reached through its declarations in [`profiles_cache.h`](profiles_cache.h.md); callers name that, not this file.

**Tier floor** — T2: a sorted array sized from a byte budget.

## Purpose

Every profile lookup that reaches the remote service costs a round trip measured in
hundreds of milliseconds, and a server's player list is small and asked about repeatedly.
So the tool keeps recent answers. The cache is deliberately crude — a sorted array with
bisection lookup, a fixed capacity, and an eviction pass that runs only when the array is
full — because the working set is a few thousand names and anything cleverer would be
more code than the problem deserves.

Two policies are worth naming before the contracts, because they interact:

- **Age expiry** — an entry older than the expiry window is not served, and is removed on
  the attempt.
- **Popularity eviction** — when the array is full, the sweep drops every entry that has
  never been served *as well as* every entry past the window. An entry fetched and never
  asked about again is exactly the thing worth dropping first.

## State

```text
RECORD CacheEntry
  name          : text (inline, fixed width)
  profile       : ProfileData
  fetched_at    : int (milliseconds, monotonic)
  hits          : int          # times served since it was inserted

RECORD Cache
  entries  : list<CacheEntry>  # sorted by name; capacity fixed at construction
  capacity : int               # memory budget / entry size
```

**Invariants**

- **The array is sorted by name and searched by bisection.** Everything else in this file
  is subordinate to that: `add` appends out of order and *reports* that it did, so the
  caller can restore the ordering once after a batch rather than per insertion. Serving a
  lookup against an unsorted array silently misses.
- **Capacity is fixed at construction and never grows.** The array is reserved to capacity
  up front and "full" is tested as reserve-exhausted, so growth would change the meaning
  of full. That coupling — using the reservation as the capacity — is the one genuinely
  fragile thing here; a rebuild should hold the capacity explicitly.
- **A full cache refuses an insertion rather than evicting.** The caller's response is to
  run the eviction sweep and try once more, and to give up if that also fails. So a cache
  that is full of popular, fresh entries simply stops caching — it never evicts something
  useful to make room.
- The clock these ages are measured against is the broken one described in
  [`threads.h`](threads.h.md): it advances with processor time at one-second resolution,
  so on a mostly-idle service the five-minute window is effectively never reached, and
  **age expiry almost never fires**. What actually bounds the cache is the popularity
  sweep. A rebuild with a real clock will see age expiry start working and should expect
  a higher miss rate.

## `search`

**Contract** — look a name up. On a hit within the expiry window, copies the profile out,
counts the hit and reports success. On a hit outside the window, removes the entry and
reports failure. On a miss, reports failure. Requires the array to be sorted; the caller
holds the lock.

```text
FUNCTION search(name, OUT profile) -> bool
  position <- first entry not less than name        # bisection
  IF position IS past the end THEN RETURN false
  IF entries[position].name != name THEN RETURN false
  IF now() - entries[position].fetched_at >= expiry_window
    remove entries[position]
    RETURN false
  profile <- entries[position].profile
  entries[position].hits <- entries[position].hits + 1
  RETURN true
```

**Notes**

- Removing the stale entry here rather than leaving it for the sweep means a stale name is
  re-fetched exactly once rather than repeatedly, and the array stays sorted because a
  removal preserves order.

## `add`

**Contract** — insert a profile under a name, or refresh it if already present. Reports
whether it appended out of order, which is the caller's signal to re-sort later. Reports
failure when the cache is full, leaving the caller to sweep and retry. Requires the array
to be sorted on entry for the refresh path to find an existing entry.

```text
FUNCTION add(name, profile) -> bool
  IF no free space THEN RETURN false

  position <- first entry not less than name
  IF position EXISTS AND entries[position].name == name
    entries[position].profile    <- profile          # refresh in place, order preserved
    entries[position].fetched_at <- now()
    RETURN true

  append CacheEntry(name, profile, now(), hits = 0)  # out of order on purpose
  RETURN true
```

**Invariants**

- The refresh path deliberately **does not reset the hit count**, so an entry that has been
  useful stays protected from the popularity sweep across refreshes.
- The append is unsorted, and the return value does not distinguish "refreshed in place"
  from "appended out of order" — both report success, and the caller re-sorts if *any*
  call succeeded. That over-sorts and is harmless.

## `clear_expired`

**Contract** — remove every entry that has never been served, and every entry older than
the expiry window. Preserves the relative order of what remains, so a sorted array stays
sorted.

```text
FUNCTION clear_expired()
  now <- current_time()
  remove every entry WHERE hits == 0 OR (now - fetched_at) >= expiry_window
```

**Notes**

- Dropping never-served entries is the load-bearing half, and it is aggressive: a batch of
  profiles fetched together but asked about individually loses all the ones whose request
  has not yet been answered — except that the answers are delivered in the same pass that
  populates them, so in practice every entry has been served once before a sweep can see
  it. That ordering dependency is real and undocumented; a rebuild that reorders the
  populate and answer steps will start losing entries.

## `sort`

**Contract** — restore the name ordering. Called once after a batch of insertions, under
the caller's lock.
