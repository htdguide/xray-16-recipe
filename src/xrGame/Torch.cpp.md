# src/xrGame/Torch.cpp

> The flashlight: three render objects that must follow the player's gaze rather than his model, plus — for historical reasons — the switch that drives night vision.

**Needs** — [`Torch.h`](Torch.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`ActorHelmet.h`](ActorHelmet.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`Level.h`](Level.h.md) · [`HudSound.h`](HudSound.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`Torch.h`](Torch.h.md); callers name that, not this file.
**Tier floor** — T2: light parameter management, an angular filter and a per-frame pose decision

## Purpose

A flashlight sounds trivial and is not, because of one conflict: the light belongs to an
*object* attached to a *model*, and the player expects it to point where he is *looking*.
A bone on an animated arm lags the camera, swings with the walk cycle, and is offset to one
side of the eye. Resolving that conflict is most of this file.

Three render objects are kept because one cannot do the job: a shadow-casting spot for the
beam, a cheap shadowless point light for the near-field fill that a pure spot leaves black,
and a glow sprite for the lens itself seen head-on.

The file also carries night vision, which has nothing to do with a flashlight. The torch
item is simply where the switch ended up, and the *parameters* come from whatever headgear
or suit the player is wearing. A rebuild should put night vision on the equipment that
provides it and leave the torch alone; the coupling here is historical and the recipe
records it rather than endorsing it.

## State

```text
RECORD Torch EXTENDS InventoryItem
  beam        : SpotLight        # shadow-casting
  fill        : PointLight       # no shadows; kills the black ring under a bare spot
  glow        : GlowSprite       # the lens seen from in front
  switched_on : bool
  brightness  : real             # the light's intensity, separated from its animated hue
  colour_animation : optional<ColourAnimation>   # from the library, by name
  guide_bone  : bone identifier  # the bone the light hangs off
  lens_bone   : bone name        # a piece of geometry visible only while lit
  beam_yaw_correction : real     # see net_Spawn
  smoothed_aim : (yaw, pitch)    # the filtered gaze the beam actually follows
  aim_origin  : vector

  night_vision_available : bool  # from configuration
  night_vision_on        : bool
  night_vision           : optional<NightVisionEffect>
  sounds                 : SoundSet
```

**Invariants**

- All three render objects exist for the torch's whole life and are activated and
  deactivated, never created and destroyed. Creating a shadow-casting light per switch
  would stall the renderer.
- The lit lens geometry's visibility tracks the switch exactly. A torch that is off but
  whose lens still glows is the most obvious possible artifact.
- The beam is never active while the torch is lying on the ground.

## Construction

**Contract** — creates the three render objects, fixes the beam as a shadow-casting spot
and the fill as a shadowless point.

**Notes** — the lateral and forward components of the beam's offset from the model are
zeroed on the oldest renderer, which has no dynamic lighting to place the offset light
into: the offset exists to make a *dynamically lit* scene look right, and on the forward
path it would only move the glow off the lens. A rebuild targeting one lighting model
keeps one offset.

## `Load`

**Contract** — reads from the item's configuration section: the name of the lens bone,
whether this torch model carries night vision at all, and the optional switch-on and
switch-off sounds.

## `net_Spawn`

**Contract** — brings the torch up and configures its three render objects. The light's
*description does not come from the item's configuration section*: it is read from a
user-data block embedded in the **model**, so that the light travels with the art rather
than with the item's game statistics. That is the file's most reusable decision.

```text
FUNCTION net_Spawn(record) -> bool
  visual <- record.visual_name
  REQUIRE no collision form yet AND the visual is animated
  collision_form <- a skeleton collision form      # hits resolve to a bone

  IF NOT base.net_Spawn(record) THEN RETURN false

  deferred <- the active renderer is a deferred one
  data <- the model's embedded user-data block named "torch_definition"
  FAIL WITH "torch model carries no light definition" IF data IS none

  colour_animation <- library entry named data."color_animator"
  guide_bone       <- bone named data."guide_bone"        # must exist

  # two parameter sets: the deferred path wants different values
  colour <- data[deferred ? "color_r2" : "color"]
  range  <- data[deferred ? "range_r2" : "range"]
  brightness <- intensity of colour
  beam.colour <- colour ; beam.range <- range
  IF deferred AND data."volumetric_enabled" THEN
    beam.volumetric <- on with quality, intensity and distance from data, each clamped

  fill.colour <- data[deferred ? "omni_color_r2" : "omni_color"]
  fill.range  <- data[deferred ? "omni_range_r2" : "omni_range"]

  beam.cone     <- radians(data."spot_angle")
  beam.texture  <- data."spot_texture"          # the beam's cross-section pattern
  glow.texture  <- data."glow_texture"
  glow.colour   <- colour
  glow.radius   <- data."glow_radius"

  Switch(record.was_on)
  REQUIRE record.was_off OR record has a parent    # a lit torch lying loose is impossible
  IF record.parent IS the player THEN restore night vision from the record, silently

  beam_yaw_correction <- quarter_turn - arctangent( (range / 2) / |lateral_offset| )
  RETURN true
```

**Invariants**

- The **yaw correction** is the answer to the conflict this file exists to solve. The beam
  is emitted from a point offset to one side of the eye; if it pointed exactly where the
  player looks, the lit patch would sit off-centre by that offset. The correction is the
  angle that makes the beam's axis converge with the view axis at roughly half the light's
  range — so the bright spot lands where the player is looking, at the distance he is
  likely to be looking. It depends on both the offset and the range, which is why it is
  computed here and not written as a constant.
- Night vision is restored from the save only for the *player's own* torch, identified by
  its parent being the player entity. A creature's torch has no night vision to restore,
  and restoring it silently — without sounds — avoids a click on every load.
- Two complete parameter sets keyed by renderer generation are a **data** contract: the
  shipped models carry both, and a rebuild with a different lighting model needs its own
  third set rather than a translation of either.

## `UpdateCL`

**Contract** — places and orients the three render objects each frame. Returns immediately
when the torch is off. Has three distinct cases: held by the player, held by anyone else,
and lying on the ground.

```text
FUNCTION UpdateCL()
  base.UpdateCL()
  IF NOT switched_on THEN RETURN
  bone <- the guide bone's current transform

  IF held THEN
    IF the holder is the player THEN
      invalidate the holder's pose
      IF holder is within 100 m of the camera OR this is multiplayer THEN
        evaluate the holder's skeleton and take the exact guide-bone transform
      ELSE
        approximate: the holder's own transform, moved to the holder's bounding centre,
        raised by two thirds of the holder's radius          # see note

      # follow the gaze, not the bone
      smoothed_aim <- angular_filter(smoothed_aim, -camera.yaw, -camera.pitch,
                                     min_speed, max_speed, max_lag)
      beam_dir <- direction from (smoothed_aim.yaw + beam_yaw_correction, smoothed_aim.pitch)
      beam.position <- bone_origin displaced along the holder's basis by the beam offset
      fill.position <- bone_origin displaced by the fill offset
      glow.position <- bone_origin
      beam.orientation <- fill.orientation <- glow.direction <- beam_dir
    ELSE
      # a creature's torch points where its model points; no filtering
      beam.place_and_orient_from(bone)
      fill.place_and_orient_from(bone)
      glow.place_and_orient_from(bone)

  ELSE IF visible AND has a physics body THEN
    switched_on <- false ; deactivate all three          # a dropped torch goes out

  IF colour_animation EXISTS THEN
    hue <- colour_animation sampled at the global clock
    tint <- hue scaled by brightness
    beam.colour <- fill.colour <- glow.colour <- tint
```

**Invariants**

- The **angular filter** is what makes the beam feel like a hand-held object rather than a
  head-mounted one. It moves the beam toward the camera's angles at a speed that grows with
  the error, between a floor and a ceiling, and it *clamps the lag*: the beam can never
  trail the view by more than a fixed angle. Without the clamp, a fast spin leaves the beam
  pointing at a wall behind you and then whips across the scene.
- The lag clamp is a fixed angle of thirty degrees. Below that the beam swings naturally;
  at it, the beam is dragged. This one number is the entire feel of the flashlight and a
  rebuild should tune it by eye, not copy it blindly.
- The colour animation is sampled in a channel order that is *not* the engine's, and the
  channels are swapped on the way out. That is a property of the animation library's
  storage format, which is frozen by the shipped animation data.
- Brightness and hue are kept separate: the animation supplies a hue that flickers, and the
  model's own intensity scales it. Baking them together makes a flickering torch also
  change brightness range when its model changes.

**Notes** — the distance approximation is the only optimization in the file and it is a
real one: evaluating an animated skeleton to find one bone costs far more than the light is
worth when the holder is a hundred metres away and the beam is a distant smudge. Inside
that radius, or in multiplayer where another player's torch is a targeting cue, the exact
bone is used. The approximate placement — the holder's bounding centre raised by two
thirds of his radius — is "roughly chest height, roughly where a hand would be", and is
deliberately crude.

Several conditionals in the routine are permanently true, left from a version in which the
fill light and the offsets were optional. They should be flattened.

## `Switch`

**Contract** — turns the torch on or off. The no-argument form toggles and is refused on a
client: the switch is authoritative state and reaches clients as a replicated flag.

```text
FUNCTION Switch(on)
  IF held by the player THEN
    play the switch-on or switch-off sound, in first person if the player is,
      but only on an actual change of state
  switched_on <- on
  IF dynamic lights are permitted THEN beam.active <- fill.active <- on
  glow.active <- on                          # the glow is always affordable
  IF a lens bone is named THEN
    make it visible exactly when lit, and re-evaluate the pose immediately
```

**Invariants** — the glow is activated regardless of whether dynamic lights are permitted.
When they are not — a low graphics setting, or a creature the game has decided is not worth
lighting — the torch still *looks* on from the front, which is what another player or the
artificial intelligence needs to see.

The pose is re-evaluated immediately after the lens bone's visibility changes, rather than
being left to the next frame, because the switch may happen in a frame that has already
drawn.

## `SwitchNightVision`

**Contract** — turns night vision on or off. The no-argument form toggles and is refused on
a client. Does nothing if this torch model does not carry night vision, or if it is not
held by the player.

The *parameters* of the effect come from the player's equipment, searched in a fixed order —
**headgear first, then suit** — and the first one that declares a night-vision
configuration wins. If neither does, the request is silently refused and the flag is forced
back off, because there is nothing to turn on.

```text
FUNCTION SwitchNightVision(on, with_sounds)
  IF NOT night_vision_available THEN RETURN
  night_vision_on <- on
  IF NOT held by the player THEN RETURN
  IF night_vision effect object does not exist THEN create it from this item's section

  permitted <- the current level's name is NOT in this item's "disabled levels" list
  source <- headgear's night-vision configuration, else suit's, else none

  IF source EXISTS AND NOT permitted THEN
    play the "malfunction" cue and RETURN        # see note

  IF on AND the effect is not already running THEN
    IF source EXISTS THEN start the effect from source AND RETURN
    night_vision_on <- false                      # nothing to turn on
  ELSE IF NOT on AND the effect is running THEN
    stop the effect
```

**Invariants** — the level blacklist is per item and is read as a list of level names. A
level on the list makes night vision *fail audibly* rather than silently: the player must
understand that the device is refusing, not broken. This is a narrative mechanism — certain
places jam the equipment — implemented as data.

**Notes** — the headgear-before-suit order is a real precedence rule and the shipped data
relies on it: a suit and a helmet may both declare night vision and the helmet's is the one
worn closer to the eyes. A rebuild must keep the order.

## `net_Export` / `net_Import`

**Contract** — the torch's replicated state is **one byte of three flags**: the light is
on, night vision is on, and the item is attached to its owner rather than merely carried.
Import applies each flag only when it differs from the local value, by calling the same
switch routines a local action would, so that sounds and lens visibility follow on a remote
machine exactly as they do locally.

**Invariants** — night vision is only applied on import when the holder is the player.
Replicating it onto a creature would start a post-process effect for a viewer who is not
that creature.

## `OnH_A_Chield` / `OnH_B_Independent` / `afterDetach` / `enable`

**Contract** — the attachment lifecycle. On becoming a child of an owner, the aim origin is
seeded from the current position so the angular filter does not start from a stale angle.
On ceasing to be one — dropped, taken, destroyed — the light and night vision are switched
off and all sounds stopped. The same on detaching from a slot, and on being disabled: a
disabled torch that is still lit is switched off.

**Invariants** — every path out of "held and lit" switches the light off. There are four of
them and they are enumerated here deliberately; missing one leaves a light floating at the
last bone position it saw.

## `can_be_attached`

**Contract** — a torch held by the player may be attached only while it occupies an
inventory slot; a torch held by anything else may always be. The distinction exists because
the player's attachment is driven by his slot layout and a creature's is not.

## `net_Destroy`

**Contract** — switches the light and night vision off before the base teardown, so neither
render object nor post-process effect outlives the item.

## `can_use_dynamic_lights`

**Contract** — asks the holder whether dynamic lights are worth spending on this torch. An
unheld torch, or one held by something that is not an inventory owner, always may. The
answer gates the beam and the fill but never the glow.

## `create_physic_shell` / `activate_physic_shell` / `setup_physic_shell`

**Contract** — the torch's physics body is built by the shell-holder base directly, skipping
the inventory item's own handling. An inventory item normally defers these to its owner's
attachment logic; a dropped torch needs a real body.

## The night-vision effect

A small object owning four sounds — switch on, switch off, a looping hum, and a malfunction
cue — and a handle on the player's post-process effect.

**`Start`** — installs a post-process effector of the night-vision kind, configured from a
named section, onto the player's camera, then plays the switch-on cue and starts the hum.

**`Stop`** — stops the effector with an effectively instantaneous fade, plays the switch-off
cue, and stops both the switch-on sound and the hum.

**`IsActive`** — whether the player's camera currently carries a night-vision effector. The
camera, not this object, is the source of truth: the effect can be removed by the camera
for reasons this object does not know about.

**`OnDisabled`** — plays the malfunction cue without touching the effect, for the blacklisted
levels.

**`PlaySounds`** — plays one of the four cues at the player's position, in first-person mix
when the player is in first person; the hum is the only looping one.

**Notes** — the stop fade is requested as an enormous rate, which the effector interprets as
"immediately". A rebuild should have an explicit immediate-stop rather than an
out-of-range rate. **Could not recover**: whether an actual fade was ever intended.
