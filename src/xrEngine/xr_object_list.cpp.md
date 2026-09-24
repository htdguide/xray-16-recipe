# src/xrEngine/xr_object_list.cpp

> The registry of every client object on the loaded level — who exists, who gets updated this frame, who is being destroyed, and who answers to a network id.

**Needs** — [`xr_object.h`](xr_object.h.md) · [`xr_object_list.h`](xr_object_list.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`xrSheduler.h`](xrSheduler.h.md) · [`CustomHUD.h`](CustomHUD.h.md) · [`GameFont.h`](GameFont.h.md) · [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [`xrCore/Threading/TaskManager.hpp`](../xrCore/Threading/TaskManager.hpp.md)
**Used by** — [`xr_object_list.h`](xr_object_list.h.md)
**Tier floor** — T2 for the bookkeeping, T1 for the network export: the packet writer measures itself in bytes against a hard packet limit and back-patches a size field, which needs a byte-addressable buffer. Everything else is lists and indices.

## Purpose

There is exactly one of these per loaded level, and it is the answer to "what exists right
now". It owns three orthogonal responsibilities that share one data structure because they
share one lifetime:

1. **Membership** — create, destroy, find, count; the two-bucket split between objects that
   want per-frame work and objects that do not.
2. **The per-frame update pass** — deciding *which* objects run this frame, in what order,
   and exactly once each.
3. **Destruction as a transaction** — an object is never deleted at the moment someone
   asks; it is queued, everyone who could be holding a reference is told, and only then is
   the memory returned.

Note the division of labour with the [scheduler](xrSheduler.cpp.md): this file runs the
*unconditional, this-frame* update (`UpdateCL`); the scheduler runs the *time-budgeted,
rate-limited* update. An object can participate in both, and they mean different things.

## State

```text
RECORD ObjectRegistry
  by_net_id      : list<optional<Object>>   # fixed 65535 slots, indexed directly by id
  active         : list<Object>             # processing enabled: candidates for UpdateCL
  sleeping       : list<Object>             # processing disabled
  destroy_queue  : list<Object>             # queued this frame, drained at update end
  primary_crows  : list<Object>             # accumulated work for the next update
  worker_crows   : list<list<Object>>       # one list per task worker; merged into primary
  relcase_hooks  : list<(slot_ref, callback)>
  stats          : update statistics for this frame
```

Invariants:

- An object is in `active` or in `sleeping`, never both and never neither. Destroying an
  object that is in neither is a fatal error, not a no-op — a leaked or double-registered
  object must stop the program, because from here on the registry's counts are lies.
- Bucket membership tracks the object's own "processing enabled" flag. Nothing else moves
  an object between buckets.
- `by_net_id` holds at most one object per id; the id `0xffff` is reserved to mean *none*
  and is never stored. (A spectator pseudo-object legitimately carries that id and is
  therefore simply absent from the map.)
- A relcase hook's recorded slot always equals its index in `relcase_hooks`.
- On level unload all three object lists must be empty; anything left is reported by name
  as a leak before being force-destroyed.

### Why the network map is a flat array

Ids are 16-bit and assigned by the server, so the entire key space is 65536 entries. A
direct-indexed array is 512 KB and makes lookup a single load with no hashing and no
branch on a miss. That lookup happens once per object per imported network packet, on the
frame thread. A rebuild with a fast hash map may use one; the decision recorded here is
that *lookup must not allocate and must not be able to fail slowly*.

## `Update`

**Contract** — the per-frame object pass. Called once per frame from the level; takes a
force flag that overrides the paused state (used to flush one update while the game is
paused, e.g. after a teleport). Runs on the frame thread. Does not block. Ends by draining
the destroy queue, so an object queued this frame is gone by the time this returns.

**Invariants** — every object's `UpdateCL` runs at most once per frame, and never before
its parent's. After the pass, no object is marked as a crow.

```text
FUNCTION ObjectRegistry.update(force)
  begin frame statistics if this is a new frame

  # Fold in work requested from worker threads (see "crows" below)
  FOR EACH list IN worker_crows
    append list to primary_crows; clear list

  IF (not paused OR force) AND (frame_delta > epsilon OR force) THEN
    IF crow mode enabled THEN workload = primary_crows
    ELSE                      workload = active, and clear primary_crows

    snapshot = copy of workload            # the pass may mutate the source lists
    clear primary_crows

    FOR EACH o IN snapshot                 # clear marks first, as a separate pass:
      o.is_crow = false                    # an object may re-request during its own update
      o.crow_request_frame = none

    FOR EACH o IN snapshot
      o.pre_update()
      single_update(o)

    FOR EACH o IN active   DO o.post_update(was_skipped = false)
    FOR EACH o IN sleeping DO o.post_update(was_skipped = true)

  drain_destroy_queue()
```

**Notes** — the snapshot is taken onto scratch storage that dies with the call, and the
source lists are cleared *before* the pass runs, not after. That ordering is what makes it
legal for an object to request another update from inside its own update: the request
lands in the now-empty list and is serviced next frame, instead of being wiped by the
clear.

The three-phase shape — pre-update on the workload, update on the workload, post-update on
*everything* — is a real decision. `post_update` is the hook for work that must happen
whether or not the object ran this frame (position interpolation, visual bookkeeping), and
it is told which case it is in.

## Crows — the update workload

A *crow* is an object that has asked to be updated on the next pass. The name is the
project's own and has no meaning beyond "in the flock that runs this frame".

The problem it solves: most objects on a level do not need per-frame work on most frames,
but the ones that do change from frame to frame — an object becomes interesting because it
is visible, because the player is within a radius of it, or because it declares itself
permanently interesting. Walking all active objects to ask each one would cost more than
updating them.

So the polarity is inverted: an object marks *itself* a crow, from anywhere, including
from a worker thread. To make that free of locks, each task worker owns its own crow list
and appends to it without synchronisation; the frame thread concatenates all of them at
the start of the update. Requests are deduplicated by a per-object frame stamp rather than
by searching the list.

A console flag disables the whole mechanism and updates every active object instead. That
exists as a correctness escape hatch — if an object forgets to mark itself, the bug looks
like "it stopped moving" — and it is the reason the crow lists are cleared rather than
merely ignored when the flag is set.

## `SingleUpdate` — parent-before-child, once per frame

**Contract** — updates one object, having first updated its parent chain. Idempotent
within a frame.

```text
FUNCTION single_update(o)
  IF o.last_update_frame == current_frame THEN RETURN   # already ran this frame
  IF not o.processing_enabled THEN RETURN               # went to sleep mid-pass
  IF o has a parent THEN single_update(o.parent)        # attachments see a fresh parent

  o.last_update_frame = current_frame
  o.update()
```

**Invariants** — after `o.update()` returns, the object must have recorded that it ran.
The engine asserts this: a subclass that overrides the update and forgets to call its
base's is a silent, gradually-diverging bug, so it is turned into a loud one naming the
object.

**Notes** — the parent recursion is what makes attachment correct without an explicit
ordering pass. A weapon's transform is derived from the actor's, so the actor must have
moved first; rather than topologically sorting the whole workload, each object pulls its
ancestors forward on demand and the frame stamp keeps the work linear.

It also catches a real failure mode: if a child survives into the update while its parent
or root is already queued for destruction, that is an out-of-order teardown and is
reported with both names. It is not fixed, only reported — the destruction ordering is the
caller's to get right.

## Destruction

**Contract** — `register_object_to_destroy` queues; the queue drains at the end of the
next update pass. Between those two moments the object is still fully present and still
answers to its id. An object may be queued only once.

```text
FUNCTION register_object_to_destroy(victim)
  REQUIRE victim not already queued
  append victim to destroy_queue
  FOR EACH o IN active + sleeping
    IF o.parent == victim AND o is not already dying THEN
      report "child outlived parent"; mark o for destruction too

FUNCTION drain_destroy_queue()
  IF destroy_queue is empty THEN RETURN

  # Phase 1: tell everything that could hold a reference. Order matters only in that
  # all notification precedes all deletion.
  FOR EACH o IN active + sleeping
    FOR EACH victim IN destroy_queue DO o.drop_reference_to(victim)
  FOR EACH victim IN destroy_queue DO sound_world.drop_reference_to(victim)
  FOR EACH hook IN relcase_hooks
    FOR EACH victim IN destroy_queue DO hook.callback(victim)
  FOR EACH victim IN destroy_queue DO hud.drop_reference_to(victim)

  # Phase 2: now nothing points at them.
  FOR EACH victim IN destroy_queue (reverse order)
    victim.net_destroy()
    remove_from_registry(victim)
  clear destroy_queue
```

**Notes** — this two-phase shape is the single most important invariant in the chapter and
is named in the conformance list: *a destroyed object is unreferenced by the scheduler, the
render graph, the sound world and the physics world before its memory is released*. The
quadratic notification loop (every survivor is told about every victim) is accepted because
the queue is normally one or two objects; a rebuild that regularly destroys hundreds per
frame should index the references instead.

The reverse-order drain matters when queuing cascades: a victim's destruction may queue its
own children, which were appended after it.

Removing from the registry itself is careful about the crow lists — an object may be
sitting in a worker's crow list at the moment it dies, so all of them are searched. It is
*not* an error for an object to be absent from the crow lists; it *is* a fatal error for it
to be absent from both bucket lists.

## `Create` / `Destroy`

**Contract** — creation asks the persistent object pool for an instance by class name and
files it as sleeping; the object activates itself later if it wants updates. Destruction is
the private counterpart of the drain above: unregister the network id, scrub the crow
lists, remove from whichever bucket holds it, return the instance to the pool.

Objects are never allocated or freed here. They come from and return to a pool owned by the
persistent game layer, which is what makes level reload cheap and what makes "unregistered
object being destroyed" a detectable condition.

## `net_Export`

**Contract** — writes as many objects' network state into one packet as will fit, starting
at a cursor, and returns the cursor to resume from. Only objects that declare themselves
relevant and are not dying are written. Called on the frame thread by the client's network
update.

```text
FUNCTION net_export(packet, start, max_object_size) -> next_start
  FOR i FROM start WHILE i < active.count + sleeping.count
    o = object at flat index i across (active then sleeping)
    IF o.net_relevant AND not o.dying THEN
      packet.write id as 16-bit
      mark = packet.open_sized_chunk(size field is 8-bit)
      o.net_export(packet)
      REQUIRE bytes written since mark < 256      # the size field is one byte
      packet.close_sized_chunk(mark)
      IF max_object_size >= remaining space in packet THEN BREAK
  RETURN i + 1
```

**Invariants** — the per-object payload is under 256 bytes, because the length prefix is
one byte. This is a frozen wire fact, not a tunable: the importer skips unknown objects by
advancing exactly that many bytes, so a longer payload does not merely truncate, it
desynchronises the rest of the packet. The engine treats overflow as fatal in debug builds.

**Notes** — the flat index over "active then sleeping" is why the cursor is a plain integer
and why it is only valid within one frame: any membership change reshuffles the mapping.
The caller resumes from `i + 1` rather than `i`, which means the object that did not fit is
skipped entirely this round rather than retried — a deliberate choice of forward progress
over completeness.

## `net_Import`

**Contract** — reads (id, sized payload) pairs until the packet is exhausted, handing each
payload to the object with that id. An id that resolves to nothing has its payload skipped
using the length prefix. Never fails on unknown ids: a client legitimately receives updates
for objects it has not yet spawned.

## `net_Register` / `net_Unregister` / `net_Find`

**Contract** — install, remove and look up by network id. Lookup of the reserved *none* id
yields nothing rather than indexing the map. Registration requires an in-range id.

## `FindObjectByName` / `FindObjectByCLS_ID`

**Contract** — linear scans over both buckets, active first. Used by the console, by
scripts and by debug tooling; not on any per-frame path. The class-id search returns the
first match, which is meaningful only for classes the level has one of.

## `o_activate` / `o_sleep` / `o_crow` / `o_count` / `o_get_by_iterator`

**Contract** — bucket movement and enumeration. Activation and sleep move the object
between the two lists and mark it a crow, so that a newly-woken object gets one update
immediately rather than waiting to be noticed. Enumeration is by flat index across the two
buckets, which is the same ordering the network export uses.

## `Load` / `Unload`

**Contract** — `Load` asserts the registry is empty, which is how a level-load-over-a-live-
level bug surfaces immediately rather than as corruption. `Unload` reports every surviving
object by id, section and name — a deliberate leak log — then force-destroys each one.

## `DumpStatistics` / `GetStats`

**Contract** — per-frame counters: time spent in the update pass, crow count, active count,
total count, and the same time as a percentage of the frame's engine total. Raises a
performance alert when the pass exceeds three milliseconds, which is the chapter's
convention for "this subsystem is now the frame's problem" on a sixteen-millisecond budget.

## `relcase_register` / `relcase_unregister`

**Contract** — install or remove a destruction-notification callback for a non-object
subscriber (see [`pure_relcase.cpp`](pure_relcase.cpp.md)). Registration writes the
assigned slot back through the caller's handle.

```text
FUNCTION relcase_unregister(slot_ref)
  i = value at slot_ref
  move the last entry into position i
  write i back through the moved entry's own slot handle   # keep its self-knowledge true
  drop the last entry
```

**Notes** — swap-with-last removal in exchange for one write-back through a pointer the
subscriber owns. Order of callbacks is therefore not stable and must not be relied on. A
rebuild with generational handles gets the same O(1) removal without the aliasing.
