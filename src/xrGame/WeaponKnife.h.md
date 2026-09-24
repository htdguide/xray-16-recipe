# src/xrGame/WeaponKnife.h

> Declares the knife implemented in [`WeaponKnife.cpp`](WeaponKnife.cpp.md).

**Needs** — [`Weapon.h`](Weapon.h.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)
**Used by** — [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`WeaponKnife.cpp`](WeaponKnife.cpp.md) · [`WeaponScript.cpp`](WeaponScript.cpp.md) · [`game_cl_deathmatch_buywnd.cpp`](game_cl_deathmatch_buywnd.cpp.md) · [`game_cl_mp.cpp`](game_cl_mp.cpp.md) · [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponKnife`, a melee weapon with two distinct attacks. Substance is in
[`WeaponKnife.cpp`](WeaponKnife.cpp.md).

The header carries one design fact worth stating here: the knife derives from the
**weapon base**, not from the magazined weapon, so it has none of the burst, reload or
addon machinery and runs its own small state machine. (It includes the semi-automatic
pistol's header without using it — residue.)

Exported units, by group:

**Configuration** — `Load` (the all-or-nothing splash parameter table, with the shipped
misspellings as fallbacks), `LoadFireParams` (the second attack's per-difficulty damage).

**State machine** — `OnStateSwitch`, `switch2_Idle`, `switch2_Showing`, `switch2_Hiding`,
`switch2_Hidden`, `switch2_Attacking`, `OnAnimationEnd`, `state_Attacking` (empty).

**Attacks** — `FireStart` (primary), `Fire2Start` (secondary), `Action` (the zoom binding
becomes the second attack), `OnMotionMark` and `OnKnifeStrike` (the animation-timed
blow), `KnifeStrike` (the three-way resolution), `MakeShot` (one hit, delivered through
the bullet path).

**Targeting search** (private; the substance of the file) — `SelectBestHitVictim`
establishes the search sphere and the candidate creatures; `SelectHitsToShot` ranks
skeleton shapes along the swing's axis; `fill_shapes_list`, `fill_shots_list`,
`make_hit_sort_vectors` and `create_victims_list` are its named steps; `TryPick` is the
first-hit ray query for something directly in front; `victim_filter` and
`best_victim_selector` are the two candidate-narrowing rules (keep all within range
versus keep the single nearest).

**Presentation** — `GetBriefInfo` (name and icon only), `IsZoomEnabled` (always false, so
the aim button is free for the second attack).

**Dead** — `GetVictimPos` and `m_SplashHitBone`: an abandoned "aim at a named bone"
approach.

All the buffers the search uses are stack-allocated with a computed capacity — the
targeting runs on a swing, which is frequent enough that a heap allocation per swing was
worth avoiding. A rebuild with a pooled or arena allocator gets the same property.
