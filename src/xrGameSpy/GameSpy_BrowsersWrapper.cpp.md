# src/xrGameSpy/GameSpy_BrowsersWrapper.cpp

> Presents several master lists as one list: fans a refresh out across all three titles,
> concatenates their servers into one stable index space, and reduces their several states
> into one verdict.

**Needs** — [`GameSpy_BrowsersWrapper.h`](GameSpy_BrowsersWrapper.h.md) · [`GameSpy_Browser.h`](GameSpy_Browser.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [`xrCommon/xr_array.h`](../xrCommon/xr_array.h.md) · [`xrCore/Threading/ScopeLock.hpp`](../xrCore/Threading/ScopeLock.hpp.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_BrowsersWrapper.h`](GameSpy_BrowsersWrapper.h.md)
**Tier floor** — T2: concurrent mutation of a shared index from a query thread while the
frame thread reads it.

## Purpose

The three shipped titles each have their own master list, and a player browsing for a
match wants to see all of them at once. This file owns one list per title and makes them
look like a single list to the UI: one index space, one count, one refresh, one status.

It also answers a question the single-list object cannot: *is the online layer working?*
One title's master list being unreachable is not an outage; all three being unreachable
is. The state accumulator in this file is what makes that distinction.

## State

```text
RECORD ListSlot
  list             : ServerList    # one per title, from GameSpy_Browser
  known_count      : int           # how many of this list's servers we have indexed
  active           : bool          # this title is browsed (all three are, as shipped)
  reported_failure : bool          # latched once this list has failed; cleared on refresh

RECORD ServerRef                   # one entry per server in the unified index
  list      : ServerList           # which list it came from
  local_idx : int                  # its index within that list
  raw       : optional<handle>     # last raw handle handed out for it; see the note

RECORD ListAggregate
  slots         : list<ListSlot>          # fixed at three
  index         : list<ServerRef>         # the unified, append-only index
  index_lock    : mutex                   # guards slots and index together
  subscribers   : list<optional<callback>>
  subscriber_lock : mutex
```

**Invariants**

- The unified index is **append-only within a refresh**. A slot's `known_count` only ever
  grows until the next refresh clears everything. This is the whole point of the object:
  the underlying lists may re-sort themselves as they learn about servers, and a UI that
  is holding an index must not have that index change meaning underneath it.
- `index_lock` covers both the slots and the index, because appending to the index reads a
  slot's count. They are one piece of state wearing two names.
- A raw handle is re-read from its list every time it is handed out, and the *previous*
  handle for that entry is overwritten. Handles are therefore valid only until the next
  request for the same entry.

## `CGSUpdateStatusAccumulator`

**Contract** — collects a sequence of list states and reduces it two ways. Constructed
with a seed value, which is the answer if nothing else is ever registered. Registering is
**not** locked; reducing is. Cheap; allocates only as the sequence grows.

```text
FUNCTION optimistic() -> UpdateStatus     # the best state anyone reported
FUNCTION pessimistic() -> UpdateStatus    # the worst state anyone reported
FUNCTION is_good(s) -> bool               # s is success or connecting_to_master
```

**Invariants** — both reductions depend on the state enum being *ordered* best-to-worst
(see [`GameSpy_Browser.h`](GameSpy_Browser.h.md)). "Good" is the top two values: a list
that is still connecting has not failed.

**Notes**

The seed is the accumulator's honest answer to "what if there is nothing to reduce", and
the two call sites choose different seeds on purpose: a refresh seeds with *out of
service* (no list even tried, so the service is out), a poll seeds with *master
unreachable* (lists exist but none answered). A rebuild that collapses this into a plain
fold must keep the seeds distinct.

Registering without the lock while reducing with it is a real race, not a documented
relaxation — the reductions run on the frame thread and registration happens on the same
thread in both call sites, so it does not fire, but a rebuild should not reproduce the
asymmetry.

## `RefreshList_Full`

**Contract** — clears the unified index and starts a refresh on every active list.
Returns the *optimistic* reduction: the aggregate is refreshing if any list is. Latches
a failure on any list whose refresh was refused outright. Invalidates every index and
every raw handle the caller holds — the doc comment on the declaration says so, and it is
the caller's job to drop them first.

```text
FUNCTION refresh_all(local : bool, filter : text) -> UpdateStatus
  acc <- accumulator seeded with out_of_service
  LOCK index_lock DURING
    forget_all_servers()
    FOR EACH slot IN slots
      IF NOT slot.active THEN CONTINUE
      status <- slot.list.refresh(local, filter)
      acc.register(status)
      IF NOT is_good(status) THEN slot.reported_failure <- true
  RETURN acc.optimistic()
```

**Notes** — optimistic is the right reduction here because a refresh is a *start*: one
list that began connecting is enough to show the player a "connecting" dialog, and the
lists that failed will be reported by the poll.

## `Update`

**Contract** — one poll per frame. Polls every list (including inactive ones), latches
failures, and then makes the judgement this object exists for: if *any* list is still
working, report the best state anyone has; if *every* list has failed or is inactive,
forget every server and report the worst state anyone has.

```text
FUNCTION poll() -> UpdateStatus
  acc <- accumulator seeded with master_unreachable
  LOCK index_lock DURING
    dead <- 0
    FOR EACH slot IN slots
      status <- slot.list.poll()
      acc.register(status)
      IF NOT is_good(status) THEN slot.reported_failure <- true
      IF slot.reported_failure OR NOT slot.active THEN dead <- dead + 1

    IF dead < count of slots
      RETURN acc.optimistic()
    forget_all_servers()
    RETURN acc.pessimistic()
```

**Invariants** — the total outage branch *empties the index*. That is what the player
sees when the service is gone: the list does not merely stop growing, it clears, and the
menu raises the master-unreachable dialog. This is the null path's visible shape on the
server-list screen.

**Notes** — inactive lists are polled anyway and counted as dead. Harmless as shipped
(all three are active), but a rebuild that ever deactivates a list must not let that
deactivation read as an outage.

## `UpdateCb` — the change notification

**Contract** — invoked from a list's query thread whenever anything about that list
changed. Extends the unified index to cover any newly discovered servers in that list,
then fans the notification out to every subscriber. Takes both locks, one after the
other, never together.

```text
FUNCTION on_list_changed(which : ServerList)
  LOCK index_lock DURING
    slot <- the slot owning `which`            # linear search over three
    cur  <- which.count()
    ASSERT cur >= slot.known_count             # a list never shrinks mid-refresh
    WHILE cur > slot.known_count
      append ServerRef(which, slot.known_count, none) to index
      slot.known_count <- slot.known_count + 1

  LOCK subscriber_lock DURING
    FOR EACH cb IN subscribers
      IF cb is set THEN cb()
```

**Invariants** — new servers are appended to the *unified* index in discovery order, not
in the underlying list's order. The comment in the source is explicit about why: the
underlying list re-orders itself as records fill in, and a client holding an index must
not see that index change meaning. This intermediate index is the entire reason this
object is more than a loop.

**Notes** — subscribers are called **on the query thread**, with the subscriber lock
held. Everything a subscriber touches must tolerate that, and a subscriber must not
subscribe or unsubscribe from inside its own callback. A rebuild would do better to
marshal the notification onto the frame thread.

## `SubscribeUpdates` · `UnsubscribeUpdates`

**Contract** — register a change callback and get back a stable id; unregister by that id.
Subscribing reuses the first vacated slot before growing, so ids are stable for a
subscriber's lifetime but are recycled afterwards. Unsubscribing an id that was never
registered is undefined — the index is not bounds-checked.

**Notes** — the recycling is what makes a plain index usable as a handle. A rebuild with
a generational handle or a token object gets the same property without the recycling
hazard.

## `GetServersCount` · `GetServerInfoByIndex` · `GetServerByIndex` · `RefreshQuick` · `HasAllKeys` · `CheckDirectConnection`

**Contract** — each takes a unified index, maps it to (list, local index) and delegates.
`GetServerInfoByIndex` overwrites the delivered record's index with the *unified* one, so
that a caller can hand the record back to this object. Every one of them asserts the
index is in range rather than returning an error: an out-of-range index here means the
caller kept an index across a refresh, which the contract forbids.

## `GetBool` · `GetInt` · `GetFloat`

**Contract** — read one advertised field off a raw handle. The handle does not say which
list it came from, so each call linearly searches the unified index for the entry whose
last-handed-out handle matches, and delegates to that entry's list.

**Notes** — the source explains the choice and the explanation is worth keeping: returning
a pointer *into* the index would be faster but the index grows while a refresh is running,
so such a pointer would dangle. A rebuild should sidestep both by making the handle carry
its list — the linear search is an artefact of the handle being an opaque address.

No string reader is exposed here even though the single-list object has one. That is a
gap, not a decision.

## `ForgetAllServers`

**Contract** — empties the unified index and resets every slot's count and failure latch.
Called on refresh and on total outage. Takes `index_lock` — and is also called from
`RefreshList_Full`, which already holds it, so the lock must be re-entrant. A rebuild
should split this into a locked entry point and an unlocked body rather than rely on
re-entrancy.
