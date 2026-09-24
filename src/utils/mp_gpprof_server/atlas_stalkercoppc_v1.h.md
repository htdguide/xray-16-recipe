# src/utils/mp_gpprof_server/atlas_stalkercoppc_v1.h

> The generated registry that maps this game's multiplayer statistics onto the numeric identifiers the vendor's competition service knew them by.

**Needs** — [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — [`atlas_stalkercoppc_v1.c`](atlas_stalkercoppc_v1.c.md) · [`gamespy_sake.cpp`](gamespy_sake.cpp.md) · [`gamespy_sake.h`](gamespy_sake.h.md) · [`profile_data_types.cpp`](profile_data_types.cpp.md) · [`profile_data_types.h`](profile_data_types.h.md)

**Tier floor** — T4: it is a name-to-number table with no computation in it.

## Purpose

The vendor's competition service stored a player's history as numbered fields in a remote
table. The numbers were assigned when the game's rule set was registered with the service,
and neither end could choose them afterwards. This file is the record of that assignment,
generated from the registration and never edited by hand.

It declares two parallel vocabularies and the four lookups between their names and their
numbers. The distinction between them is the only load-bearing idea on the page:

- **Keys** are what a game *reports* at the end of a match — raw per-match measurements
  the client sends up.
- **Stats** are what the service *accumulates* — the persisted per-player totals a client
  or a tool reads back.

They are separate numbering spaces with overlapping names, and confusing one for the other
produces a request that succeeds and returns nothing. This tool only ever reads, so it
touches the stat space almost exclusively.

## State

```text
RECORD FieldRegistry              # fixed at registration time; not negotiable at run time
  rule_set_version : int          # 1
  keys  : map<text, int>          # per-match reported measurements
  stats : map<text, int>          # per-player accumulated totals
```

**Invariants**

- **The numbers are the wire format.** A name is a convenience for the code; the service
  only ever sees the number. Renumbering anything invalidates every stored profile.
- The stat space is numbered densely from a small base up to the player-name field, which
  is the last and is the only one of string type — everything else is an integer. The name
  field is what a search filters on; the rest are what it returns.
- The key space and the stat space are unrelated numberings. A name that exists in both
  has different numbers in each.
- Two names are misspelled in the registry — the "overwhelming superiority" award appears
  under a transposed spelling in the stat space, and the "faster than bullets" award is
  spelled without its first *s*. Both are frozen: the service knows them by those
  spellings. Correcting them in a rebuild breaks the lookups.

## Exported units

- `ATLAS_GET_KEY` / `ATLAS_GET_KEY_NAME` — number and name of a per-match reported field.
- `ATLAS_GET_STAT` / `ATLAS_GET_STAT_NAME` — number and name of a per-player accumulated
  field.
- `ATLAS_GET_STAT_PAGE_BY_ID` / `ATLAS_GET_STAT_PAGE_BY_NAME` — which of the service's
  stat groupings a field belongs to.
- `ATLAS_RULE_SET_VERSION` — the registration's version, sent with every request so the
  service can reject a client built against an older assignment.
- The `KEY_*` and `STAT_*` constants — the numbers themselves.

**Notes**

- A rebuild reproduces this file only if it is talking to a service that still uses these
  numbers, and none does: the vendor shut the service down in 2014. What survives is the
  *field list* — thirty awards each with a count and a last-earned date, and seven
  best-streak scores — which is the shape any replacement must carry. See
  [`profile_data_types.h`](profile_data_types.h.md), where that shape is written down
  independently of the numbering.
