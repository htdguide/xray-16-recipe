# src/utils/mp_gpprof_server/profile_data_types.cpp

> The three tables that join the game's award vocabulary to the service's field numbering, and the lookups in both directions.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T3: three parallel tables and linear searches over them.

## Purpose

[`profile_data_types.h`](profile_data_types.h.md) says what a profile *is*. This file says
what each of its fields was called on the wire, in two directions, so a request can be
built from the record's shape and a response can be decoded back into it.

It is three tables, and the fact that they are three parallel arrays indexed by the same
enumeration — rather than one array of records — is the only structural decision in it.
That choice is incidental; a rebuild uses one table per field with three columns.

## State

```text
RECORD FieldMapping                 # one row per award, in enumeration order
  published_name   : text           # the game's own name, prefixed as a multiplayer award
  count_field      : int            # the service's number for the occurrence count
  date_field       : int            # the service's number for the last-earned date

RECORD ScoreMapping                 # one row per streak, in enumeration order
  published_name   : text
  value_field      : int
```

**Invariants**

- **All three arrays are indexed by the enumeration and must stay in its order.** There is
  no key and no check; a row inserted in the wrong place silently reports one award's
  count under another's name.
- **The remote table name is frozen**: a profile lives in a table named for the statistic
  set and its version. Changing the version means a different table, not a migration.
- The published names carry a prefix marking them as multiplayer fields, and that prefix is
  what the game's user interface strings key on. It is part of the name, not decoration.
- The service-side names used for *decoding* are recovered by asking the registry for a
  number's name, not by storing a second copy. That keeps one spelling authoritative even
  where it is misspelled — see [`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md).

## `get_award_name`, `get_best_score_name`

**Contract** — index to published name. The index must be in range; out of range is a
programming error, not an input, and is asserted rather than handled.

## `get_award_id_stat`, `get_award_reward_date_stat`, `get_best_score_id_stat`

**Contract** — index to the service's field number for that field's count, date or value.
Same range discipline.

## `get_award_by_stat_id_name`, `get_award_by_stat_rdate_name`, `get_best_score_type_by_sname`

**Contract** — the reverse: given a field name as it came back from the service, which
award or streak it denotes and which of its two halves. Return the enumeration's count
sentinel when the name belongs to neither, which is how a caller distinguishes "this is
the count column", "this is the date column", "this is the streak column" and "this is
something else" — the decode loop in [`gamespy_sake.cpp`](gamespy_sake.cpp.md) tries all
three in turn and falls through to the name column.

```text
FUNCTION award_for_count_field(name) -> index or none
  FOR index IN 0 .. award_count - 1
    IF name == registry_name_of(mapping[index].count_field) THEN RETURN index
  RETURN none
```

**Notes**

- Each lookup is a linear scan that resolves a number to a name *inside the loop*, so
  decoding one response field costs a scan of the registry per candidate row. With thirty
  awards and a hundred-odd responses it never mattered; a rebuild inverts the table once
  at startup and looks up in constant time.
- Returning the enumeration's terminating sentinel as "not found" means the caller's test
  is a range comparison rather than an equality — which is why every caller writes
  `if result < count` rather than `if result != none`. That is the idiom, not a decision.
