# src/xrGame/stalker_combat_actions.h

> Declares every leaf action a stalker can take in a firefight.

**Needs** — [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md) · [`stalker_combat_actions_inline.h`](stalker_combat_actions_inline.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_combat_actions_inline.h`](stalker_combat_actions_inline.h.md) · [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_danger_by_sound_actions.h`](stalker_danger_by_sound_actions.h.md) · [`stalker_danger_grenade_actions.h`](stalker_danger_grenade_actions.h.md) · [`stalker_danger_in_direction_actions.h`](stalker_danger_in_direction_actions.h.md) · [`stalker_danger_unknown_actions.h`](stalker_danger_unknown_actions.h.md) · [`stalker_get_distance_actions.h`](stalker_get_distance_actions.h.md) · [`stalker_kill_wounded_actions.h`](stalker_kill_wounded_actions.h.md)
**Tier floor** — T2: eighteen action objects per creature, built once at brain setup.

## Purpose

Declares the surface implemented in
[`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md). All derive from the combat
action base, so all have the firing gate, the burst selector, the voice lines and the cover
helper.

## Exported units

Arming:

- `GetItemToKill` — walk to a weapon that was spotted on the ground.
- `MakeItemKilling` — walk to ammunition for the weapon already held.

Fighting from cover — the core loop:

- `GetReadyToKill` — move onto the chosen cover point and raise the weapon. Takes a
  construction flag deciding whether it also resets the cover-sequence propositions; the
  planner instantiates it twice, once each way.
- `TakeCover` — the same move, entered from a different point in the sequence.
- `KillEnemy` — shoot at the enemy from where the creature stands.
- `LookOut` — shift to somewhere with a line of sight. Carries a timestamp and a private
  random source that together rate-limit the crouch-or-stand choice.
- `HoldPosition` — wait, and hand the sequence on when the squad allows.
- `DetourEnemy` — flank.

Special cases:

- `RetreatFromEnemy` — run away. Overrides the operator weight.
- `HideFromGrenade` — abandon everything and get behind something.
- `SuddenAttack` — approach an enemy that has not noticed you.
- `KillEnemyIfPlayerOnThePath` — shoot even though the player is in the line of fire.
- `CriticalHit` — the stagger after a critical wound.
- `PostCombatWait` — the few seconds after the last enemy is gone.
- `CombatActionThrowGrenade` — throw a grenade. Remembers which grenade, so that it can
  tell when the throw completed.
- `CombatActionSmartCover` — fight from an authored smart cover. Saves and restores one
  movement-layer flag.

Declared and never defined:

- `GetDistance` and `SearchEnemy` — no implementation exists anywhere in the source, and
  nothing constructs them. The behaviours they name were moved into sub-planners of their
  own. A rebuild should omit both.
