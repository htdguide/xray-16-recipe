# src/xrGame/ai/monsters/bloodsucker/bloodsucker_attack_state.h

> Declares the bloodsucker's own attack composite — the one that would weave feeding and cloaked withdrawal into the shared attack tree — together with the "get behind him" approach state it owns. Nothing registers it.

**Needs** — [`monster_state_attack.h`](../states/monster_state_attack.h.md) · [`state.h`](../state.h.md) · [`state_data.h`](../states/state_data.h.md) · [`bloodsucker_attack_state_inline.h`](bloodsucker_attack_state_inline.h.md)
**Used by** — [`bloodsucker_attack_state_inline.h`](bloodsucker_attack_state_inline.h.md) · [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md)
**Tier floor** — T3: declarations over the shared state contract

## Purpose

Declares the surfaces implemented in [`bloodsucker_attack_state_inline.h`](bloodsucker_attack_state_inline.h.md): a bloodsucker-flavoured replacement for the shared attack composite, and a leaf approach state that tries to reach the enemy's *back* rather than the shortest route to him.

Read this file together with [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md), which registers the generic attack composite instead and leaves the line that would register this one commented out. Everything declared here is therefore unreachable in the shipped build. It is kept in the recipe because it is the most complete statement anywhere of what the creature's combat was *meant* to be, and because a rebuild choosing to enable it needs the contract.

## `BloodsuckerAttackState`

The creature's attack composite. It extends the shared attack composite with two extra substates — the feed and the back-approach — and overrides the substate selector, the entry and both exits, and the parameter fill.

Its own fields are a cloak-expiry stamp (declared and written, never read), a remembered direction point (declared, never used), the health level the last withdrawal decision was taken at, and a flag saying the next withdrawal should begin by circling.

## `BackstabEnemyState`

A leaf approach state: run at the enemy, optionally arriving oriented to match *his* facing so that the creature ends up behind him. Its parameter record extends the shared "move to point with path options" record with one extra field, "begin by circling".

Its own fields are the health level the last behaviour flip was taken at, whether circling is currently on, the tick circling should stop at, and the earliest tick the next flip is allowed.

**Notes** — the original spells the class *Backstub*; read it as *backstab*. The recipe uses the intended word.
