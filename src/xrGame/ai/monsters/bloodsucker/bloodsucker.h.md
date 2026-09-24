# src/xrGame/ai/monsters/bloodsucker/bloodsucker.h

> Declares the bloodsucker: the creature base plus invisibility, a life-draining grab, and the ability to take the player's camera away entirely.

**Needs** — [`bloodsucker.cpp`](bloodsucker.cpp.md) · [`bloodsucker_alien.h`](bloodsucker_alien.h.md) · [`basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`ai_monster_bones.h`](../ai_monster_bones.h.md) · [`anim_triple.h`](../anim_triple.h.md) · [`controlled_entity.h`](../controlled_entity.h.md) · [`controlled_actor.h`](../controlled_actor.h.md)
**Used by** — [`bloodsucker.cpp`](bloodsucker.cpp.md) · [`bloodsucker_alien.cpp`](bloodsucker_alien.cpp.md) · [`bloodsucker_attack_state_inline.h`](bloodsucker_attack_state_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](bloodsucker_predator_lite_inline.h.md) · [`bloodsucker_script.cpp`](bloodsucker_script.cpp.md) · [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md) · [`bloodsucker_vampire_approach_inline.h`](bloodsucker_vampire_approach_inline.h.md) · [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md) · [`script_game_object_use2.cpp`](../../../script_game_object_use2.cpp.md)
**Tier floor** — T2: the creature base plus a visual swap and a camera takeover

## Purpose

Declares the surface implemented in [`bloodsucker.cpp`](bloodsucker.cpp.md). It is the most
elaborate of the creatures and the clearest case of "a creature is the base plus one or two
distinctive abilities" — except that the bloodsucker has four.

## Exported units

**The four abilities**

- **Invisibility** — the creature swaps to a second *visual* (a different model with a
  refractive material) rather than becoming untextured. It also carries its own movement
  velocity pair for the invisible state.
- **Visibility states** — three: none, partial, full. Which one applies is a function of
  distance to the enemy and of a runaway timer, with a minimum dwell time between changes.
  The creature is not rendered at all in the *none* state.
- **The vampire grab** — a three-phase animation that pins the player, drains health into the
  creature and runs a dedicated camera and post-process effector. Gated by a want value that
  fills over time, a global cooldown shared by *every* bloodsucker, and a count of successful
  running-attack hits.
- **Alien control** — the creature takes over the player's camera entirely: the player sees
  through the bloodsucker's head, at the bloodsucker's speed-dependent field of view, with the
  weapon holstered and the crosshair hidden.

**Bone control** — the creature registers additive rotation channels on its spine and head
(see [`ai_monster_bones.h`](../ai_monster_bones.h.md)), with a bone callback wired into the
animation layer.

**Ability answers** — invisibility yes, dragging yes, pitch correction no. The last means
the creature does not tilt to match the ground, which matters for a model this tall.

**The critical hit** — a probability per melee blow; a critical multiplies the impulse
tenfold and is remembered so the attack state can break off immediately afterwards.

**The drag jump** — the scripted grab-and-leap: a captured entity, a bone name to grab it
by, a landing position and a jump factor, plus the two flags that sequence the animation.

**Sounds** — seven of its own beyond the shared set: vampire grasp, vampire sucking, vampire
hit, start of hunt, growl, visibility change, and the alien-control drone.

**Script surface** — one method, the forced visibility state.
