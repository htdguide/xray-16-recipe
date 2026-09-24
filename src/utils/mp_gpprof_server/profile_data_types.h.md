# src/utils/mp_gpprof_server/profile_data_types.h

> The shape of a player's multiplayer record — thirty awards with a count and a last-earned date each, and seven best-streak scores — independent of how any service stores it.

**Needs** — [`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — [`gamespy_sake.cpp`](gamespy_sake.cpp.md) · [`gamespy_sake.h`](gamespy_sake.h.md) · [`profile_data_types.cpp`](profile_data_types.cpp.md) · [`profile_printer.h`](profile_printer.h.md) · [`profile_request.cpp`](profile_request.cpp.md) · [`profile_request.h`](profile_request.h.md) · [`profiles_cache.cpp`](profiles_cache.cpp.md) · [`profiles_cache.h`](profiles_cache.h.md)

**Tier floor** — T3: it is a record of counters and the names they are published under.

## Purpose

This is the page to keep. Everything else in this directory is plumbing for a service that
no longer answers; **this file is the data model**, and it is stated here in terms that
outlive the service: what a player's multiplayer history consists of, what each piece is
called when published, and how wide each piece is.

It is substantive rather than a declaration header because it is where the two vocabularies
meet: the game's own names for these things — the ones that appear in the game's
configuration and in the published output — and the service's numbering, which
[`profile_data_types.cpp`](profile_data_types.cpp.md) maps onto.

## State

```text
RECORD AwardEntry
  count            : int (16-bit)    # how many times the player earned it
  last_reward_date : int (32-bit)    # when they last did; see Invariants

RECORD ProfileData
  awards       : list<AwardEntry>    # exactly 30, indexed by the award enumeration
  best_scores  : list<int (32-bit)>  # exactly 7, indexed by the streak enumeration
```

**Invariants**

- **The two lists are fixed-length and index-addressed, not keyed.** An award's identity is
  its position in the enumeration; the name is presentation. That is what makes a whole
  profile a flat block of counters with no allocation, which in turn is what makes the
  cache in [`profiles_cache.cpp`](profiles_cache.cpp.md) able to size itself in records.
- **A count is half as wide as a date.** The count is a small number of occurrences and
  the date is an absolute timestamp, and mixing them up truncates a date to its low half.
  The widths are the service's, not a choice.
- **The date's epoch and encoding are not recoverable from this repository.** Nothing here
  interprets the value — it is fetched, cached and printed as an integer, end to end. A
  rebuild publishing the same field must decide what it means; the original never did.
- A zero count means "never earned". There is no separate absent state, and a default
  profile is all zeros — which is exactly what a player who has never played looks like.
  The service cannot distinguish "no record" from "an empty record", and neither can this.

## The award vocabulary

Thirty awards, in the enumeration's order, each published under a name prefixed to mark it
as a multiplayer award: massacre, paranoia, overwhelming superiority, blitzkrieg, dry
victory, multichampion, mad, achilles heel, faster than bullets, harvest time, skewer,
double shot double kill, climber, opener, toughy, invincible fury, oculist, lightning
reflexes, sprinter stopper, marksman, peace ambassador, deadly accuracy, remembrance,
avenger, cherub, dignity, stalker flair, lucky, black list, silent death.

**Invariants**

- **The order is frozen** because it is the index, and the index is what every array here
  is addressed by.
- The published names are the game's own and appear verbatim in the game's user interface
  strings, so they are the join between this record and everything a player sees. They are
  spelled correctly here even where the service's numbering misspells them — the
  correction lives in the mapping, not in the vocabulary.

## The best-score vocabulary

Seven streaks, each the longest run of a kind of kill a player ever achieved in one match:
kills, knife kills, backstabs, head shots, eye kills, bleed kills, explosive kills.

**Invariants**

- A best score is a *maximum*, not a total, which is why it is one number and has no date.
  Nothing in this repository computes it; the service did, from what the game reported per
  match.

## Exported units

- `enum_awards_t` — the thirty awards, in index order, terminated by a count sentinel used
  as the "not an award" answer.
- `enum_best_score_type` — the seven streaks, likewise.
- `enum_award_params` — the two fields an award has, which is also the width of each row in
  the numbering table.
- `award_data`, `profile_data` — the records above.
- `get_award_name`, `get_best_score_name` — the published name of a field.
- `get_award_id_stat`, `get_award_reward_date_stat`, `get_best_score_id_stat` — the
  service's number for a field.
- `get_award_by_stat_id_name`, `get_award_by_stat_rdate_name`,
  `get_best_score_type_by_sname` — the reverse: which field a service-side name denotes.
- `profile_table_name` — the remote table a profile lives in.

**Notes**

- A rebuild that keeps multiplayer keeps this record and the two vocabularies, and
  replaces everything that maps them onto a numbering. The record is the product; the
  numbering was a shipping detail.
