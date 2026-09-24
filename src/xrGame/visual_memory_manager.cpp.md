# src/xrGame/visual_memory_manager.cpp

> Decides what a creature can see, how long it takes to notice, and what it remembers having seen.

**Needs** — [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`visual_memory_params.h`](visual_memory_params.h.md) · [`memory_space.h`](memory_space.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai/monsters/basemonster/base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`visual_memory_manager.h`](visual_memory_manager.h.md)
**Tier floor** — T2: per-frame work over small vectors of plain records, plus a save format; no device or layout constraint

## Purpose

Sight in this engine is not a boolean. The engine's `feel` layer answers the cheap
geometric question — is this object inside my frustum, and does a ray reach it through
materials that are not fully opaque. This file answers the expensive one: given that the
object is geometrically reachable, has the creature actually *noticed* it yet, and for how
long will it keep believing it is there after the object leaves view.

The mechanism is an accumulator per candidate object. Each update adds a *visibility
increment* derived from distance, angle off the eye axis, the object's speed and the light
falling on it; when the accumulator crosses a threshold the object becomes a remembered
visible object. Backing away drains the accumulator instead. That is what produces the
series' characteristic behaviour: creeping toward a stalker in the dark buys you seconds,
sprinting across his field of view buys you none.

The file also owns the *remembered* set — the visible-object records with their positions,
their timestamps and their per-squad-member visibility bits — and its save format.

## State

```text
RECORD VisionParameters              # one tuned profile, see visual_memory_params
  min_view_distance        : real    # multiplier on eye range, off-axis
  max_view_distance        : real    # multiplier on eye range, on-axis
  visibility_threshold     : real    # accumulator value at which "noticed" flips true
  always_visible_distance  : real    # inside this, noticing is instant
  time_quant               : real    # seconds the increment is normalized against
  decrease_value           : real    # drained per update when out of range
  velocity_factor          : real    # how much the target's speed betrays it
  transparency_threshold   : real    # opacity below which a ray still counts as seeing
  luminocity_factor        : real    # exponent applied to the light on the target
  still_visible_time       : int     # ms a remembered object stays "visible now"

RECORD NotYetVisibleObject           # a candidate being noticed
  object      : ref game object
  value       : real                 # invariant: 0 <= value <= visibility_threshold
  update_time : int                  # global ms of the last update that touched it
  prev_time   : int                  # timestamp of the position sample used for velocity

RECORD VisibleObject                 # a remembered sighting (shape in memory_space)
  object        : ref game object
  object_params : (level vertex, position)   # where it was when last seen
  self_params   : (level vertex, position)   # where I was when I saw it
  level_time    : int                # global ms of the most recent sighting
  squad_mask    : int (64-bit, bitset)  # which squad members contributed
  visible       : int (64-bit, bitset)  # which squad members see it right now
  enabled       : bool

RECORD VisualMemoryManager
  owner         : one of { creature, stalker, standalone sensor }  # exactly one is set
  visible_now   : list<ref game object>   # scratch, refilled every update
  objects       : optional<ref list<VisibleObject>>  # see the sharing invariant below
  not_yet       : list<NotYetVisibleObject>
  delayed       : list<(entity id, VisibleObject)>   # loaded sightings awaiting their spawn
  free, danger  : VisionParameters
  max_objects   : int                # default 128, raised by configuration
  enabled       : bool
  last_update   : int                # global ms
```

**The remembered-object list is borrowed, not owned, for creatures.** A stalker's list
belongs to its squad: the agent layer hands every member the same list so that one member
seeing an enemy is the squad seeing it, with the per-member bitmask recording who. A
standalone sensor has no squad and allocates its own list. The distinction runs through
every method: the list pointer being absent means *the owner is dead* and every query must
answer "no" rather than fault.

**The squad mask is why the visibility bits are 64-bit.** A squad is capped at the bit
width of that mask. A creature with no squad uses an all-ones mask, which makes the same
code path serve both.

**`max_objects` caps memory, and the eviction policy is load-bearing**: when full, the
*least recently seen* record is overwritten. A creature can therefore forget an old enemy
because it walked past a crowd, and that is the intended behaviour.

## `update`

**Contract** — one sensing pass. Called from the owner's scheduled update with the elapsed
time in seconds. Drops every pending delayed-sighting registration first, then does nothing
if sensing is disabled. Allocates only when the remembered list grows. Not thread-safe; it
reads the collision database through the `feel` layer, which is.

**Invariants** — after it returns, every remembered record whose object was destroyed or
was picked up by a parent is gone; every remembered record not refreshed within
`still_visible_time` has its visible bit cleared for this owner; the last-update timestamp
equals the current global time.

```text
FUNCTION update(time_delta)
  clear_delayed_objects()
  IF NOT enabled THEN RETURN
  last_update := now_ms()
  m := squad_mask()

  visible_now := feel_layer.query()          # geometric candidates, this owner's eye

  # a sighting goes stale on its own clock, not when the object leaves the frustum
  FOR EACH r IN objects
    IF r.level_time + params.still_visible_time < now_ms() THEN r.clear_visible(m)

  FOR EACH o IN visible_now
    add_visible_object(o, time_delta)

  # a candidate not touched this pass has lost its accumulation entirely
  FOR EACH c IN not_yet
    IF c.update_time < now_ms() THEN c.value := 0

  drop FROM objects AND not_yet every entry whose object is destroyed or has a parent

  # feed the player's "you are being noticed" indicator
  IF owner is a creature AND the player exists AND owner treats the player as an enemy
    v := candidate value for the player, or 0 if there is none
    set_player_visibility(owner id, clamp(v / visibility_threshold, 0, 1))
  ELSE
    set_player_visibility(owner id, 0)
```

**Notes** — The "not touched this pass ⇒ reset to zero" rule is stricter than the drain in
`visible`. The drain applies while the target is still in the frustum but too far; the
reset applies when the target left the frustum entirely. Breaking line of sight therefore
costs the observer everything it had accumulated, which is the difference between
*breaking* cover and *backing away*.

**The player-visibility readout is a side effect of the AI on a user-interface widget.**
A rebuild should keep it as a published signal the interface subscribes to, not as a call
out of the sensing loop; the coupling here is incidental.

## `visible`

**Contract** — the accumulator step for one candidate. Returns whether the candidate is now
noticed. Mutates or creates the candidate's accumulator record as a side effect, so it is
not a query. Answers false immediately for an absent object, a destroyed object, a monster
that has declared itself unseeable, and — in non-shipping builds — for the player when the
debug flag that blinds the AI is set.

```text
FUNCTION visible(target, time_delta) -> bool
  IF should_ignore(target) OR target.destroyed THEN RETURN false

  object_distance, reach := object_visible_distance(target)
  c := candidate record for target, or none

  IF reach < object_distance                 # too far for this angle
    IF c IS none THEN RETURN false
    c.value := c.value - params.decrease_value
    IF c.value < 0
      c.value := 0                           # note: update_time is NOT refreshed here,
    ELSE                                     # so a fully drained candidate is also
      c.update_time := now_ms()              # eligible for the reset in update()
    RETURN c.value >= params.visibility_threshold

  increment := get_visible_value(target, reach, object_distance, time_delta,
                                 velocity(target), luminocity(target))
  IF c IS none
    c := new candidate for target with value 0, prev_time 0
    append to not_yet
  c.value := clamp(c.value + increment, 0, params.visibility_threshold + epsilon)
  c.update_time := now_ms()
  c.prev_time   := timestamp of the target's second-newest position sample
  RETURN c.value >= params.visibility_threshold
```

**Notes** — The accumulator is clamped *at* the threshold, not above it. There is no
credit for staring: an object that has been plainly visible for a minute is forgotten as
fast as one noticed a moment ago. That is a deliberate flattening of the model and a
rebuild that lets the value run away will make creatures impossible to shake.

When the build does not enable stalker-grade vision for monsters, a monster's sensing
short-circuits to "everything the frustum reached is seen". The shipped configuration
enables it, so the accumulator is the live path for every creature.

## `object_visible_distance`

**Contract** — how far this observer can see *in the direction of this target*, together
with how far away the target actually is. Pure. Reads the eye bone's world transform from
the skeleton for a creature, or the supplied camera for a standalone sensor.

```text
FUNCTION object_visible_distance(target) -> (reach, object_distance)
  IF owner is a creature
    eye_position  := owner transform applied to the eye bone's origin
    eye_direction := from the head's current yaw and pitch      # the head, not the body
    range, fov    := owner's configured eye range and field of view
  ELSE
    (eye_position, eye_direction, fov, range) := sensor.camera()

  to_target       := target.center - eye_position
  object_distance := |to_target|
  alpha := clamp(angle between eye_direction and normalized to_target, 0, fov/2)

  # linear falloff from the on-axis reach to the off-axis reach
  reach := (1 - alpha/(fov/2)) * (range*max_view_distance - range*min_view_distance)
           + range*min_view_distance
  RETURN (reach, object_distance)
```

**Notes** — Reach is measured from the *head bone*, not the object's origin, and along the
head's aim rather than the body's facing. A creature that has turned its head to look at
something sees along that look. This is the reason the animation pose has to be current
before sensing runs, which constrains the order of the per-frame update: pose, then sense,
then decide.

Both view distances are *multipliers* on the configured eye range, not absolute distances.
A configuration that sets them above one extends sight beyond the declared range.

## `get_visible_value`

**Contract** — the visibility increment for one update of one candidate. Pure except for
the script hook. Returns the threshold outright — instant notice — when the reach at this
angle is within the always-visible distance.

```text
FUNCTION get_visible_value(target, reach, object_distance, time_delta,
                           velocity, luminocity) -> real
  IF reach <= always_visible_distance THEN RETURN visibility_threshold

  IF the script layer defines an override THEN RETURN override(...)

  RETURN (time_delta / time_quant)
       * luminocity
       * (1 + velocity_factor * velocity)
       * (reach - object_distance) / (reach - always_visible_distance)
```

**Invariants** — the last factor is in `(0, 1]` whenever the caller reached here, because
the caller only calls with `object_distance <= reach` and the early return has removed the
case where the denominator vanishes.

**Notes** — Read the formula as four independent multipliers: *how much time passed*
(scaled so that one `time_quant` of exposure is the unit), *how lit the target is*, *how
much its motion betrays it*, and *how deep inside the observer's reach it is*. Each is
separately tunable from configuration, and the game's difficulty tuning moves exactly these
numbers.

**The script override replaces the whole formula, not a term of it.** A modder-supplied
function receives every input and returns the increment. This is a deliberate extension
point added by the open-source project, not part of the original; a rebuild that omits it
still passes the conformance criteria, but mods that use it will not run.

## `object_luminocity`

**Contract** — how much the light on a target contributes to being spotted. Returns one for
anything that is not a living entity, so inert objects are lit-neutral. Otherwise raises
the renderer's sampled luminance for that object to the configured exponent, with the
luminance floored at one thousandth before the logarithm.

```text
FUNCTION object_luminocity(target) -> real
  IF target is not a living entity THEN RETURN 1
  l := renderer's sampled luminance at the target
  RETURN exp(log(max(l, 0.001)) * luminocity_factor)     # i.e. max(l,0.001) ^ factor
```

**Notes** — The floor exists so total darkness yields a small positive factor rather than
zero: a creature standing in pitch black is eventually noticed, just very slowly. Setting
the exponent to zero disables light sensitivity entirely, which is how monsters that see in
the dark are configured.

## `get_object_velocity`

**Contract** — the target's recent speed, from its own position history rather than from
its physics state. Returns zero for anything that is not a living entity, for a target with
fewer than two recorded samples, and for a target whose history has not advanced since this
candidate last looked.

```text
FUNCTION get_object_velocity(target, candidate) -> real
  IF target is not a living entity THEN RETURN 0
  IF target.samples < 2 OR candidate.prev_time == time of sample[-2] THEN RETURN 0
  a := sample[-2] ; b := sample[-1]
  RETURN distance(a.position, b.position) / (b.time - a.time)     # seconds
```

**Notes** — Using the sampled history rather than the instantaneous velocity is what makes
this work for an object whose motion is driven by animation or by the network rather than
by the physics solver; every game object keeps this history for exactly such consumers.
The `prev_time` guard stops a stale pair of samples from being counted twice at a high
update rate, which would otherwise make a fast mover *easier* to lose track of the faster
the observer thinks.

## `add_visible_object`

**Contract** — two entry points that both end at the remembered-object list. The public one
takes a candidate and the elapsed time, runs the accumulator, and records the sighting if
it crossed; the internal one takes an already-formed record (from a save, or from a squad
mate) and inserts it directly. Both refuse ignored objects. Both may evict.

```text
FUNCTION add_visible_object(target, time_delta, fictitious = false)
  IF NOT fictitious AND should_ignore(target) THEN RETURN
  IF target is not a game object THEN RETURN
  IF NOT fictitious AND NOT visible(target, time_delta) THEN RETURN

  r := remembered record for target, or none
  IF r IS none
    r := fill a fresh record from (target, self, squad mask)
    IF objects.size >= max_objects
      overwrite the record with the smallest level_time      # evict least recently seen
    ELSE
      append r
  ELSE IF NOT fictitious
    refresh r from (target, self) and OR this owner's bit into both masks
  ELSE
    OR this owner's bit into both masks and re-enable, but leave the positions alone
```

**Notes** — *Fictitious* means "record this sighting without anyone actually having seen
it": the squad layer uses it to share a mate's sighting, and scripts use it to plant
knowledge. The distinguishing rule is that a fictitious add never rewrites the recorded
positions, so a shared sighting keeps the position of whoever really saw it.

## `visible_right_now` and `visible_now`

**Contract** — the two questions the brain asks. `visible_right_now` is true only if the
sighting was refreshed by the most recent sensing pass. `visible_now` is true if the
sighting's visible bit is still set, which — because the bit survives for
`still_visible_time` after the last refresh — means "seen now, or recently enough that I
still believe it is there". Both answer false when the owner is dead (no remembered list)
and for ignored objects. Neither mutates.

**Notes** — With `still_visible_time` configured to zero the two collapse to the same
predicate. The gap between them is what lets a creature keep shooting at a position for a
moment after the target ducks behind cover, and the configured value is a per-creature
difficulty knob.

## `visible(level vertex, yaw, field of view)`

**Contract** — a cheap variant that asks whether a *navigation position* is within a given
cone from the owner and unobstructed along the level graph. Used by the cover and position
selection code, which reasons about vertices rather than objects. Does not touch the
collision database: obstruction is answered by the level graph's own directional walk, so
it agrees with what the pathfinder believes rather than with what a ray would find.

## `feel_vision_mtl_transp`

**Contract** — how much sight a given surface lets through, from the material table. For a
dynamic object the element selects a bone and the bone names a material; for the static
world the element indexes a triangle of the collision database and the triangle names one.
Returns the material's visual transparency factor. This is the callback the `feel` layer
uses when its ray crosses geometry, and it is why a creature can see through a chain-link
fence and not through a wall.

## `mask`

**Contract** — this owner's bit in the squad's visibility bitmask. A squad member asks its
agent layer for its member bit; anything without a squad returns an all-ones mask, which
makes every mask test trivially true and lets the squad-aware code run unchanged for
solitary creatures.

## `current_state`

**Contract** — selects between the two tuned parameter profiles. A stalker uses the *danger*
profile while its movement layer reports a danger mental state; a monster uses it while it
has an enemy; a standalone sensor always uses *free*. Nothing else switches profiles, and
the switch is instantaneous — there is no blend between the two sets of numbers.

## `reload`

**Contract** — reads the tuned profiles for one configuration section. A stalker requires
both a free and a danger profile section to be named. Another creature may name them and
falls back to its own section for both. A standalone sensor loads only the free profile.
The dynamic-object cap is read if present and only ever *raises* the default of 128.

**Notes** — That the cap can only rise is the interesting part: a configuration cannot make
a creature more forgetful than the built-in default, only less. Whether that was intended
or is the accident of a `max` where a plain assignment was meant is not recoverable from
the source.

## `reinit`

**Contract** — reset to a fresh-game state. Clears the candidate set, the geometric scratch
list and the `feel` layer's own accumulated visibility, and sets the last-update timestamp
to the maximum value so that the first comparison against it fails rather than succeeds.
For a creature it also *drops* the remembered-object list pointer, because that list
belongs to the squad and will be handed back on the next registration; for a standalone
sensor it clears the list it owns.

## `save` and `load`

**Contract** — serialize the remembered sightings into and out of a save. A standalone
sensor saves nothing. A dead owner saves nothing, and a dead owner's load is skipped
entirely — the count byte is then never consumed, so a rebuild must reproduce the *same*
decision or the surrounding stream desynchronizes.

```text
FUNCTION save(packet)
  IF owner is a standalone sensor OR owner is dead THEN RETURN
  worth := [ r IN objects WHERE is_valuable(r) ]
  packet.write_u8(count of worth)                # invariant: caps saved sightings at 255
  IF empty THEN RETURN
  FOR EACH r IN worth
    write entity id (16-bit)
    write r.object_params.level_vertex, r.object_params.position
    write r.self_params.level_vertex,   r.self_params.position
    write (now - r.level_time) and (now - r.last_level_time) as ages, not timestamps
    write r.visible bitset (64-bit)

FUNCTION is_valuable(record) -> bool
  IF record.object is not a living entity THEN RETURN false
  IF record.object is dead THEN RETURN true          # remembering a corpse matters
  RETURN owner treats it as an enemy                 # remembering a neutral does not
```

**Invariants** — timestamps are written as *ages relative to now* and read back as
`now - age`. Level time restarts at zero on load, so an absolute timestamp would be
meaningless; this is the general rule for every saved timestamp in the chapter.

**Notes** — **Load has to cope with a sighting of an entity that does not exist yet.** The
save file has no ordering guarantee between an observer and what it remembers, so a loaded
record whose entity is not yet resolvable is parked in the delayed list and a callback is
registered with the client spawn manager keyed by (awaited entity, this owner). When the
entity appears, the parked record is completed and inserted — unless the owner has died in
the meantime, in which case it is simply dropped. The delayed list must be drained of its
registrations before the manager is destroyed or the spawn manager is left holding a
callback into freed state; that is what the drain at the top of every `update` and in the
destructor is for.

The saved count is one byte and the in-memory cap is 128, so the cap is what actually
bounds it. A rebuild that raises the cap past 255 must widen the count field, which changes
the save format.

## `remove_links` and `remove`

**Contract** — sever every reference to a departing object: one remembered record and one
candidate record, each removed if present. `remove` takes a record by identity instead of
an object, for a caller that already holds the record. This is the standard teardown hook
every object registry in the chapter calls before an object is destroyed, and skipping it
leaves a dangling reference that the next sensing pass will follow.

## `enable`

**Contract** — two forms. With an object, sets the enabled flag on that one remembered
record, which suppresses it from the brain's consideration without forgetting it. Without,
switches the whole manager off: sensing stops, but the remembered set is preserved, so
re-enabling resumes from what was known rather than from nothing.

## `visible_object`, `visible_object_time_last_seen`, `not_yet_visible_object`

**Contract** — lookups into the three sets by object identity. The time-last-seen query
returns the maximum integer value when there is no record, which every caller must treat as
"never" rather than as "long ago"; comparing it as a number gives the opposite of the
intended answer.

## `should_ignore_object`

**Contract** — the blanket exclusion filter, consulted by every entry point. Excludes an
absent object; excludes the player when the debug flag that blinds the AI is set (absent
from shipping builds); and excludes a monster that reports itself as currently unseeable,
which is how burrowing and cloaking creatures work. Everything else is fair game.

**Notes** — This is checked in the query paths as well as the recording paths, so a
creature that cloaks becomes invisible *in memory* as well as in perception — the brain
cannot act on a remembered sighting of a cloaked monster. That is a stronger statement than
"it cannot be seen" and it is deliberate.
