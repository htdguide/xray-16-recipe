# src/xrGame/player_hud.cpp

> The first-person view: one pair of arms, up to two items in them, the authored measurements that place each item, the motion aliases that animate it, and the inertia that makes the weapon lag the camera.

**Needs** — [`player_hud.h`](player_hud.h.md) · [`HudItem.h`](HudItem.h.md) · [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`physic_item.h`](physic_item.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [`Level.h`](Level.h.md) · [`static_cast_checked.hpp`](static_cast_checked.hpp.md) · [`firedeps.h`](firedeps.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame transform composition and skeletal animation driving

## Purpose

Everything the player sees of their own body is here. There is one rig for the whole
process, it holds one arms model and up to two attached items, and every frame it composes
the camera's transform with a chain of authored offsets and runtime lag to place them.

The file is large because three unrelated problems live in it, and each earns its length:

1. **Placement.** An item must sit in the hands exactly where its art was drawn to sit, in
   both aspect ratios, in three aiming modes, and with its muzzle and ejection port
   findable by the weapon code.
2. **Animation.** One logical action names a set of clips that must play in lockstep on two
   different skeletons at the same speed, with a random variant chosen each time and an
   optional camera shake to match.
3. **Feel.** The weapon lags the camera, swings when you turn, sinks when you look up, and
   settles faster when you are aiming. That is entirely the inertia routine, and it is the
   only part of this file a player would notice changing.

## State

Declared in [`player_hud.h`](player_hud.h.md). The rig owns the arms model, the anchor bone
list, the two attachment slots, and a **cache of constructed items keyed by configuration
section** — an item's rig representation is built once and reused for every instance of that
section the player ever holds.

**Invariants**
- The cache outlives any individual held item. Detaching clears the slot, not the cache
  entry.
- An item in slot 0 is the main hand and one in slot 1 the off hand. The slot comes from the
  item's configuration, not from the order it was picked up.
- A rig with any monolithic item attached does not draw its own arms and does not apply its
  own attachment offset — the item's model *is* the arms.
- Measurements are re-read whenever the aspect ratio changes, because the authored offsets
  differ between them.

## `player_hud_motion_container::load`

**Contract** — builds the motion alias table for one item from its configuration section.
Every key beginning `anm_` or `anim_` is an alias. Its value is one to three
comma-separated fields: the base clip name, an optional distinct clip name for the item's own
model, and an optional speed multiplier. Then up to nine numbered variants of the base name
are probed against the arms model and every one that exists is recorded.

```text
FUNCTION load(model, section)
  FOR EACH (key, value) IN section WHERE key starts with "anm_" or "anim_"
    fields = split_by_comma(value)                 # at most three
    base       = fields[0]
    additional = fields[1] IF non-empty ELSE base  # the item model's own clip
    speed      = fields[2] IF non-empty ELSE 1

    variants = []
    FOR i IN 0 .. 8
      name = base IF i IS 0 ELSE base + i          # "anm_reload", "anm_reload1", …
      IF model has a cycle named name
        variants.append((cycle_id_of(name), name))
    FAIL WITH "motion not found" IF variants is empty
    aliases[key] = (base, additional, variants, speed)
```

**Notes** — the variant probe starts at the *unnumbered* name and then tries one through
eight. That is why an item with three reload animations is authored as `anm_reload`,
`anm_reload1`, `anm_reload2` and not as a list: the set is discovered, not declared. Nine is
the ceiling; a tenth variant in the data is silently ignored.

The second clip name exists because a two-part item has two skeletons — the arms and the
item — and they are not always animated by the same clip name. When the two names differ the
item's model plays the declared additional name; when they are the same it plays *the chosen
variant's* name, so the item's animation varies with the hands'.

## `CalcMotionSpeed`

**Contract** — the authored speed applies in single player. In multiplayer, only the show and
hide animations are affected and they run at double speed; everything else runs at one.

**Notes** — stated in the source as a fairness decision: an item's configured reload speed is
a single-player tuning value and letting a player bring a modified configuration into a match
would let them reload faster than everyone else. Draw and holster are exempted and forced
fast for both sides, so switching weapons feels responsive without being a competitive
advantage.

## `hud_item_measures::load` — two-part items

**Contract** — reads the arms placement, the item placement, the three named points and
their bones, the two non-hip aiming offsets, and the inertia parameters. Produces the item's
attachment transform. Aspect-ratio dependent: six of the keys have a `_16x9` suffixed
variant, and which set is read depends on the current display.

```text
FUNCTION load(section, model) -> attachment transform
  suffix = "_16x9" IF widescreen ELSE ""
  hands_attach = (section."hands_position"+suffix, section."hands_orientation"+suffix)
  item_attach  = (section."item_position",         section."item_orientation")
  attach_transform = compose(item_attach)

  FOR EACH (bone_key, point_key, slot) IN
        (("fire_bone","fire_point",1), ("fire_bone2","fire_point2",2),
         ("shell_bone","shell_point",3))
    present = section has bone_key
    IF present
      bone[slot]   = model.bone_id(section.bone_key)
      offset[slot] = section.point_key
    ELSE
      offset[slot] = zero

  hands_offset[hip]      = zero
  hands_offset[aim]      = (section."aim_hud_offset_pos"+suffix, …"_rot"+suffix)
  hands_offset[launcher] = (section."gl_hud_offset_pos"+suffix,  …"_rot"+suffix)

  assert each point key is present exactly when its bone key is
  load_inertion_params(section)
  record which aspect ratio these were read for
```

**Invariants** — the hip offset is always zero. The authored offsets are *deviations from
the resting pose*, so the resting pose needs none.

**Notes** — the aspect-ratio suffix covers the hand placement and the two aiming offsets and
**not** the item placement or the named points. The item sits on the same bone at the same
place either way; what changes is where the arms are relative to the camera, because a wider
field of view puts them further out of frame. The consequence is that the authored data is
two hand placements and one item placement, not two of everything.

The three point/bone pairings are asserted rather than tolerated because a fire point with no
bone would silently place the muzzle flash at the model's origin, which is the kind of bug
that survives a review.

## `hud_item_measures::load_monolithic`

**Contract** — the same job for an item whose model contains its own arms. Reads a simpler
placement (no separate hands attachment), resolves a single fire bone that serves as the
second fire bone and the shell bone too, and reads the aiming offsets from the weapon's zoom
keys instead of from the hand-offset keys. A non-weapon monolithic item gets no points at
all.

**Notes** — the collapse of three bones into one is the statement that a monolithic model
carries its muzzle, its second muzzle and its ejection port on the same bone; the three
offsets from it still differ. The shell point is read only when the item's own section
declares ejection particles, which avoids a required key on weapons that eject nothing.

A monolithic weapon with a grenade launcher reads *three* sets of zoom offsets: the plain
zoom, a grenade-launcher zoom, and — when the launcher is a detachable attachment rather than
an integral one — a separate "normal" zoom that replaces the plain one. That last case is the
difference between a rifle whose sight picture changes when the launcher is fitted and one
whose does not.

The fire bone here is fatal if missing, where the two-part path treats every point as
optional. A monolithic weapon with no muzzle is meaningless.

## `hud_item_measures::load_inertion_params`

**Contract** — reads the eight inertia numbers, each with a compiled-in default. A section
that declares none gets the defaults, which is the common case: the parameters were added
after the shipped data was authored.

**Notes** — the defaults are the shipped feel. Three of them are zero — the sideways and
vertical pitch offsets, and the pitch lower limit at negative half a turn, which is
effectively unbounded. Only the *depth* offset is non-zero at 0.02, so the stock behaviour is
that looking up and down moves the weapon slightly toward and away from the camera and does
not move it on screen. The other two axes exist for authored data to turn on.

The catch-up speed is 5 at the hip and 8 while aiming; the swing magnitude is -0.05 and
-0.03. Aiming therefore both swings less and settles faster, which is the whole of "the sight
picture steadies when you aim". None of the six values is derived; they are feel values.

The swing magnitudes are **negative**, and the comment in the source says smaller means more
inertia — so the sign is load-bearing and a rebuild that normalizes it to positive must
invert its use.

## `attachable_hud_item` construction

**Contract** — builds one item's rig representation from its configuration section. Decides
monolithic from which visual key is present, creates the model, reads the attachment slot,
loads the motion aliases against **the arms model for a two-part item and the item's own
model for a monolithic one**, and reads the measurements.

**Notes** — which skeleton the aliases are probed against is the whole monolithic/two-part
distinction expressed once. A two-part item's animations live on the arms; a monolithic
item's live on itself.

## `attachable_hud_item::update`

**Contract** — recomputes the item's world transform and its skeleton, at most once per
frame unless forced. Re-reads the measurements if the aspect ratio changed since they were
loaded. Applies the live tuning offsets when the in-game tuner is open.

```text
FUNCTION update(force)
  IF NOT force AND already updated this frame THEN RETURN
  IF widescreen state differs from the loaded measurements THEN reload_measures()
  IF the hud tuner is open THEN rebuild the attachment transform from the tuner's values
  parent.calc_transform(slot, attachment_offset, out item_transform)
  mark updated this frame
  IF the item's model is animated
    advance its animation tracks, then recompute its bones
```

**Invariants** — the once-per-frame guard exists because several callers each want a current
transform — the renderer, the weapon's muzzle query, the interface — and recomputing a
skeleton three times a frame is the difference between affordable and not.

## `attachable_hud_item::setup_firedeps`

**Contract** — produces the weapon's world-space muzzle point, muzzle direction, ejection
point and the transform particles are spawned in. Forces an update first. Each point is
found by transforming its authored offset through its bone's current pose and then through
the item's world transform.

```text
FUNCTION setup_firedeps(out fd)
  update(false)
  IF has fire point
    fd.muzzle    = item_transform ∘ bone[fire].pose ∘ fire_point_offset
    fd.direction = item_transform applied to the model's forward axis
    fd.particle_transform = an orthonormal basis built with direction as its forward axis
  IF has second fire point
    fd.muzzle2   = item_transform ∘ bone[fire2].pose ∘ fire_point2_offset
  IF has shell point
    fd.shell     = item_transform ∘ bone[shell].pose ∘ shell_point_offset
```

**Notes** — the fire direction is the model's own forward axis carried into world space, not
the camera's forward axis. That is why a weapon whose muzzle is visibly off-axis during a
reload animation actually shoots off-axis: the bullet comes out of the model, not out of the
crosshair. The game relies on this — the recoil and sway you see is the recoil and sway you
get.

The particle transform is built as an orthonormal basis from the fire direction alone, so the
muzzle flash has no defined roll. Any roll-asymmetric flash art will spin as the weapon
turns. Nothing in the shipped art is roll-asymmetric.

## `attachable_hud_item::anim_play`

**Contract** — plays one logical animation. Resolves the alias, picks a random variant,
plays it on the arms through the rig, plays the corresponding clip on the item's own model
across every partition, and — in single player, for the entity the player is controlling —
starts a matching camera shake if one is authored. Reports the animation's duration and the
variant index it chose.

```text
FUNCTION anim_play(alias_name, mix_in, out motion_def, out variant_index) -> duration
  # the off hand has its own widescreen variants of every alias
  name = alias_name + ("_16x9" IF slot IS 1 AND widescreen ELSE "")
  alias = motion_aliases[name]                       # must exist
  speed = CalcMotionSpeed(alias.base, alias.speed)
  variant_index = random over alias.variants
  duration = parent.anim_play(slot, alias.variants[variant_index], mix_in, motion_def,
                              speed, the item's model IF monolithic ELSE none)

  IF the item has its own animated model
    item_clip = alias.additional IF it differs from alias.base
                ELSE the chosen variant's name
    resolve item_clip on the item's model, falling back to "idle"
    IF two-part
      pin the item model's root bone to identity     # the arms drive it, not its own root
    play item_clip on every partition of the item's model at the same speed

  IF single player AND the holder is the entity the player controls
    IF a camera-shake file named after the chosen variant exists
      replace any running weapon-action camera effect with a new one from that file
  RETURN duration
```

**Invariants** — the arms and the item play at the **same speed**, taken from the alias.
Two skeletons animating the same action at different rates is the failure this guards
against.

**Notes** — pinning the two-part item's root bone to identity is the mechanism that makes
attachment work. The item's own animation may move its root; if it did, the item would drift
off the hand it is attached to. The arms' anchor bone is the authority on where the item is,
and the item's animation is only allowed to move its own sub-bones. A monolithic item is not
pinned, because its root *is* the authority.

Falling back to `idle` when the named item clip is missing, rather than failing, is what lets
an item declare a hands animation it has no matching item animation for. The fallback is then
asserted — an item with no `idle` at all is an authoring error.

The camera shake is **discovered by file existence**, keyed by the chosen variant's clip
name. There is no configuration key: authoring a file named after the animation is how you
add a shake to it. That is a genuinely unusual binding mechanism and a rebuild must keep it,
because the shipped data uses it and declares it nowhere.

The shake is single-player only and only for the controlled entity, so a spectated player's
reload does not shake your camera.

## `player_hud::load`

**Contract** — brings up an arms model by configuration section. Does nothing if the section
is already loaded. Releases any previous model. A section that does not exist leaves the rig
with no arms — which is legitimate, for a monolithic-only loadout — and still re-notifies the
attached items so they can rebuild against the change.

**Notes** — the first load plays a specific idle cycle by literal name to put the arms in a
defined pose; a reload does not, because the attached items' re-notification restarts their
own animations. The literal cycle name is one of the few in the file and is frozen against
the shipped arms models.

The default section is likewise a literal. It is the fallback when nothing has selected one.

## `player_hud::load_ancors`

**Contract** — collects the bones items attach to, from every configuration key beginning
`ancor_`, in the order the section declares them. The list is indexed by attachment slot, so
**the authored order of those keys decides which bone is the main hand**. A section that
declares them in the wrong order puts the weapon in the off hand.

## `player_hud::update`

**Contract** — the per-frame composition. Takes the camera transform and produces the rig's
world transform and, through it, each attached item's.

```text
FUNCTION update(camera_transform)
  trans = camera_transform
  IF left-handed mode THEN negate the transform's first row      # mirror across the view axis
  update_inertion(trans)          # the lag and swing
  update_additional(trans)        # each item's own per-frame transform adjustment

  IF no arms model OR either attached item is monolithic
    rig_transform = trans         # the item's model is the arms; no attachment offset
  ELSE
    take the hands attachment from whichever slot is filled, main hand first
    attachment = compose(that rotation in degrees, that position)
    rig_transform = trans ∘ attachment
    advance the arms' animation tracks and recompute its bones

  force an update of both attached items
```

**Invariants** — inertia is applied before the attachment offset, so the offset is in the
already-lagged frame. Reversing them would make the weapon's authored placement swing with
the lag rather than the whole rig doing so.

**Notes** — left-handed mode negates the transform's first row rather than multiplying by a
mirror matrix, which the source notes is the faster equivalent. It mirrors the entire rig
including the item, so a left-handed player's weapon ejects to the left and its text reads
backwards. That is a known consequence and the mode is off by default.

Authored rotations are in **degrees** and converted here; positions are in metres. The mixed
units are in the shipped data.

## `player_hud::anim_play`

**Contract** — plays a motion on the arms. When the item is monolithic the arms are not
involved and this only reports the duration. Otherwise it plays the cycle on the arms'
partitions — all of them when only one item is held, and only the relevant hand's partition
plus the root when both hands are full.

```text
FUNCTION anim_play(slot, motion, mix_in, out motion_def, speed, item_model) -> duration
  IF item_model IS none AND an arms model exists
    IF both slots are filled
      target = partition named "right_hand" IF slot IS 0 ELSE "left_hand"
    ELSE
      target = every partition
    FOR EACH partition p
      IF p IS partition 0 OR p IS target OR target IS every
        play(p, motion, mix_in) at speed
  RETURN motion_length(motion, out motion_def, speed, item_model)
```

**Invariants** — partition 0 always plays, whatever the target. It is the root partition and
carries the whole-body motion; excluding it would leave the arms' base pose from a previous
animation.

**Notes** — the hand partition names are literals matching the shipped arms models. The
split only happens when *both* hands are full, so a player holding one item animates their
whole body with it — which is right, since the free hand should follow.

## `player_hud::motion_length`

**Contract** — how long a motion runs at a given speed, in milliseconds. Zero for a looping
motion, which has no length. Rounds to the nearest millisecond.

```text
FUNCTION motion_length(motion, out def, speed, item_model) -> int
  model = item_model IF present ELSE the arms model
  def   = model.motion_definition(motion)
  IF def does not stop at its end THEN RETURN 0
  RETURN round(1000 * motion.length / (def.speed * speed))
```

**Notes** — reporting **zero for a looping motion** is the contract the whole weapon state
machine depends on: a state that waits for its animation to end uses this duration as its
timer, and zero means "never ends on its own". The by-name form additionally returns a
hard-coded 100 milliseconds when the alias does not exist, marked temporary in the source —
a placeholder so a missing animation does not hang the state machine, and a rebuild should
make it an error instead.

## `player_hud::update_inertion`

**Contract** — the weapon lag. Perturbs the rig's transform so the weapon trails the camera
when it turns, swings past and settles, and shifts with pitch. Skipped entirely when the held
item forbids it.

```text
FUNCTION update_inertion(inout trans)
  IF NOT inertion_allowed THEN RETURN
  params = the main-hand item's inertia parameters, or the compiled-in defaults

  # the difference between where we are looking and where the weapon is pointing
  difference = trans.forward - last_direction

  # if they have diverged by more than a quarter turn, re-seat last_direction
  # perpendicular to the current view rather than letting it swing the long way
  IF dot(normalize(last_direction), trans.forward) < epsilon
    axis = cross(last_direction, trans.forward)
    last_direction = cross(trans.forward, axis)
    difference = trans.forward - last_direction

  # aiming uses the aim values, blended toward the hip values by the item's factor
  IF the item is in an aiming mode
    f = item.inertion_factor
    catch_up = lerp(params.tendto_speed_aim, params.tendto_speed, f)
    swing    = lerp(params.origin_offset_aim, params.origin_offset, f)
  ELSE
    catch_up = params.tendto_speed ; swing = params.origin_offset
  scale both by the item's inertia power factor

  last_direction = last_direction + difference * catch_up * frame_delta
  trans.position = trans.position + difference * swing

  pitch = signed normalized pitch of trans.forward, scaled by the item's inertion factor
  trans.position -= trans.forward * pitch * params.pitch_offset_d   # nearer / further
  trans.position -= trans.right   * pitch * params.pitch_offset_r   # sideways
  clamp pitch to [params.pitch_low_limit, half a turn]
  trans.position -= trans.up      * pitch * params.pitch_offset_n   # up / down
```

**Invariants** — the lagged direction is **process-wide state**, not per-item. It persists
across weapon switches, which is deliberate: switching weapons mid-turn should not snap the
new weapon into alignment.

**Notes** — the quarter-turn re-seat is the guard that makes fast turns behave. Without it, a
player spinning past a quarter turn in one frame would have the weapon take the long way
round and swing wildly. Re-seating perpendicular caps the lag at a quarter turn and lets it
resolve from there.

The pitch shift is applied in three axes with separate magnitudes, and only the third is
clamped by the lower limit. The order matters: the depth and sideways shifts read the raw
pitch, the vertical one reads it clamped. That asymmetry is why the lower limit is named for
the vertical offset in the configuration keys.

The catch-up is multiplied by the frame delta and the swing is not. The lagged direction
therefore converges at a rate independent of frame rate while the visible swing is a pure
function of the current difference — which is the right split, and easy to get wrong in the
other direction.

## `player_hud::attach_item` / `detach_item_idx`

**Contract** — attach puts an item's rig representation into its configured slot, detaching
whatever was there and notifying both items. Re-attaching the same item to the same slot is a
no-op except for refreshing the back-reference. Detach clears the slot, notifies the item,
and then does one of two repairs.

```text
FUNCTION detach_item_idx(slot)
  IF slot is empty THEN RETURN
  notify the item it is being detached ; clear the slot and the back-reference

  IF slot WAS the off hand AND the main hand is still filled
    # the main hand's animation was confined to its own partition while both
    # hands were busy; spread it back across every partition
    FOR EACH blend running on the right-hand partition
      play its motion on every other partition and copy the blend's state onto it
  ELSE IF slot WAS the main hand AND the off hand is still filled
    re-trigger the movement notification so the remaining item re-chooses its animation
```

**Invariants** — the animation spread must copy the running blend's *state* (its time,
weight and speed), not merely start the same motion elsewhere, or the newly freed hand starts
its animation from the beginning while the other is halfway through.

**Notes** — the spread is the counterpart of the partition split in `anim_play`. While both
hands are full each animates its own partition; when one empties, the survivor's animation
must reclaim the body. Doing it by copying blend state is fragile — starting a cycle on
another partition can itself advance the animation system and invalidate the blend being
copied from, which the source notes — and a rebuild with a cleaner animation layer should
express "this motion now covers these partitions" directly.

Attachment checks compatibility in both directions: attaching to the main hand asks the off
hand's item whether it tolerates the new item, and `allow_activation` asks the same question
before an item is even drawn. That is how a two-handed weapon prevents a detector being held.

## `player_hud::calc_transform`

**Contract** — places an item. A two-part item is placed at its anchor bone on the arms, then
offset; a monolithic item is placed directly by the rig transform and its offset.

```text
FUNCTION calc_transform(slot, offset, out result)
  IF the slot holds a two-part item
    result = rig_transform ∘ anchor_bone[slot].pose ∘ offset
  ELSE
    result = rig_transform ∘ offset
```

This one function is where the two kinds of item finally differ visibly: one rides the hand,
the other rides the camera.

## `player_hud::render_hud` / `render_item_ui` / `render_item_ui_query`

**Contract** — draws the rig. Returns immediately when nothing is attached, and again when
neither attached item currently wants to be drawn. The arms are submitted first, then each
item. The interface queries ask whether either held item has a world-space display to draw —
a detector's screen, a scope's reticle — and draw it.

**Notes** — the arms are drawn only when at least one item wants drawing. Empty hands are not
rendered at all, which is why the player has no visible arms when holding nothing.

## `player_hud::create_hud_item` / destruction / `inertion_allowed` / `OnMovementChanged`

**Contract** — `create_hud_item` returns the cached rig representation for a section,
building it on first request, and records the section in a process-wide variable the renderer
reads while the model loads. Destruction releases the arms model and every cached item.

`inertion_allowed` asks only the **main-hand** item, and defaults to allowed when the main
hand is empty — the off hand never vetoes inertia.

`OnMovementChanged` tells both held items the player's movement state changed. The stopped
case is special: rather than forwarding, it restarts the idle animation of any item that is
in its idle state, which is what makes a weapon settle when the player stops walking.
