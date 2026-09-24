# src/xrGame/game_object_space.h

> The numbered catalogue of every callback a game object can fire into script — the event vocabulary of the whole script layer.

**Needs** — _(none)_
**Used by** — [`xr_object.h`](../xrEngine/xr_object.h.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`CarDoors.cpp`](CarDoors.cpp.md) · [`CarWeapon.cpp`](CarWeapon.cpp.md) · [`GameObject.cpp`](GameObject.cpp.md) · [`GameObject.h`](GameObject.h.md) · [`GameTask.cpp`](GameTask.cpp.md) · [`HangingLamp.cpp`](HangingLamp.cpp.md) · [`HelicopterMovementManager.cpp`](HelicopterMovementManager.cpp.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`ai_crow.cpp`](ai/crow/ai_crow.cpp.md) · [`aimers_bone.h`](aimers_bone.h.md) · _and 21 more_
**Tier floor** — T3: one enumeration

## Purpose

One enumeration, alone in a header because nearly every file in the chapter fires one of its
values and almost none of them needs anything else about game objects. It is the list of
moments at which the engine will call a script function registered against an object, and it
is frozen by conformance criterion 10: shipped scripts bind by these names, and the *values*
are visible to script too, so both the names and the numbering matter.

## State

`Stateless.`

## `GameObject::ECallbackType`

**Contract** — the callback identifier a script registers a function against on a particular
game object. One object may hold at most one function per identifier. The engine fires them;
scripts never do. Grouped by what they report:

- **Trade** — a conversation with a trader opened and closed; an item was bought or sold; a
  trade operation was performed. Four separate points because the trade screen can be
  refused at any of them.
- **Zones and borders** — an entity entered or left an anomaly, and entered or left the
  level's boundary volume. The border pair is how the engine hands off an entity that walks
  off the edge of the loaded level to the alife simulation.
- **Death** — the entity died.
- **Patrol** — a creature reached a point on its patrol path.
- **Knowledge** — a data-device article was received, an information token was gained or
  removed, an article was read, a task changed state, a map marker appeared. These drive the
  plot layer: the game's whole quest system is built on the information-token pair.
- **Use** — somebody used this object.
- **Hit** — this object was damaged; the handler receives the full hit description and may
  modify it.
- **Sound** — this object heard a sound, with the emitter's perception attributes.
- **Action completion** — seven of them, one per action channel a script can drive a creature
  through: movement, watching, animation, sound, particles, object interaction, and the
  removal of an action. A script that scripts a creature step by step waits on these.
- **Sleep** — the actor slept.
- **Helicopter** — reached a waypoint, or was hit.
- **Item flow** — an item was taken, dropped, taken from a container; an item moved to the
  belt, to a slot, or to the rucksack.
- **Weapon** — no ammunition available, zoom in, zoom out, jammed, magazine empty.
- **Actor** — about to die (fired before death, so a script can prevent it), and the
  first-person animation ended.
- **Vehicle** — attached, detached, used.
- **Trader animation** — the trader's screen asks the game layer for a body animation, a head
  animation, or reports a line of speech finishing.
- **Script animation** — a scripted animation finished.
- **Controller** — press, release, hold and an attitude change, for the bound-control layer.
- **Raw input** — key press, release and hold, mouse wheel and mouse move.

**Invariants** — the raw-input group is numbered **explicitly at 123 to 127**, far above the
dense run, while everything else is implicitly numbered from zero. That gap is deliberate:
the sequentially numbered values grew over three games and several mod projects, and pinning
the input callbacks high keeps them stable while values are appended in the middle. A rebuild
must reproduce both the dense run's order and the five fixed numbers, because scripts compare
against the numeric values.

The terminator is the all-ones value, used as "no callback" rather than as a count. There is
no count, so a rebuild cannot size an array by the enumeration's last member.

**Notes** — the sequence records its own history: three distinct later additions sit at the
end of the dense run rather than beside the callbacks they belong with, because inserting
them in place would have renumbered everything after. The grouping above is by meaning; the
numbering is by date. Preserve the numbering.
