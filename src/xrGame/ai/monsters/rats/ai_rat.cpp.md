# src/xrGame/ai/monsters/rats/ai_rat.cpp

> The rat's lifecycle: how it is loaded, spawned into a group, joined to its nest's shared activity budget, serialised for the network, turned into a corpse, and handed back and forth between being a creature and being an item.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`ai_rat_impl.h`](ai_rat_impl.h.md) · [`ai_rat_space.h`](ai_rat_space.h.md) · [`../../../rat_state_manager.h`](../../../rat_state_manager.h.md) · [`../../../rat_states.h`](../../../rat_states.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../../../../xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../../../../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`../../../../xrAICore/Navigation/game_graph.h`](../../../../xrAICore/Navigation/game_graph.h.md) · [`../../../../xrAICore/Navigation/PatrolPath/patrol_path.h`](../../../../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [Seam: Networking transport](../../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ai_rat.h`](ai_rat.h.md)
**Tier floor** — T1: builds a physics body from an explicit box, and writes a fixed-layout network record

## Purpose

Most of this file is the cost of the rat being two things at once — a creature and an edible
item — and of it being outside the shared monster base. What is genuinely load-bearing:
the brain's construction, the two-source parameter split, the group budget it joins on spawn,
and the corpse's transition into a physics object.

## `init` and `reinit`

**Contract** — `init` clears every runtime field to a defined start. `reinit` runs it, then
re-runs both parents' reinitialisation and rebuilds the brain from scratch. Called on spawn and
on every save load.

**Notes** — `init` seeds the wander goal with a **uniform random point in a twenty-unit cube
centred on the origin**, and sets the current heading to point at it. That is a placeholder,
not a destination: the first goal change replaces it, and its only real effect is that a rat's
very first turn is in a random direction rather than always the same one. A rebuild may seed
it from the rat's own facing instead and lose nothing.

The default time between bites is seeded to two seconds and is then overwritten from the spawn
record; the seeded value is only visible if the spawn record is malformed.

## `init_state_manager`

**Contract** — allocates the brain, binds it to this rat, registers one instance of each of the
twelve states against its identifier, and pushes the wandering state as the initial one. Called
from `reinit`, so a save load rebuilds the whole brain rather than restoring it.

**Invariants** — the brain is a **stack**, and this is where the bottom of it is established.
Every diversion afterwards pushes, and every resumption pops back to whatever was underneath.
Because `reinit` rebuilds from empty, **a rat's stack does not survive a save**: a rat that was
three diversions deep resumes as a plain wanderer. Whether that was intended is not recoverable,
but it is observable, and a rebuild that serialises the stack changes behaviour across saves.

## `Load` — the tuned parameters

**Contract** — reads the creature's configuration section. Every key is mandatory. Runs `init`
first, so a reload cannot inherit stale runtime state. Registers nothing and allocates nothing
beyond what the sound table below needs.

**Notes** — the two range conversions here are the interesting detail: the wall-turn bounds and
all four angular speeds are **authored in degrees and stored in radians**, because designers
tune turn rates in degrees. Nothing else in the rat's section is unit-converted.

The active update interval is not authored: it is read back from whatever the scheduler's base
already assigned this object, so "active" means the rat's normal rate and "passive" means the
authored slower one. Only the slow rate is tunable, which is the right way round — it is the
one that decides how cheap a large nest is.

A stray statement at the top of the routine perturbs the rat's recorded position by a random
sub-unit offset in two axes and then discards the result. Dead.

## `reload` — the sound table

**Contract** — registers five sounds against the rat's head bone, each with a kind, a priority,
a selection mask and an identifier. Called separately from `Load` so that sounds can be rebuilt
without re-reading the numbers.

**Notes** — the masks come from [`ai_rat_space.h`](ai_rat_space.h.md) and decide which sounds
may interrupt which; see that page, because the encoding is not obvious.

## `net_Spawn` — joining the world and the nest

**Contract** — brings the rat online from its authoritative spawn record. Fails if either parent
refuses. Registers the rat with the squad manager, copies the spawn record's parameters onto
itself, establishes its home anchor and its navigation vertices, joins the group's activity
budget, loads its animations, and picks up an authored patrol path if the spawn record names
one.

```text
FUNCTION net_Spawn(record) -> bool
  IF NOT base.net_Spawn(record) OR NOT edible_item.net_Spawn(record)
    RETURN false

  squad_manager.register(team, squad, group, self)

  body.yaw = -record.torso_yaw ; body.pitch = 0 ; body.turn_rate = one turn per second
  copy from record: eye field-of-view and range, health, the three speeds,
                    pursuit and home radii, the four morale quanta and three bounds,
                    hit power and interval, attack distance and angle
  # note: the attack angle is authored in degrees and converted here, like Load's turn rates

  current_graph_point = next_graph_point = my coarse vertex
  IF any of my permitted terrain types matches this coarse vertex's type
    graph_point_change_at = now + uniform(60_000 .. 120_000)   # the nest migrates on this clock

  speed = max_speed

  home_position = IF alive THEN my squad leader's position ELSE my own position
  push_state(free_active)
  IF alive
    join_activity_budget(forced: true)     # the first rat in is always granted active

  my coarse vertex = cross_table(my mesh vertex)   # re-derived, authoritatively
  orientation = body orientation
  disable the movement manager                     # the rat steers itself; see ai_rat_templates
  load_animations()

  IF alive
    group.last_action_time = 0

  mark self as not takeable                        # only a corpse can be picked up

  IF the spawn record carries a patrol path name
    patrol_path = patrol_registry.path(that name)
    current_way_point = 0
```

**Invariants**

- **The movement manager is explicitly disabled.** Every other creature in the game routes its
  motion through it; the rat is the exception and integrates its own position. That single line
  is why [`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) exists at all.
- **The home anchor is the squad leader's position, not the rat's own.** A nest spawned across
  a room converges on one anchor immediately.
- **The migration clock is one to two minutes, uniformly.** It only starts if the rat's
  permitted terrain types include the terrain it spawned on — an unreachable-terrain rat never
  migrates. The two-minute span is compiled in and has no recoverable derivation.
- Parameters arrive from **two** places: the configuration section (shared by every rat) and the
  spawn record (per-instance). Speeds, senses, damage and the two radii are per-instance, which
  is what lets one level's rats be more dangerous than another's without a second section.

## `net_Export` and `net_Import` — the network image

**Contract** — write and read the rat's replicated state. Export asserts the object is locally
authoritative and that an update exists to send; import asserts it is remote. Fixed field
order, frozen by the protocol.

**Notes** — the record carries health, a timestamp, a flags byte, position, model and torso
orientation, the three hierarchy identifiers, and then **two copies of the coarse vertex
identifier followed by two copies of the distance from the rat to that vertex's world point**.
Import reads both copies of the vertex and keeps the second, and does not read the distances at
all. The duplication is a protocol fossil — a pair of fields that once held a current and a
next value, collapsed when the migration logic moved — and the distances are computed, written
and never consumed. They cannot be removed without changing the wire format, which is frozen.

Orientation is quantised to a byte on import but written at full precision on export, which is
an asymmetry the transport tolerates because the reader is the only one that has to agree with
itself.

## `CreateSkeleton` — the corpse becomes physical

**Contract** — builds the rat's physics body: a single box element, offset forward and slightly
up from the origin, with the density taken from the authored corpse mass and the surface
material read from the visual's root bone. Activates it at the rat's current transform. If a
hit had been recorded before the body existed, replays that hit into the new body so the corpse
flies the way the shot that killed it implied.

**Invariants** — the deferred hit replay is the load-bearing part. A rat killed by a shot has
no physics body at the instant of the shot; saving the impulse, direction, position and damage
kind and applying them at body creation is what makes a shot rat tumble instead of dropping
straight down. See `Hit` below for the other half.

**Notes** — the box's half-extents and offset are compiled in and describe the rat's body
roughly; a sphere alternative sits commented out beside them. The material is inherited from
the model rather than named, so a retextured rat sounds and slides correctly without a code
change.

## `Hit`

**Contract** — routes damage. Always informs the creature side. Then, **if there is no physics
body yet**, records the impulse, direction, position and damage kind for `CreateSkeleton` to
replay; otherwise forwards the hit to the item side so the existing body responds immediately.

**Invariants** — exactly one of the two branches runs, so a hit is never both saved and
applied. Only the *last* pre-body hit is kept; a rat shot twice before dying flies according to
the second shot.

## `Die`

**Contract** — marks the rat dead and takeable, switches it to the death state directly,
re-selects its animation immediately so the death clip starts on the same tick, plays the death
sound, broadcasts a morale penalty to the whole group, leaves both activity budgets, and
decrements the group's living count.

**Invariants** — the state is assigned directly rather than pushed, and the stack is left as it
was. Because the brain is rebuilt on reinitialisation anyway, nothing ever unwinds it.

**Notes** — the morale broadcast is the nest's alarm: see
[`ai_rat_impl.h`](ai_rat_impl.h.md). Its radius parameter is accepted and ignored — the
broadcast reaches the whole group regardless of distance.

## `UpdateCL` and `shedule_Update`

**Contract** — `shedule_Update` runs on the rate-degraded schedule and, before anything else,
elects this rat as squad leader if the squad has none or its leader is dead. `UpdateCL` runs
per frame and branches on whether the rat is currently an item: a live rat runs the creature
path (look, leader election, squad index refresh) and a corpse runs the item and physics path.

**Invariants**

- **Leader election is opportunistic and happens in two places** — here and in the per-frame
  update — because either may run first depending on scheduling. The rule is the same in both:
  a rat with no leader, a dead leader, or no index in its own squad elects itself. A nest whose
  leader is killed therefore re-anchors within a tick, and the home anchor moves to the new
  leader's position.
- The squad's combat fan-out indices (which give each rat a different approach angle; see
  [`rat_state_switch.cpp`](rat_state_switch.cpp.md)) are recomputed only when the living count
  changes, not every tick.

## The dual-parent lifecycle dispatch

**Contract** — a dozen routines — attachment and detachment from a container, physics
correction and prediction phases, interpolation, serialisation, construction — exist only to
call both parents in the correct order. Two of them do real work: detaching from a container
makes the rat visible and re-activates its physics body, and attaching hides it and deactivates
the body.

**Notes** — this is the largest block of purely incidental code in the rat's files. The problem
it solves is that one object is simultaneously a simulated creature and an inventory item, and
both parents want the same lifecycle callbacks. A rebuild that composes the two facets rather
than inheriting both deletes nearly all of it; the two lines that hide and show the object are
the only decisions.

`create_physic_shell` and `setup_physic_shell` are deliberately **empty overrides** marked "do
not delete", because the base would otherwise build a body the rat does not want — the rat
builds its own, later, only when it dies.

## `Useful`

**Contract** — answers whether the rat is currently an item worth interacting with. A living rat
is never useful; a dead one defers to the item facet, which weighs its remaining food value.
This single predicate is what switches the per-frame update between its two paths.

## `get_custom_pitch_speed`

**Contract** — reports how fast the rat may pitch its body, chosen by which of its four discrete
speeds it is currently at: stationary turns slowest, attack speed fastest, walking and running
in between. **Fails fatally on any other speed.**

**Invariants** — this asserts the rat's speed is always exactly one of four authored values,
never interpolated. That is true because
[`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) only ever assigns from that set — and it is a
brittle contract: the comparison is a float similarity test, and any rebuild that eases speed
changes rather than stepping them crashes here. A rebuild should map speed to pitch rate
continuously.

## `renderable_ShadowReceive` / `renderable_ShadowGenerate`

**Contract** — a rat receives shadows but casts none. A deliberate cost decision: rats appear in
large numbers and are small enough that their absent shadows are not missed.
