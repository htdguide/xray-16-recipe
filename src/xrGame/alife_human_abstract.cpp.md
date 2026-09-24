# src/xrGame/alife_human_abstract.cpp

> The offline human: a server record that is simultaneously a creature and a trading party, and that forwards every behavioural question to its brain.

**Needs** — [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`alife_human_brain.h`](../xrServerEntities/alife_human_brain.h.md) · [`alife_human_object_handler.h`](alife_human_object_handler.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`relation_registry.h`](relation_registry.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: delegation and one ordering decision.

## Purpose

A human in the off-screen simulation is the engine's most composite record: a creature with
health and a position, a trader with money and an inventory, and a brain that decides. This
file is the seam where those three meet — mostly by forwarding, but the forwarding order
and the choice of which base handles which transition are real decisions.

## `update`

**Contract** — Skips inactive records entirely; otherwise runs the brain for one coarse
step. This is the whole of a human's off-screen behaviour.

## Delegation to the brain

**Contract** — Nine questions the simulation asks a creature are answered by the brain or by
its object handler, with no logic of their own here:

- **`bfPerformAttack`** — consume ammunition for one abstract shot and say whether another
  is possible.
- **`tfGetActionType`** — attack, interact or ignore, on meeting something.
- **`vfDetachAll`** — drop everything, optionally *fictitiously* (a bookkeeping-only detach
  used when items are about to be re-attached elsewhere and must not touch the world).
- **`vfUpdateWeaponAmmo`**, **`vfProcessItems`**, **`vfAttachItems`** — inventory upkeep.
- **`tpfGetBestDetector`**, **`tpfGetBestWeapon`** — what this character would search with,
  and fight with.

**Notes** — The best-weapon query declares two output parameters for hit type and power and
fills neither — it returns only the weapon. Callers that relied on the outputs get whatever
they passed in. That is a real inconsistency with the anomaly's implementation of the same
interface (see [`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md)), which fills the
outputs and returns no weapon. A rebuild should return one value describing both.

## `on_register`

**Contract** — Runs the trader base's registration, then immediately forces the character's
*profile* to be loaded.

**Invariants** — The profile is loaded eagerly rather than on demand because it supplies the
graph vertex masks — which regions of the cross-level graph this character is allowed to
travel through. Those masks are consulted by the simulation as soon as the record is in its
registries, so a lazy load would be too late. This is the whole reason the method exists.

## `spawn_supplies`

**Contract** — Forces the profile load, then runs *both* bases' supply spawning in order:
the creature side first, the trader side second.

**Invariants** — The order is load-bearing: the creature side spawns what the character is
as a creature, the trader side spawns what its profile says it trades. Reversing them would
let the profile's stock be filtered by equipment rules that have not yet run.

## `add_online` / `add_offline`

**Contract** — The promotion and demotion path goes through the **trader** base, not the
creature base — the same diamond resolution the player's record makes (see
[`alife_creature_abstract.cpp`](alife_creature_abstract.cpp.md)) — and then notifies the
brain, which drops or rebuilds whatever it derived from being live.

**Invariants** — The brain is notified *after* the base transition, in both directions. It
reads registry state that must already be correct.
