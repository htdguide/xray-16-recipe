# src/xrGame/game_sv_mp_team.h

> The per-team tuning record: which skins a team may wear, what it spawns holding, and the full money-reward table that drives multiplayer economy.

**Needs** — _(none)_
**Used by** — [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T3: a data record loaded from configuration

## Purpose

Team-based multiplayer modes buy their balance entirely from configuration: every reward and
penalty in the game's economy is a signed amount in one record, one record per team, read
from the team's own configuration section at session creation. Keeping the record in its own
file lets both the session and the money-awarding code see it without depending on either.

## State

```text
RECORD Team
  section          : text        # the configuration section this team's numbers came from
  skins            : list<text>  # permitted player appearances; a joining player picks one
                                 # by index, so the ORDER is part of the protocol
  default_items    : list<int>   # class identifiers granted on every respawn

  # --- economy: all amounts are signed, so a "reward" may be a fine ---
  start            : int (signed)  # money at first spawn
  on_respawn       : int (signed)  # money granted at each respawn
  minimum          : int (signed)  # floor the balance is clamped to

  kill_rival       : int (signed)  # killing an enemy
  kill_self        : int (signed)  # killing yourself
  kill_team        : int (signed)  # killing a team-mate — normally negative

  target_rival     : int (signed)  # acting on the enemy objective
  target_team      : int (signed)  # acting on your own objective
  target_succeed   : int (signed)  # the player who completed the objective
  target_succeed_all : int (signed) # every member of the completing team
  target_failed    : int (signed)

  round_win        : int (signed)
  round_lose       : int (signed)
  round_draw       : int (signed)
  round_win_minor  : int (signed)  # won without completing the objective
  round_lose_minor : int (signed)

  rivals_wiped_out : int (signed)  # won by elimination rather than objective
  clean_run_bonus  : int (signed)  # survived the round without dying

  invincible_kill_modifier : real  # scales a kill reward when the victim was still
                                   # spawn-protected, so camping a spawn pays less
```

**Invariants** — the reward set is *per team*, not global: an asymmetric mode gives the two
sides different numbers for the same event, which is how the shipped modes balance an
attacking side against a defending one. A rebuild that hoists these into one global table
loses that.

The skin list is addressed by index over the network, so entries may not be reordered
without invalidating clients' stored preferences.

**Notes** — the team list is held as a sequence indexed by team number, with team numbering
starting at the first playing team rather than at zero in some modes; the index convention
belongs to the session, not to this record.
