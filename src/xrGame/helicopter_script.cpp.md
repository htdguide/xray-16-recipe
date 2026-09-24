# src/xrGame/helicopter_script.cpp

> Exports the helicopter to the script virtual machine: four state enumerations, the flight and targeting commands, and the tuning fields a mission script writes directly.

**Needs** — [`helicopter.h`](helicopter.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data, plus three one-line state readers

## Purpose

Every helicopter in the shipped games is driven entirely from a mission script: nothing in
the engine decides that a helicopter should patrol, hunt or land. This file is that control
surface, and it is unusually wide for a `_script.cpp` sibling because the helicopter has no
autonomous behaviour to fall back on.

## State

`Stateless.`

## `GetMovementState`, `GetHuntState`, `GetBodyState`

**Contract** — each returns the current sub-state as an integer for comparison against the
corresponding exported enumeration. They exist as separate methods rather than field reads
because the sub-states are nested records the script layer does not see.

## `CHelicopter::script_register`

**Contract** — registers the helicopter as a game object with four nested enumerations and
the following surface. Names and signatures are frozen by conformance criterion 10.

**The four enumerations**, all nested so scripts address them as members of the class:

- **state** — alive, dead. The forcing sentinel is deliberately not exported.
- **movement_state** — none, to point, patrol path, round path, landing, take off.
- **hunt_state** — none, point, entity.
- **body_state** — by path, to point.

**Flight commands** — patrol along a named authored path from a starting index; orbit a
centre at a radius in a given direction; fly to a destination; point the nose at a point (or
stop doing so). Query the distance remaining to the destination, the current and maximum
speed, the velocity as a vector, the real altitude and the safe altitude.

**Flight tuning** — maximum velocity, the speed to be carrying on arrival, the forward and
backward linear accelerations (as a pair, because they are meaningless apart), and the
arrival tolerance radius.

**Targeting** — set the enemy as either a game object or a position (two entry points of the
same name, distinguished by argument), clear it, set the walking fire-trail length, set the
barrel direction tolerance, and read or write whether the fire trail is used (again one name,
two entry points, distinguished by whether an argument is supplied).

**Damage and death** — read and write the helicopter's health directly, die, start the flame,
explode. All three destruction steps are separately callable, because a mission may want the
burning flight without the explosion, or the explosion without the burn.

**Visibility** — whether a given game object is visible to the helicopter, which is how a
script gates an attack on line of sight rather than on distance alone.

**Presentation** — turn the lighting and the engine sound on or off, independently of state.

**Writable fields** — whether rockets and the machine gun are used on attack, the minimum and
maximum engagement distance for each, the delay between rocket attacks, and whether rocket
launches are synchronized. These are exposed as fields rather than through setters because a
mission script sets a whole profile at once at spawn time.

**Readable fields** — flame started, light started, exploded, dead. Read-only: these are
consequences, and a script that wants to cause one calls the corresponding command.

**Notes** — the health accessors are thin renames of the entity's own. They exist because the
entity's names are already bound on a base class the helicopter's script binding does not
derive from, so without them a script would have to hold two handles to the same object.

The overall state reader is exported under a different name than the method it wraps, and the
setter is not exported at all: a script may observe that a helicopter is dead but must kill
it through the death command, which runs the death sequence rather than just flipping a flag.
