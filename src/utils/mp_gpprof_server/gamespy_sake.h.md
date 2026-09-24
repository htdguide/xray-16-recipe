# src/utils/mp_gpprof_server/gamespy_sake.h

> Declares the session that logs in to the vendor's account service and queries its persistent-storage service for a batch of player profiles.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — [`gamespy_sake.cpp`](gamespy_sake.cpp.md) · [`sake_worker.cpp`](sake_worker.cpp.md) · [`sake_worker.h`](sake_worker.h.md)

**Tier floor** — T1 as written, because the vendor library requires fixed-layout request records and raw pointers into caller-owned text; the decisions are T2.

## Purpose

Declares the surface implemented in [`gamespy_sake.cpp`](gamespy_sake.cpp.md), and with it
the one shape that a rebuild has to preserve: **profiles are fetched in batches, not one at
a time.** The session accumulates names, issues one query for all of them, and hands back a
lookup table. Everything about the tool's threading and queueing exists to make that batch
big enough to be worth a round trip.

The session must be a singleton — the vendor library keeps global state — which is why
[`sake_worker.h`](sake_worker.h.md) gives it a thread of its own rather than locking it.

## Exported units

- `sake_processor` — one logged-in session. Constructing it logs in and fails hard if it
  cannot.
- `think` — give the session a time slice to advance its asynchronous work. Nothing
  happens between calls.
- `begin_fetch` / `add_name` / `fetch` — accumulate a batch and issue it.
- `is_result_ready` — whether the outstanding query has completed.
- `get_profile` — look one name up in the completed batch's results.

**Notes**

- The three-call batch protocol is a state machine with no guard: calling them out of order
  is undefined. A rebuild makes the batch a value and the query a function of it.
- The record holds the request's field-name array as a fixed-size block sized from the
  profile's shape — thirty awards times two fields, plus seven scores, plus the name
  column. That arithmetic is the join between the data model and the query, and it is the
  one thing in the header worth reproducing exactly.
