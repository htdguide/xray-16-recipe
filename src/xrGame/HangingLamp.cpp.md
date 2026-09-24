# src/xrGame/HangingLamp.cpp

> A light fixture that can be shot out: up to three render lights riding on named bones of a physically-jointed model, with a health value and a bone whose destruction kills the lamp.

**Needs** — [`HangingLamp.h`](HangingLamp.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`HangingLamp.h`](HangingLamp.h.md); callers name that, not this file.
**Tier floor** — T2: transform composition and light property updates; the light and body handles are interfaces

## Purpose

The game's light fixtures are entities rather than level decoration, for one reason: the
player can shoot them out. That requirement drags in everything else here — a health
value, a hit handler, a bone that disappears when the bulb breaks, a physics body so the
fixture swings, and a state flag that must survive a save.

The lamp carries **three** distinct light contributions and they are not interchangeable:

- the **main light**, which casts shadows, may be a spot or a point, and may be
  volumetric;
- an optional **ambient point light**, shadowless and much larger in radius, which exists
  to fill the room without paying for a second shadowed light. Its colour is the main
  colour scaled by an ambient power factor;
- an optional **glow**, a screen-facing sprite that represents the visible bulb itself.

Each rides on a bone of the model, so a swinging lamp's light swings with it.

## State

```text
RECORD HangingLamp
  light_bone     : int          # the bone the main light rides on; also the "bulb" hit target
  ambient_bone   : int          # the bone the ambient light rides on; often the same one
  main_light     : LightHandle
  ambient_light  : optional<LightHandle>
  glow           : optional<GlowHandle>
  colour_animator: optional<LightAnimation>   # a shared, named colour-over-time curve
  brightness     : real         # the configured multiplier on the authored colour
  ambient_power  : real         # the ambient light's colour is the main colour times this
  health         : real         # starts at the spawn record's value; <= 0 means broken
  on             : bool         # the saved on/off state
```

Invariants:

- The lamp is *alive* exactly when its health is positive; a dead lamp can never be
  turned on again, and turning it on is silently refused.
- The main light's activity and the bulb bone's visibility are kept in step: turning off
  hides the bone, turning on shows it. The model's broken-bulb appearance is therefore
  free — it is the same bone toggle.
- Per-frame processing is enabled exactly while the lamp is lit. An unlit lamp costs
  nothing per frame.
- At least one bone must remain visible. Hiding the bulb bone must not hide the whole
  model; the code asserts this rather than checking it, so it is a constraint on the
  *authored model*, not a runtime case.

## `net_Spawn`

**Contract** — build the live lamp from its server record. Resolves the two named bones,
builds the skeletal collision form, creates the main light with every property the record
carries, creates the glow and the ambient light if the record asks for them, takes the
health, binds the named colour animation, spawns the physics skeleton, starts the idle
animation, evaluates the pose once, and finally turns the lamp on or off according to its
saved state.

**Invariants** — the collision form is *replaced*, not added to: the generic one the base
built is discarded and a skeleton-shaped one built in its place, because a lamp is hit
per-bone. Visibility and enabledness are then derived from what actually exists — visible
only with a model, enabled only with a collision form — rather than assumed.

The bone pose must be computed *before* the first frame, because the first frame's light
placement reads bone transforms.

```text
FUNCTION net_spawn(record)
  base_spawn(record)
  replace the collision form with a skeleton-shaped one
  light_bone   = bone id of record.main_bone         # required to exist
  ambient_bone = bone id of record.ambient_bone
  colour = record.colour, opaque, scaled by record.brightness

  main_light = create light
    shadow, volumetric and spot-versus-point from record flags
    range, cone angle, projected texture, virtual size from the record
    volumetric quality, intensity and distance from the record
  IF record names a glow texture THEN glow = create glow with that texture, colour, radius
  IF record asks for point ambient THEN
    ambient_light = create a shadowless point light
      range = record.ambient_radius, colour = colour * record.ambient_power
  health = record.health
  colour_animator = the named animation from the shared library, if any

  spawn the physics skeleton from the record
  play the idle animation and evaluate the pose once
  IF alive AND saved state is on THEN turn_on() ELSE turn_off()
```

**Notes** — turning off during spawn is preceded by *enabling* per-frame processing and
followed by the disable inside the turn-off, because the processing counter must be
balanced. That is an artifact of a reference-counted enable; a rebuild with a plain flag
does not need the dance.

A record that asks for physics but supplies no model is reported and otherwise ignored.

## `UpdateCL`

**Contract** — the per-frame update, run only while the lamp is lit. Interpolates the
object transform from the physics body, re-evaluates the skeleton, and moves each light
onto its bone. Then, if a colour animation is bound, samples it by global time and
re-tints every light.

**Invariants** — the light's orientation is taken from the bone's *forward and right*
axes, which is what aims a spot light down the fixture. The ambient light reuses the main
light's computed transform when the two bones coincide, which is the common case, and
only recomputes when they differ.

```text
FUNCTION update_per_frame()
  IF a physics body exists THEN transform = body's interpolated transform
  IF not alive OR main light is off THEN RETURN
  evaluate the skeleton's bone transforms
  bone_xf = object transform composed with the main bone's transform
  main_light.orientation = (bone_xf forward, bone_xf right)
  main_light.position = bone_xf.origin
  glow.position = bone_xf.origin
  IF an ambient light exists THEN
    IF its bone differs THEN recompose bone_xf from that bone
    ambient_light.orientation and position from bone_xf
  IF a colour animator exists THEN
    c = animator sampled at global time
    main_light.colour = c scaled by brightness
    glow.colour = the same
    ambient_light.colour = that scaled again by ambient power
```

**Notes** — the shared light-animation library returns its colour with the red and blue
channels transposed relative to the engine's own colour order, and the sample is
un-transposed here at every use. That is a frozen quirk of the animation data's format,
not a decision; a rebuild parsing the same data must apply the same swap.

The brightness divisor of 255 at this site, absent at spawn, is because the animation
yields byte channels while the record yields normalized ones.

## `TurnOn` / `TurnOff`

**Contract** — activate or deactivate every light the lamp owns, show or hide the bulb
bone, enable or disable per-frame processing, and record the state. Turning on a dead lamp
does nothing; turning off an already-off lamp does nothing.

**Invariants** — each light is positioned at the object's origin *before* being activated,
so a light never appears for one frame at wherever it was last left. The per-frame update
will move it onto its bone on the next frame.

**Notes** — the bone is made visible, the pose recomputed, and then made visible *again*,
with the source calling the repetition a hack. The underlying problem is that recomputing
the pose can itself reset per-bone visibility; a rebuild should make pose evaluation not
touch visibility, and then one assignment suffices.

## `Hit`

**Contract** — take damage. Fires the script hit callback, passes the impulse to the
physics body so the fixture swings, and then applies damage: a hit **on the bulb bone**
is instantly fatal regardless of magnitude, any other bone subtracts a hundred times the
damage from the health. A lamp that was alive and now is not turns itself off.

**Invariants** — the fatal-bulb rule is what makes shooting out a light feel right: the
glass breaks when hit, and the housing merely takes damage. The factor of a hundred
against a health that starts near a hundred means an ordinary bullet destroys a fixture
in a few hits; nothing in the source derives it.

## `CreateBody`

**Contract** — build the physics body from the model's skeleton, fixing the bones the
record names in place so the fixture hangs rather than falls. With no named bones the
model's root is fixed instead. Sets a very low air resistance so a hit makes the lamp
swing for a long time, and loads the auto-sleep parameters from the model's own embedded
user data.

**Invariants** — fixing happens *after* the body is built and activated, because the
elements do not exist until then. A lamp with no fixed bone at all would simply fall,
which is why the root is the fallback.

**Notes** — air resistance is set twice, once with defaults and once with the small
explicit pair. Only the second takes effect. The swing damping is the whole feel of a
shot-out lamp, so the numbers matter even though their derivation is not recorded.

The bone map is a file-scope shared structure reused across every lamp, cleared at the
start of each build. That is an allocation avoidance in the original; it also makes lamp
construction non-reentrant. A rebuild should make it local.

## `net_Destroy` / `RespawnInit`

**Contract** — destroy the three light handles, reset every field to its constructed
value, make all the model's bones visible again and recompute the pose, then let the
physics skeleton and the base reset. The lamp is being returned to a reusable state, not
merely freed.

**Invariants** — bones are made visible again on teardown so that a re-spawned lamp does
not inherit a hidden bulb. This is the reset half of the invariant that lamp state lives
partly in bone visibility.

## Save and load — `save`, `load`, `net_Save`, `net_SaveRelevant`

**Contract** — the lamp's only own saved state is one byte: whether it is on. Everything
else is either in the spawn record or in the physics skeleton's own saved state, which is
appended by the base. The lamp is always save-relevant.

**Notes** — the health is *not* saved. A lamp shot out before a save will be lit again
after a load unless it was also switched off, which it is — the hit path turns it off and
the off state is saved. The effect is right, but the mechanism is indirect and a rebuild
should save the health instead.

## `CopySpawnInit`

**Contract** — after the physics skeleton has copied its state from the record, turn the
lamp off if the bulb bone came back hidden. This is how a lamp that was destroyed before
the level was saved stays destroyed: the bone visibility mask is part of the skeleton's
saved state and the lamp's own flag is re-derived from it.

## `Center` / `Radius`

**Contract** — the lamp's bounding sphere, taken from its model's own and transformed
into the world, or degenerate at the origin when there is no model. Used by the spatial
registry and by visibility.

## `UsedAI_Locations`

**Contract** — false. A lamp never occupies a navigation-graph position; creatures walk
under it.

## `SpawnInitPhysics`, `shedule_Update`, `net_Export`, `net_Import`, `Load`

**Contract** — build the body when the record asks for physics and evaluate the pose;
forward the scheduled tick to the physics skeleton and the base; and nothing at all for
the network import and export, since a lamp's only replicated state is the events that
change it.

## `script_register`

**Contract** — exports the lamp to the script layer as `hanging_lamp`, derived from the
game object, with a default constructor and two methods, `turn_on` and `turn_off`. That
is the whole script surface: scripts switch lamps, they do not configure them.
