# src/xrNetServer/NET_PlayersMonitor.h

> The server's client roster: a collection that is only ever reached through a locked
> traversal, because two threads reach it.

**Needs** — [`NET_Shared.h`](NET_Shared.h.md) · [`NET_Common.h`](NET_Common.h.md) · [Seam: Threads](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`NET_Server.cpp`](NET_Server.cpp.md) · [`NET_Server.h`](NET_Server.h.md) · [`NET_Server.cpp`](empty/NET_Server.cpp.md) · [`NET_Server.h`](empty/NET_Server.h.md)
**Tier floor** — T2: a guarded collection with callback traversal. Nothing here touches
layout or a device.

## Purpose

The transport delivers admission and disconnection events on its own thread while the
simulation walks the roster on another. This type exists so that fact is enforced rather than
remembered: there is no operation that hands out a reference to the collection, or to an
iterator over it, or to a position in it. Every read and every write happens inside a call
that holds the lock for its whole duration.

That is the entire design. A rebuild in a language with a different concurrency model — an
actor that owns the roster, a channel that serializes access — satisfies it differently and
just as well. What may not change is the guarantee: **no traversal may observe the roster
mid-mutation, and no mutation may happen during a traversal.**

## State

```text
RECORD ClientRoster
  clients       : list<ClientRecord>
  lock          : Mutex
  traversing    : bool          # a traversal is in progress
  owner_thread  : ThreadId      # debug builds only: which thread is traversing
```

**Invariants** — `traversing` is true exactly while a traversal callback may be running.
Adding a client or removing one asserts it is false — a mutation from inside a traversal is a
defect, not a supported operation, and a rebuild should make it impossible rather than
merely checked.

The roster is *only* the connected clients. A second collection for disconnected ones is
declared and never used; see the note below.

## `ForEachClientDo`

**Contract** — invokes the caller's action once per client, in roster order, with the lock
held for the whole sweep. The action may read and mutate the client records themselves; it may
not touch the roster. Two forms exist, one taking a direct callable and one taking an
indirected one, which is an artifact of the host language and not a distinction.

## `ForFoundClientsDo`

**Contract** — invokes the action once per client matching a predicate, lock held throughout.
Both the predicate and the action run under the lock. The count it claims to return is never
computed and is always zero; no caller reads it.

## `GetFoundClient`

**Contract** — returns the first client matching a predicate, or nothing. Takes the lock for
the search and releases it before returning.

**Invariants** — and here is the sharp edge: the lock is released, but the caller now holds a
*reference to a record inside the roster*. Nothing prevents the transport thread from
destroying that record before the caller uses it. Every call site in the engine is on the
simulation thread, where destruction is deferred, which is why this has never bitten — but it
is a real lifetime hole and a rebuild should close it, either by returning a copy of what the
caller needs or by keeping the record alive independently of its membership.

Notably this operation does **not** set the traversal flag, unlike every other one, so the
mutation assertion cannot catch a search that happens during a traversal.

## `FindAndEraseClient`

**Contract** — finds the first client matching a predicate, removes it from the roster and
returns it. The caller takes ownership. Asserts no traversal is in progress.

## `AddNewClient`

**Contract** — appends a client. Asserts no traversal is in progress. Order of arrival is
preserved, which matters in exactly one place: the first client admitted to a listen server is
that server's own.

## `ClientsCount`

**Contract** — the number of connected clients, read under the lock. Necessarily stale the
moment it returns; every caller uses it to size a buffer, which is safe because the buffer is
then filled under a traversal that cannot exceed it.

## Notes

Roughly half this file is a commented-out parallel roster of *disconnected* clients with its
own add, search, erase and traversal operations. The feature it served was reconnection with
preserved state — a player who drops and comes back finds their character where they left it.
It was designed and abandoned; what is left in the live code is a `reconnect` flag on the
client record that nothing sets. A rebuild wanting the feature is designing it fresh, and the
shape here is a hint, not a specification.

The debug-only "am I the thread that is traversing" query exists so the server can assert that
a caller reaching the roster indirectly is not already inside a traversal. It is a check on a
convention, and a rebuild whose types make re-entry impossible does not need it.
