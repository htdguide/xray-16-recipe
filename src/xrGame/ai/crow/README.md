# src/xrGame/ai/crow — ambient birds

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
See the [chapter opener](../README.md) for the machinery every other creature shares; the
crow shares almost none of it.

The crow is decoration that can be killed. It steers toward a goal point re-rolled every few
seconds in the air above the player, caws on its own cadence, and if anything hits it at all
it dies, becomes a ragdoll, falls, and on landing turns into a corpse that creatures and
scripts can see. It exists so that the sky is not empty and so that an idle player has
something to shoot at.

## What makes it different

It is the only entity in this chapter that is **not** built on the creature base. No
memories, no perception, no enemies, no navigation-mesh position, no state manager, no pack.
Its mind is a small state identifier advanced by a switch, and its movement is an integrated
heading and speed rather than a route through the level graph.

That is worth saying plainly, because a rebuilder reading the rest of the chapter will
expect the crow to be a thin creature. It is not a creature at all. It is the control group:
what an entity costs when you remove everything chapter 24 exists to provide.

Three of its decisions are still load-bearing.

**It is tuned entirely from its configuration section** — speed, turn rate, how often the
goal is re-rolled, how far above the player the goal sits, how much the goal is jittered,
and the calling cadence. The same claim the chapter makes about creatures holds here in its
purest form: the code is a flight integrator and every number is data.

**Any hit kills it.** There is no damage model; the hit path does not compare amounts. That
is what makes a crow feel like scenery rather than like an enemy with one hit point.

**Its corpse rejoins the world.** While flying it is outside the set of objects creature
perception considers; on landing it re-enters, so a dead crow is a thing that can be found,
searched and reacted to. The transition from decoration to entity happens exactly once and
in one direction.

## What could not be recovered

- The flock's visual cadence — the goal re-roll period, the height offset and the jitter box
  — ship as defaults in code *and* are overwritten from configuration, so the code defaults
  are only reachable through an incomplete section. Whether they were ever the shipped
  values is not recoverable.

| Twin | Role |
|---|---|
| [`ai_crow.cpp`](ai_crow.cpp.md) | The crow: ambient flying decoration that circles the player, can be shot down, and becomes a lootable corpse when it lands. |
| [`ai_crow.h`](ai_crow.h.md) | Declares the crow: a flying ambient creature with four states, no perception and no navigation. |
