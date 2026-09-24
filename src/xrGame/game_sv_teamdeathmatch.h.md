# src/xrGame/game_sv_teamdeathmatch.h

> Declares team deathmatch: deathmatch plus two teams, a shared score, friendly fire, team balancing and a team base.

**Needs** — [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md)
**Used by** — [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md) · [`game_sv_teamdeathmatch.cpp`](game_sv_teamdeathmatch.cpp.md) · [`game_sv_teamdeathmatch_process_event.cpp`](game_sv_teamdeathmatch_process_event.cpp.md) · [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`game_sv_teamdeathmatch.cpp`](game_sv_teamdeathmatch.cpp.md)
and [`game_sv_teamdeathmatch_process_event.cpp`](game_sv_teamdeathmatch_process_event.cpp.md).
Type name `"teamdeathmatch"`.

The one piece of state declared here that is not in the base: a flag recording whether the
teams have already been swapped this map, which is what stops the auto-swap from swapping
back and forth every round.

Exported units, grouped by what they decide:

- **Session** — create, update, read options, export state to a joining client, load the
  team configuration.
- **Membership** — auto-assign a team on connect, respond to a team-change request, swap the
  two teams between rounds, rebalance unequal teams, count players per team.
- **Scoring** — classify a kill (rival or team-mate), apply the classification, roll a
  player's frag delta into the team score, decide whether the frag limit is reached and who
  has won.
- **Damage** — the friendly-fire scaling applied before a hit reaches the victim.
- **Round** — start, end, the frag- and time-limit endings.
- **Items** — the drop-bag pickup and drop rules, which in this mode replace ordinary
  ownership transfer.
- **Team base** — enter and leave notifications for the authored base volume.
- **Queries for the client** — friendly indicators, friendly names, friendly-fire factor,
  auto-balance, auto-swap, team-kill limit and punishment. All read server settings; they
  exist as methods so a derived mode can override the policy rather than the setting.
