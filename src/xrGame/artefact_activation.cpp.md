# src/xrGame/artefact_activation.cpp

> Runs the timed sequence in which a discarded artefact rises, hangs, and detonates into a new anomaly.

**Needs** — [`artefact_activation.h`](artefact_activation.h.md) · [`Artefact.h`](Artefact.h.md) · [`Level.h`](Level.h.md) · [`Inventory.h`](Inventory.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`restriction_space.h`](../xrServerEntities/restriction_space.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a timed state machine over effects, a physics force and one spawn; no layout concern beyond the spawn packet

## Purpose

An artefact is an anomaly's residue, and this file closes the loop: an artefact the player
drops in the world can be *activated*, at which point it detaches from its owner, floats
upward, hangs, and is replaced by a freshly spawned anomalous zone at its position. It is
the only path by which the world gains an anomaly that was not authored into the level.

The whole sequence is four timed states, and everything about each one — how long it
lasts, what sound it makes, what colour light it casts, how far that light reaches, what
particle effect plays and what animation the artefact runs — is a row of numbers in the
artefact's configuration section. The code owns the ordering and the spawn; the data owns
the presentation.

It is a separate object from the artefact rather than a mode of it because it is transient:
almost no artefact is ever activated, and the light, the sound and the state table would
otherwise sit in every artefact in the world.

## State

```text
ENUM ActivationState
  none, starting, flying, before_spawn, spawn_zone, count

RECORD StateDef                 # one row per state, read from configuration
  time       : real             # seconds this state lasts
  sound      : text             # empty == silent
  light_color: (real, real, real)
  light_range: real
  particle   : text             # empty == none
  animation  : text             # empty == none

RECORD SArtefactActivation
  artefact     : reference to the artefact being activated   # non-owning, outlives this
  states       : list<StateDef>   # exactly `count` entries, indexed by ActivationState
  current      : ActivationState
  state_time   : real             # seconds spent in `current`
  light        : a dynamic light, created for the whole sequence
  sound        : a playing sound handle
  owner_id     : int (entity identifier)   # who set this off
  in_process   : bool
```

**Invariants**

- `states` is indexed by the enumeration, so it must hold exactly one entry per state
  including the two that are never displayed (`none` and `count`). The index-by-enum is why
  the table is built by pushing a default row per enumerator before any of them is loaded.
- The sequence advances strictly `starting → flying → before_spawn → spawn_zone → done`. It
  never branches and never repeats, which is what lets the whole thing be a counter and a
  table rather than a state machine with transitions.
- Every entry point asserts that the physics world is **not** mid-step. This object changes
  an object's forces, destroys it, and spawns another; doing any of that while the solver is
  iterating corrupts the solver's view of the world. The one method that *is* called from
  inside the step is the force application, and it does nothing else.
- The light is created at construction and lives for the whole sequence, being
  re-coloured and re-ranged per state rather than created per state. Creating a shadow-
  casting light is expensive enough that doing it four times in four seconds is visible.

## configuration format

**Contract** — the artefact's section names an *activation sequence* section, and that
section holds four keys — `starting`, `flying`, `idle_before_spawning`, `spawning` — each a
single comma-separated row of exactly eight fields in fixed order: duration, sound name,
three light colour components, light range, particle name, animation name. A row with the
wrong field count is authoring data that is wrong and is rejected.

```text
FUNCTION load_state_def(section, key) -> StateDef
  fields = split(config.string(section, key))
  FAIL WITH "bad activation row" IF count(fields) != 8
  RETURN StateDef{
    time        = number(fields[0]),
    sound       = unquote(fields[1]),
    light_color = (number(fields[2]), number(fields[3]), number(fields[4])),
    light_range = number(fields[5]),
    particle    = unquote(fields[6]),
    animation   = unquote(fields[7])
  }
```

**Notes** — names are unquoted on the way in: the authors wrapped names containing spaces
or commas in double quotes so the field splitter would not cut them, and the quotes are not
part of the name. The unquoting strips a trailing quote before a leading one, so a row with
only one of the two still yields something usable rather than failing — a tolerance worth
keeping, since the shipped data is not consistent about it.

**Notes** — the anomaly to spawn is looked up in a *separate* global table keyed by artefact
section, holding three fields: the zone's configuration section, its radius and its power.
Keeping it out of the per-state rows means the same activation sequence can be shared by
artefacts that produce different anomalies.

## `Start`

**Contract** — begins the sequence. The ordering matters and is the reason this is not
simply "set state to starting":

```text
FUNCTION start()
  REQUIRE physics world is not stepping
  artefact.stop_idle_lights()          # the artefact's own glow, replaced by the sequence light
  current    = starting
  state_time = 0
  artefact.register_for_updates()      # it must now be ticked every frame

  # tell the authoritative side the artefact has left its owner's inventory.
  # This is a server event, not a local state change: in multiplayer the
  # artefact may be in a remote player's inventory and that side owns the record.
  emit ownership_reject(parent = artefact.owner, subject = artefact)

  light.active = true
  apply_state_effects()
  in_process = true
```

## `Stop`

**Contract** — abandons the sequence, returning the artefact to being an ordinary artefact:
unregisters from the physics update, restores its idle lights, and reapplies effects for
the (now meaningless) current state, which silences the sound and blanks the light.

**Notes** — the time field is cleared by assigning it the `none` enumerator rather than
zero. Numerically identical, and a rebuild should just write zero.

## `UpdateActivation`

**Contract** — called once per frame while in process. Advances the state clock; on
expiry moves to the next state, reapplies effects, and — at the moment the spawning state
is entered, and only on the authoritative side — creates the anomaly. Falling off the end
of the table destroys the artefact.

```text
FUNCTION update_activation()
  IF NOT in_process THEN RETURN
  REQUIRE physics world is not stepping

  state_time = state_time + frame_delta
  IF state_time >= states[current].time THEN
    current = next state in order

    IF current == count THEN              # sequence exhausted
      current = none
      artefact.unregister_from_updates()
      artefact.destroy()                  # the anomaly, spawned one state ago, replaces it

    state_time = 0
    apply_state_effects()

    IF current == spawn_zone AND this is the authoritative side THEN
      spawn_anomaly()

  update_effect_positions()
```

**Notes** — the anomaly is spawned when the *spawning* state begins and the artefact is
destroyed when that state ends, so for the length of that state both exist. That overlap is
deliberate: the anomaly's own birth effect plays while the artefact is still visible inside
it, and the player sees one become the other rather than a pop.

**Notes** — after the sequence is exhausted the code still applies effects and updates
positions for a destroyed artefact's state row. Reading the `none` row is harmless because
it is blank, but a rebuild should return immediately after destroying.

## `PhDataUpdate`

**Contract** — called from inside the physics step, with the step length. The only method
that may run while the solver is iterating, and therefore the only one that does nothing
but apply a force.

During the *flying* state the artefact must rise. It is given an upward acceleration of
110% of gravity — but **only while something is within one metre directly below it**.

```text
FUNCTION physics_step(step)
  IF artefact has no physics shell THEN RETURN
  IF current != flying THEN RETURN

  IF ray downward from artefact hits anything within 1 metre THEN
    artefact.apply_gravity_acceleration(up * gravity * 1.1)
```

**Notes** — the ground test is what makes the ascent terminate. An artefact with clearance
beneath it stops being pushed and simply falls back under gravity, so it hovers a metre or
so off whatever it was dropped on instead of climbing forever. A ten-percent excess over
gravity is a slow, floating rise rather than a launch; the value is tuned, not derived.

## `ChangeEffects`

**Contract** — reapplies every presentation channel from the current state's row: stops any
playing sound and starts the row's, if it names one; retunes the light's colour and range;
starts the row's particle effect for exactly the row's duration; plays the row's animation
on the artefact's model if it has one. Called once per state transition, never per frame.

**Notes** — the particle effect's lifetime is set to the state's duration in milliseconds,
so effects end with their state rather than needing to be stopped.

**Notes** — the particle direction is straight up for every state. Not a decision, just the
only direction the effects were authored for.

## `UpdateEffects`

**Contract** — per frame, drags the playing sound and the light to the artefact's current
position. The artefact is physically simulated and moves; both are positional and would
otherwise stay where the state began.

## `SpawnAnomaly`

**Contract** — creates the anomalous zone that replaces the artefact. Runs only on the
authoritative side. Looks up the zone's section, radius and power from the global artefact-
to-zone table, builds a server record at the artefact's centre, gives it a spherical shape
of the configured radius, sets its power and its owner, and broadcasts the spawn.

```text
FUNCTION spawn_anomaly()
  REQUIRE physics world is not stepping
  (zone_section, radius, power) = config.row("artefact_spawn_zones", artefact.section)
  FAIL WITH "bad spawn-zone row" IF that row has other than 3 fields

  record = create server record for zone_section
             at artefact.center,
             on the artefact's navigation vertex,   # none on a dedicated server: it has no level graph
             with no parent

  record.shape       = a sphere of `radius` centred on the record's own origin
  record.max_power   = power                # single player and the first game only; see notes
  record.owner_id    = owner_id             # who is to blame for this anomaly
  record.restrictor_type = none             # it constrains nobody's movement

  broadcast record.spawn_payload reliably
  destroy record                            # the world builds the real entity from the broadcast
```

**Invariants** — the record is written to the wire and then thrown away; the authoritative
world creates the live entity from the broadcast exactly as it would for any other spawn.
That is what makes an artefact-born anomaly indistinguishable from an authored one from
that point on, including through a save.

**Notes** — the zone's power is only assigned in single player and in the first game's
compatibility mode. In the other multiplayer modes the zone keeps its section's default
power, because a player-placed anomaly whose strength is chosen by the placer is a weapon.

**Notes** — on a dedicated server the navigation vertex is passed as "unknown". A dedicated
server does not load the level graph, so it has no vertex to give; the clients resolve one
when they build the object.

**Notes** — the spawn is logged unconditionally, including in release builds. Left on
deliberately: an anomaly appearing where the level author did not put one is the first thing
a bug report needs to explain.

## `IsInProgress`

**Contract** — whether the sequence is running. The artefact uses it to decide whether it
may be picked up, and to route its updates here.
