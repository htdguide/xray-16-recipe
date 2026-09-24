# src/xrGame/profile_data_types.h

> The frozen vocabulary of a multiplayer player profile: the award set, the best-score set, and the shape of one award record.

**Needs** — [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`player_account.h`](player_account.h.md) · [`profile_data_types_script.cpp`](profile_data_types_script.cpp.md) · [`profile_data_types_script.h`](profile_data_types_script.h.md) · [`profile_store.h`](profile_store.h.md)
**Tier floor** — T3: two enumerations and a two-field record

## Purpose

A multiplayer player profile carried two kinds of persistent achievement: **awards**,
which are named badges with a count and a last-earned date, and **best scores**, which are
the high-water marks of a few streak counters. This file names both sets and nothing else.

It is a separate file because these identifiers are the *protocol* between the game, the
now-dead account service and the shipped Lua that renders a profile screen. The names and
their ordinal values are addressed by number on the script side, so the order of both
enumerations is frozen even though the service that filled them is gone.

## State

```text
ENUM AwardKind          # 30 values, ordinals 0..29, order frozen
  massacre, paranoia, overwhelming_superiority, blitzkrieg, dry_victory,
  multichampion, mad, achilles_heel, faster_than_bullets, harvest_time,
  skewer, double_shot_double_kill, climber, opener, toughy,
  invincible_fury, oculist, lightning_reflexes, sprinter_stopper, marksman,
  peace_ambassador, deadly_accuracy, remembrance, avenger, cherub,
  dignity, stalker_flair, lucky, black_list, silent_death
  # plus a trailing `count` sentinel that is itself exported to scripts

ENUM AwardField         # which column of an award row a query is asking for
  id, reward_date, count

ENUM BestScoreKind      # 7 streak counters, ordinals 0..6, order frozen
  kills_in_row, knife_kills_in_row, backstabs_in_row, head_shots_in_row,
  eye_kills_in_row, bleed_kills_in_row, explosive_kills_in_row
  # plus a trailing `count` sentinel

RECORD AwardRecord
  count            : int (16-bit)   # how many times this award was earned
  last_reward_date : int (32-bit)   # service-supplied timestamp; the engine never interprets it

AwardSet     : map<AwardKind, AwardRecord>     # ordered by key, iterated in enum order
BestScoreSet : map<BestScoreKind, int (32-bit)>
```

**Invariant** — both sets are *sorted associative* containers keyed by the enumeration, so
a script iterating them walks the awards in declaration order. That order is what the
profile screen's layout assumes.

**Notes**

- The engine attaches no meaning to `last_reward_date`; it is stored and displayed. A
  rebuild is free to treat it as an opaque integer.
- One award name is misspelled in the original (`fater_than_bullets`) and one enumerator is
  commented out with a duplicate ordinal. Neither is reachable from the scripts, which
  address only the first award and the count sentinel.
- The whole of this vocabulary depends on a service that stopped answering in 2014. The
  records still exist and are still exported to Lua, but they are never filled with
  anything but zeroes — see [`profile_store.cpp`](profile_store.cpp.md). A rebuild must
  keep the *shape* so the shipped profile screen still runs, not the service.
