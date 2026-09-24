# src/xrGame/Weapon.cpp

> The base of every weapon: what a weapon is made of (a magazine of cartridges, up to three attachable addons, a zoom rig, a wear level), where its muzzle is this frame, and the lifecycle that carries all of it through spawn, save, network and destruction.

**Needs** — [`Weapon.h`](Weapon.h.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`hud_item_object.h`](hud_item_object.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`firedeps.h`](firedeps.h.md) · [`CameraRecoil.h`](CameraRecoil.h.md) · [`first_bullet_controller.h`](first_bullet_controller.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`Torch.h`](Torch.h.md) · [`WeaponBinocularsVision.h`](WeaponBinocularsVision.h.md) · [`Level.h`](Level.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`player_hud.h`](player_hud.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: game logic over bones and matrices. Two things stop it going higher: the network export/import is a frozen bit-packed layout, and the per-frame fire-point recomputation is on the hot path for every visible weapon.

## Purpose

`CWeapon` is where "an inventory item you can hold" meets "a thing that emits bullets".
It owns neither the firing loop (that is the state machine in
[`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) and its siblings) nor the bullet's flight
and impact (that is the bullet manager and the material system). What it owns is
everything both of those need: the magazine as a list of individual cartridges, the
addon attachment state, the zoom rig, the wear level and the misfire model derived from
it, and — the per-frame heart of the file — the *fire dependencies*: where the muzzle,
the second muzzle, the ejection port and the aim direction are in world space this
frame.

The fire dependencies exist because the answer differs by who is holding the weapon.
For the player the muzzle is a bone on the first-person model; for anybody else it is a
configured offset inside the world model, which itself hangs between two hand bones of
the carrier's skeleton. Both paths are computed lazily, once per frame, and cached.

## State

```text
RECORD Weapon EXTENDS HudItemObject, ShootingObject

  # --- ammunition -------------------------------------------------------
  ammo_types        : list<text>        # sections this weapon accepts, in authored order
  ammo_type         : int (8-bit)       # index into ammo_types: what the next reload loads
  magazine          : list<Cartridge>   # the actual rounds, back() is the next to fire
  ammo_elapsed      : int               # rounds in the magazine
  magazine_size     : int               # capacity
  default_cartridge : Cartridge         # a prototype of ammo_types[ammo_type]
  current_cartridge_dispersion : real   # the accuracy multiplier of the round on top
  next_ammo_type_on_reload : optional<int>
  auto_spawn_ammo   : bool              # creatures killed with it drop a box
  has_tracers       : bool
  tracer_colour_id  : optional<int (8-bit)>
  ammo_total_cached : int               # see the brief-info frame cache below
  brief_info_frame  : int

  # --- addons -----------------------------------------------------------
  addon_state       : int (8-bit)       # bit set: scope | silencer | grenade launcher
  scope_status, silencer_status, launcher_status : AddonStatus
  scopes            : list<text>        # candidate scope sections, when attachable
  current_scope     : int (8-bit)       # index into scopes
  silencer_section, launcher_section : text
  scope_icon_xy, silencer_icon_xy, launcher_icon_xy : (int, int)
  addon_holder_range_modifier, addon_holder_fov_modifier : real

  # --- zoom -------------------------------------------------------------
  zoom_enabled, zoomed_now, hide_crosshair_in_zoom, dof_in_zoom : bool
  current_zoom_factor : real            # a field of view, in degrees
  iron_sight_zoom_factor, scope_zoom_factor : real
  runtime_zoom_factor : real            # remembered across un-zoom, for dynamic zoom
  zoom_rotate_time  : real = 0.25       # seconds to swing into the aim pose
  zoom_rotation_factor : real           # 0 = hip, 1 = fully aimed
  dynamic_zoom      : bool
  zoom_postprocess_section, binocular_vision_section : optional<text>
  binocular_vision  : optional<BinocularsVision>
  night_vision      : optional<NightVisionEffector>
  remembered_actor_nightvision : bool
  scope_overlay     : optional<Window>  # the full-screen scope picture

  # --- condition and recoil ---------------------------------------------
  fire_dispersion_condition_factor : real   # dispersion multiplier at zero condition
  misfire_* (see the misfire model below)
  condition_decrease_per_shot, condition_decrease_per_queue_shot : real
  camera_recoil, zoomed_camera_recoil : CameraRecoil
  per_shot_dispersion_model : PDM           # five posture multipliers
  crosshair_inertion : real = 5.91
  first_bullet_controller : FirstBulletController

  # --- placement --------------------------------------------------------
  loaded_fire_point, loaded_fire_point_2 : vector3    # in the model's own frame
  current_firedeps  : FireDependencies                # the per-frame cache
  hand_offset       : matrix                          # model -> hand
  strap_offset      : matrix                          # model -> shoulder
  strapped          : bool
  can_be_strapped   : bool
  hand_dependence   : {none, one hand, two hands}
  single_handed     : bool

  # --- lifetime and AI --------------------------------------------------
  remove_time       : int = 60000 ms   # how long a dropped weapon survives in multiplayer
  independency_time : int              # server time it was last dropped; 0 while held
  hit_probability[difficulty] : real   # what the AI believes its chance to hit is
  main_weapon_type, weapon_type : int  # the planner's coarse categories
  activation_speed_override : optional<vector3>

RECORD FireDependencies
  muzzle, muzzle_2, ejection_port : vector3   # world space
  aim_direction  : vector3                    # world space, unit
  particle_frame : matrix                     # where muzzle flash and smoke are emitted
```

**Invariants** — these are the ones enforced by scattered assertions and are the most
valuable lines on this page:

1. **`ammo_elapsed` always equals the length of `magazine`.** Every path that changes one
   changes the other in the same breath. The redundancy exists because `ammo_elapsed`
   is what crosses the network and the save file (as a count) while `magazine` is what
   the firing code consumes (as individual rounds, which may differ in type mid-magazine).
2. The fire dependencies are valid only for the frame stamped on them; any reader goes
   through the accessor that recomputes them if the frame has moved on.
3. `zoom_rotation_factor` is clamped to [0, 1] and is the *only* thing that interpolates
   between the hip and aim poses — `zoomed_now` is the target, not the state.
4. A weapon is `strapped` only if the section declared both a strap bone pair and a strap
   position/orientation pair; a partial declaration silently disables strapping.
5. `independency_time` is zero while the weapon has a parent, and the server time it was
   dropped otherwise. That is the whole basis of the dropped-weapon cleanup.

## `Load` — turning a section into a weapon

**Contract** — reads every tuned number from the configuration section. Runs once, at
construction time, before the server record is applied. Does not allocate anything that
outlives the weapon except the scope overlay window.

The order matters in exactly two places: addon *status* must be read before the
per-addon blocks (each block is conditional on its status), and `InitAddons` runs last
because it depends on every one of them.

```text
FUNCTION load(weapon, section)
  base.load(section); shooting_object.load(section)

  ammo_types = the section's "ammo_class" list, in order
  ammo_elapsed = section "ammo_elapsed"; magazine_size = section "ammo_mag_size"

  load_camera_recoil(weapon, section)          # below
  per_shot_dispersion_model = the five PDM_disp_* values, each defaulting to 1
  crosshair_inertion = section "crosshair_inertion", default 5.91
  first_bullet_controller.load(section)
  fire_dispersion_condition_factor = section "fire_dispersion_condition_factor"
  load_misfire_model(weapon, section)          # below
  condition_decrease_per_shot = section "condition_shot_dec"
  condition_decrease_per_queue_shot = section "condition_queue_shot_dec",
                                      defaulting to the single-shot figure

  loaded_fire_point   = section "fire_point"
  loaded_fire_point_2 = section "fire_point2", defaulting to the first
  hand_dependence = section "hand_dependence"   # 0 none, 1 one hand, 2 two hands
  single_handed   = section "single_handed", default true

  scope_status    = section "scope_status"
  silencer_status = section "silencer_status"
  launcher_status = section "grenade_launcher_status"
  zoom_enabled    = section "zoom_enabled"
  zoom_rotate_time = section "zoom_rotate_time", default 0.25

  IF scope_status = attachable THEN
    scopes = the section's "scopes_sect" list, or [section] when absent
              # a weapon with one built-in scope option lists itself
  ELSE IF scope_status = permanent THEN
    scope_zoom_factor = section "scope_zoom_factor"
    build the scope overlay from the named scope texture     # skipped on a dedicated server

  IF silencer_status = attachable THEN read silencer section name and icon offsets
  IF launcher_status = attachable THEN read launcher section name and icon offsets

  init_addons(weapon)

  remove_time      = section "weapon_remove_time", default 60000 ms
  auto_spawn_ammo  = section "auto_spawn_ammo", default true
  hide_crosshair_in_zoom = the HUD section's "zoom_hide_crosshair", default true
  zoom_depth_of_field       = section "zoom_dof"          # absent is the sentinel (-1,-1,-1)
  reload_depth_of_field     = section "reload_dof"
  reload_empty_depth_of_field = section "reload_empty_dof"
  has_tracers      = section "tracers", default true
  tracer_colour_id = section "tracers_color_ID", default none
  FOR EACH difficulty level
    hit_probability[level] = section "hit_probability_<level name>", default 1
  dynamic_zoom     = section "scope_dynamic_zoom", default false
  uses_condition   = section "use_condition", default true
```

**Notes** — the depth-of-field values use an all-`-1` vector as "absent", and the
"enabled" flag is derived by comparing against that sentinel rather than by asking
whether the line existed. A rebuild should use an optional and delete the sentinel.

## Camera recoil — two rigs, one authored

**Contract** — a weapon carries two recoil descriptions, one for hip fire and one for
aimed fire. Only the hip one is fully authored; the aimed one is **copied from it field
by field and then selectively overridden** by any `zoom_cam_*` line present.

```text
RECORD CameraRecoil
  relax_speed        : real   # radians per second the camera returns at (player)
  relax_speed_ai     : real   # and for AI carriers; defaults to the player figure
  dispersion         : real   # radians of camera kick per shot
  dispersion_inc     : real   # extra radians added per shot within a burst
  dispersion_frac    : real = 0.7   # how much of the kick is vertical vs horizontal
  max_angle_vertical, max_angle_horizontal : real   # the kick's ceiling
  step_angle_horizontal : real      # a fixed horizontal drift per shot
  return_mode        : bool   # does the camera walk back to where it started
  stop_return        : bool   # does the return abort when the player moves the view
```

All angles are authored in degrees and stored in radians; all of them are forced to a
small positive epsilon if authored as zero, because they are divisors. The per-shot
`dispersion` and `dispersion_inc` are *not* loaded here but in `LoadFireParams`, which
runs again whenever the effective fire parameters change (an addon, an upgrade) — the
split exists because `dispersion` is the only recoil figure a silencer modifies.

**Notes** — the source explicitly forbids cloning the hip rig wholesale into the aimed
rig ("нельзя!!!"), because the clone helper asserts on the zero-valued fields the aimed
rig legitimately starts with. The field-by-field copy is the workaround. A rebuild
should simply make the aimed rig a set of optional overrides over the hip rig.

## The misfire model

**Contract** — a worn weapon jams. Two formulas ship, selected by which lines the section
carries, and both must be reproduced because both appear in the shipped data of
different games.

```text
FUNCTION misfire_probability(weapon) -> real
  c = weapon.condition          # 1 = pristine, 0 = ruined
  IF weapon uses the old formula THEN
    IF c > 0.95 THEN RETURN 0
    p = base_probability + (1 - c)^3 * condition_coefficient
  ELSE
    IF c > start_condition THEN RETURN 0
    IF c < end_condition   THEN RETURN end_probability
    # linear between the two authored knees
    p = start_probability
      + (start_condition - c) * (end_probability - start_probability)
        / (start_condition - end_condition, or start_condition if they are equal)
  RETURN clamp(p, 0, 0.99)
```

The cubic in the old formula is what makes wear feel harmless until it suddenly is not:
at 0.8 condition the term is 0.008, at 0.3 it is 0.34. The newer formula replaces that
curve with two authored knees so designers can place the cliff.

When a section carries only the old lines, the four new-formula fields are back-filled
with rough equivalents (knees at 0.95 and 0.0, probabilities at the base and at a
quarter of base-plus-coefficient) **purely so the condition readout in the inventory
draws something sensible**. Those four numbers never drive a shot.

## `CheckForMisfire`

**Contract** — rolled once per shot, on the authoritative side only. A jam ends the
burst, latches the jammed flag and switches the weapon to its misfire state; the flag
clears only on a reload.

```text
FUNCTION check_for_misfire(weapon) -> bool
  IF this is a network client THEN RETURN false       # the server decides jams
  IF random in [0,1) < misfire_probability(weapon) THEN
    end firing; weapon.jammed = true; switch to misfire state; RETURN true
  RETURN false
```

## `UpdateXForm` — where the weapon hangs

**Contract** — recomputes the weapon's world transform from its carrier's skeleton, at
most once per frame. The result is a transform whose *forward axis runs from the trigger
hand to the support hand*, which is how a rifle stays pointed along the arms whatever the
animation does.

```text
FUNCTION update_transform(weapon)
  RETURN IF already done this frame; stamp the frame
  parent = weapon.carrier; RETURN IF there is none

  IF parent is not a living entity THEN
    IF multiplayer THEN place weapon at parent's transform
    RETURN
  IF parent is an inventory owner that has this weapon *attached* (on the belt, not held)
    THEN RETURN                                    # the attachment code places it

  ask the carrier for three weapon bones: left hand, right hand, and a fallback left
  RETURN IF there is no right-hand bone

  IF the weapon is one-handed, OR is reloading, OR the carrier is dead THEN
    use the fallback bone as the left                # reload animations free the left hand
  compute the carrier's pose

  D = left_hand_position - right_hand_position
  IF D is degenerate THEN
    result = carrier's own orientation, translated to the right hand
  ELSE
    D = normalize(D)                                 # forward: along the arms
    R = cross(right_hand.up, D)                      # right
    N = normalize(cross(D, R))                       # up
    result = basis(R, N, D) at right_hand_position, then into world space

  place(weapon, result)

FUNCTION place(weapon, transform)
  weapon.position  = transform.translation
  weapon.transform = transform * (strapped ? strap_offset : hand_offset)
```

**Invariants** — the resulting transform must be non-degenerate; a zero-determinant
result means the carrier's pose was not computed and the weapon would vanish.

**Notes** — the "this ugly case is possible for a monster that is neither a stalker nor
the actor" comment marks the reason the bone triple is asked for rather than assumed: not
every carrier has hands in the same places, and some have none.

## `UpdateFireDependencies` — where the muzzle is

**Contract** — the lazily recomputed, once-per-frame cache every shot, particle effect
and trace reads through. Two completely different sources depending on whether the
first-person model is being used.

```text
FUNCTION update_fire_dependencies(weapon)
  RETURN IF already done this frame; stamp the frame
  update_transform(weapon)

  IF the first-person model is active THEN
    take the muzzle, second muzzle, ejection port, aim direction and particle frame
    directly from the first-person model's own measured bones
  ELSE
    # third person, or no carrier at all: the points are authored offsets in the
    # weapon's own model frame
    muzzle         = transform applied to loaded_fire_point
    muzzle_2       = transform applied to loaded_fire_point_2
    ejection_port  = transform applied to loaded_shell_point
    aim_direction  = transform applied to the model's forward axis
    particle_frame = the weapon's transform
```

The split is not cosmetic: the first-person model is a *different mesh* from the world
model, posed by a different animation, and a bullet must leave the muzzle the player can
see. Getting this wrong makes shots visibly miss the crosshair at close range.

## `ForceUpdateFireParticles`

**Contract** — in third person only, rebuilds the particle frame so that muzzle flash and
smoke point along the direction the *carrier's aiming logic* chose, not along the model's
own forward axis. Called immediately before emitting a shot's particles. The basis is
built by taking world-up as a reference, which degenerates when firing straight up or
down — visible as a spinning flash at extreme pitch, and a rebuild should pick the
reference axis away from the aim direction instead.

## Lifecycle

### `net_Spawn`

**Contract** — promotes the server record into a live weapon. Order is load-bearing:
the elapsed count and ammo type must be applied *before* the magazine is filled, and
addon visibility must be established before the addon-dependent parameters are read.

```text
FUNCTION spawn_from_record(weapon, record)
  runtime_zoom_factor = scope_zoom_factor          # before anything can zoom
  base.spawn_from_record(record)

  ammo_elapsed  = record.elapsed
  addon_state   = record.addon_flags
  ammo_type     = record.ammo_type
  set both the current and the pending state to record.weapon_state

  default_cartridge = load(ammo_types[ammo_type])
  IF ammo_elapsed > 0 THEN
    current_cartridge_dispersion = default_cartridge.dispersion
    fill magazine with ammo_elapsed copies of default_cartridge

  update_addons_visibility(weapon)
  init_addons(weapon)
  independency_time = 0
  ammo_was_spawned = false
  IF the weapon flashes on firing THEN create its light
```

**Invariants** — a magazine restored this way is homogeneous, because the record carries
only a count and one type. Mixed magazines therefore **do not survive a save or a level
change**; the mix is lost and the whole magazine becomes the current type. That is a real
behavioural limit of the frozen record format, not an oversight to fix.

### `net_Destroy`

**Contract** — stops both flame particle emitters and the light, destroys the light, and
empties the magazine. Ordering matters only in that the emitters hold positions derived
from the fire dependencies, which the base teardown invalidates.

### `save` / `load`

**Contract** — the save payload, in this exact order, after the base item's:
elapsed count, current scope index, addon bit set, ammo type, whether zoomed, and
whether the actor's night vision was suppressed by this weapon's scope. On load the
addon bit set is applied and addon visibility refreshed *before* the ammo type is read,
and the zoom state is re-entered through the normal zoom-in or zoom-out path rather than
by assignment — so the scope overlay, the binocular vision and the night-vision
suppression all rebuild themselves.

**Invariants** — this is a frozen layout. It is also asymmetric with the network
payload below, which carries different fields in a different order; the two are
independent contracts.

### `net_Export` / `net_Import`

**Contract** — the per-update replication payload:

```text
export: condition as an 8-bit quantity over [0,1]
        needs-update flag (8 bits)
        elapsed count (16 bits)
        addon bit set (8 bits)
        ammo type (8 bits)
        state (8 bits)
        zoomed (8 bits)
```

A weapon reports "needs update" when it is its owner's active item or is currently
firing; the flag lets the server skip holstered weapons.

On import, the addon set is applied and visibility refreshed immediately. The zoom is
applied **only when the carrier is a remote player**, since a local player's zoom is
driven by his own input and must not be overwritten by a round trip.

The ammo count and type are applied **only when the weapon is not mid-action** — firing,
switching or reloading. During those states the local simulation is ahead of the server
and adopting the server's count would visibly rewind the magazine. An out-of-range ammo
type is logged and ignored rather than trusted.

**Notes** — the import reads the flag byte before the elapsed count while the export
writes the flag after the condition and before the count. The orders agree; the naming
in the import (`flags`) does not. Also note the export writes the needs-update flag but
the import reads it into a variable it never uses.

### `OnEvent`

**Contract** — two authoritative events reach a weapon:

- *addon changed* — carries the new addon bit set; re-initializes addons and refreshes
  visibility.
- *weapon state change* — carries the target state, the reload sub-state, the ammo type,
  the low byte of the elapsed count, and the pending next ammo type. A network client
  adopts the elapsed count; everyone runs the state transition. This message is the
  **only** way a state transition crosses the wire, and it is emitted exactly once per
  transition.

### Attach and detach hooks

**Contract** — four hooks bracket the weapon entering and leaving an inventory:

| Hook | Effect |
|---|---|
| before becoming a child | clear the independency timer, leave zoom, clear the pending ammo type |
| after becoming a child | refresh addon visibility |
| before becoming independent | remove the recoil effector, end firing, clear pending, force the hidden state, un-strap, leave zoom, recompute the transform |
| after becoming independent | stamp the independency time with server time, destroy the light, refresh addon visibility |

The transform recompute on the way *out* is what leaves a dropped weapon where the hands
were rather than at the carrier's origin.

### `OnActiveItem` / `OnHiddenItem` / `SendHiddenItem`

**Contract** — becoming the active item refreshes addon visibility, invalidates the
ammo-count cache and enters the showing state. Becoming hidden invalidates the cache,
leaves zoom, clears the pending ammo type, and enters the *hiding* state in single player
but the *hidden* state directly in multiplayer — the holster animation is skipped
because the round trip would make it lie.

`SendHiddenItem` is the client's request to the server to holster: the same state-change
message, with the hiding state, and the weapon marked pending so no further input is
accepted until the server answers.

## `SwitchState` — the one-writer rule

**Contract** — state transitions are authoritative. A network client's call returns
immediately without doing anything. On the server, the target state is recorded as the
*next* state and a state-change message is broadcast; the transition itself happens when
that message comes back through `OnEvent`.

```text
FUNCTION switch_state(weapon, target)
  RETURN IF this is a network client
  weapon.next_state = target
  IF the weapon is locally owned, alive, in an inventory, and we are the server THEN
    broadcast a weapon-state-change carrying
      target, reload sub-state, ammo type, low byte of elapsed, pending ammo type
```

**Invariants** — exactly one message per transition. The comment "just single entry for
given state" marks this; duplicating it desynchronizes the animation from the state.

## `UpdateCL` — the per-frame pass

**Contract** — runs every frame for a weapon that is being rendered or fired.

```text
FUNCTION update_client(weapon)
  base.update_client()
  update_hud_addons_visibility(weapon)
  update the muzzle-flash light
  update both flame particle emitters
  IF multiplayer THEN interpolate the replicated transform

  # the idle fidget
  IF the state machine is settled (next state = current) AND single player
     AND the carrier is the view entity THEN
    IF the carrier is the actor, is not moving, and this is his active item THEN
      IF state is idle for more than 20 seconds, not zoomed,
         no second item in hand, and the HUD tuner is not open THEN
        IF the weapon permits it THEN switch to the inspect state
        reset the sub-state timer

  # scope night vision
  IF a night-vision effector exists and the first-person weapon is not being drawn THEN
    IF it is not running THEN
      IF the actor's torch has night vision on THEN
        remember that and switch the torch's night vision off
      start the effector with the scope's post-process
  ELSE IF we had suppressed the actor's night vision THEN
    clear the flag and switch the torch's night vision back on, with its idle sound

  IF a binocular vision exists THEN update it
```

**Invariants** — the twenty-second idle threshold is what produces the "examine weapon"
fidget; it is measured from the last sub-state change, so any action resets it.

The night-vision handshake is the subtle part: a scope with its own night vision and the
actor's head-mounted night vision must never both be on, so the weapon *takes* the
torch's night vision while zoomed and gives it back on un-zoom. The flag that remembers
it is saved, because a save taken while scoped must restore the torch on load.

## Zoom

### `OnZoomIn`

**Contract** — enters aim mode. The field of view becomes the scope's factor if a scope
is attached, the iron-sight factor otherwise — unless dynamic zoom is on, in which case
the factor the player last dialled in is restored. Applies the weapon's depth-of-field
only when there is *no* scope attached (a scope brings its own optics). Creates the
binocular vision overlay and the night-vision effector lazily, and only when a scope is
attached — both are scope properties, not weapon properties.

### `OnZoomOut`

**Contract** — remembers the current factor for dynamic zoom, restores the default field
of view and the scene's depth of field, resets the sub-state timer (so the idle fidget
does not fire immediately after un-scoping), and destroys the binocular vision and
night-vision effector. The night-vision effector is stopped with a very long fade time
before deletion, which in practice means "cut immediately".

### `ZoomInc` / `ZoomDec`

**Contract** — only meaningful with a scope attached and dynamic zoom enabled. Each step
moves the field of view by a delta derived from the scope's base factor and clamps
between that base factor (most zoomed) and a computed minimum (least zoomed). Note the
sense: *increase* zoom decreases the field-of-view number.

### `CurrentZoomFactor` · `IsRotatingToZoom` · `GetCurrentHudOffsetIdx`

**Contract** — the target factor is the scope's when attached and the iron sight's
otherwise. The weapon is "rotating to zoom" while the interpolation factor is below one,
which the renderer uses to decide whether the scope overlay may be drawn yet. The HUD
offset index is 1 while aiming or un-aiming and 0 at rest — two authored hand poses,
blended by the same factor.

### `UpdateHudAdditonal` — the aim swing

**Contract** — modifies the first-person hand transform toward the aim pose and advances
the interpolation. Runs only for an actor carrier, and only while the interpolation is
in flight in either direction.

```text
FUNCTION update_hud_additional(weapon, transform)
  actor = carrier as an actor; RETURN IF not one
  RETURN unless (zoomed AND factor <= 1) OR (not zoomed AND factor > 0)

  idx = current hud offset index
  offset   = the first-person model's authored hand offset [position][idx] * factor
  rotation = the first-person model's authored hand offset [rotation][idx] * factor

  build a rotation from rotation.x about X, then Y, then Z, in that order
  translate it by offset
  post-multiply it into transform

  IF the actor is aiming THEN factor = factor + frame_time / zoom_rotate_time
  ELSE                        factor = factor - frame_time / zoom_rotate_time
  clamp factor to [0, 1]
```

**Invariants** — the rotation order (X then Y then Z, each post-applied) is part of the
authored pose data's meaning; changing it changes every weapon's aim pose. The
interpolation is driven by *the actor's aiming intent*, not by the weapon's zoom flag, so
the swing starts on the frame the button goes down rather than after the state machine
agrees.

## Addons

### `IsScopeAttached` / `IsSilencerAttached` / `IsGrenadeLauncherAttached`

**Contract** — an addon is present when its status is *permanent*, or when its status is
*attachable* and its bit is set in the addon state. Three statuses exist: disabled,
attachable, permanent.

### `UpdateAddonsVisibility` / `UpdateHUDAddonsVisibility`

**Contract** — drives bone visibility on the world model and the first-person model from
the same three flags. For each of the three addon bones: a *disabled* addon's bone is
forced invisible, a *permanent* one forced visible, and an *attachable* one follows its
bit. Both models' bone caches are invalidated and the world model's pose recomputed, so
that a bone hidden this frame is not still drawn.

Both are suppressed entirely while the in-engine HUD tuning tool is open, because that
tool drives bone visibility itself.

**Notes** — the grenade launcher bone has **two** names across the shipped games. The
first-person path probes for the newer name and falls back to the older one; the world
path simply applies the rule to both names, since a model has at most one of them. This
is data compatibility, and a rebuild must keep both names.

### `InitAddons`

**Contract** — the base does nothing; the magazined weapon overrides it to read every
parameter that depends on which addons are attached. It is called after any change to
the addon state.

## Ammunition

### `GetSuitableAmmoTotal` — and the frame cache

**Contract** — the total rounds available: what is in the magazine plus every matching
box in the carrier's belt and rucksack. Counting walks two containers and is called from
the heads-up display, so the total is cached and recomputed only when the inventory's
modification counter says something changed.

```text
FUNCTION suitable_ammo_total(weapon, include_spawn_reserve) -> int
  IF there is no inventory THEN RETURN ammo_elapsed
  IF the inventory has not changed since the cached frame THEN
    RETURN ammo_elapsed + cached_total
  stamp the cache frame
  cached_total = 0
  FOR EACH type IN ammo_types
    cached_total = cached_total + rounds of that type in belt and rucksack
    IF include_spawn_reserve AND the owner has a deferred item to spawn THEN
      cached_total = cached_total + that item's box count
  RETURN ammo_elapsed + cached_total
```

**Notes** — the deferred-spawn addition is inside the per-type loop, so with three ammo
types it is added three times. That is a bug preserved here only because it feeds an AI
"can this stalker kill" test where over-counting is harmless; a rebuild should add it
once, outside the loop.

The cache is invalidated by writing zero to the stamped frame, which every state change
and every activation does.

### `SetAmmoElapsed`

**Contract** — the single place the count and the magazine are reconciled. Growing pushes
copies of the current ammo type; shrinking pops from the back. This is what a network
client uses to adopt the server's count, and it is why a client's magazine becomes
homogeneous after any correction.

### `SwitchAmmoType`

**Contract** — cycles to the next ammo type the carrier actually has, and arranges for
the *next reload* to use it — it does not change the rounds already in the magazine.

```text
FUNCTION switch_ammo_type(weapon, flags) -> bool
  RETURN false IF pending, or if this is a network client
  RETURN false unless this is a key-down
  candidate = ammo_type
  REPEAT
    candidate = (candidate + 1) MOD count(ammo_types)
  WHILE candidate <> ammo_type AND the inventory holds none of that type
        AND ammo is not unlimited
  IF candidate <> ammo_type THEN
    next_ammo_type_on_reload = candidate
    IF we are the server THEN reload
  RETURN true
```

The loop terminates because it stops when it wraps back to the starting type.

### `SpawnAmmo`

**Contract** — creates ammunition boxes in the world (or into a named parent's
inventory), splitting a requested count across as many full boxes as needed. Runs on the
authoritative side only. Each box is written as a spawn record and sent as a local spawn
message rather than constructed directly, so it goes through the same path as authored
content. The box's navigation vertex is the weapon's own.

**Notes** — the function computes a rotating ammo-type index and then never uses it; the
type always comes from the caller or from the first entry.

### `GetMagazineWeight`

**Contract** — the weight of the rounds in the magazine, summed per round. Because a
magazine is almost always homogeneous, the per-round weight lookup is memoized on the
section pointer from the previous round — a one-element cache that turns *N* lookups into
one in the common case and degrades gracefully for a mixed magazine.

### `Weight` / `Cost`

**Contract** — both are the base item's figure plus the attached addons' figures plus the
ammunition. Weight adds the magazine's rounds; cost adds a *fraction of a box's price*
proportional to the rounds carried, floored to an integer.

## Dropped-weapon cleanup

**Contract** — in multiplayer only, an ownerless weapon that is not remotely owned is
destroyed after its removal time has elapsed. A console variable overrides: `-1` never
removes, `0` removes immediately, anything else uses the timer. Single player never
removes weapons, because a dropped weapon there is the player's stash.

## AI-facing predicates

### `can_kill` (three forms) · `ready_to_kill`

**Contract** — the planner's questions about a weapon.

- `can_kill()` — true if any suitable ammunition exists anywhere reachable, or if the
  weapon needs no ammunition at all (a knife has an empty ammo-type list).
- `can_kill(inventory)` — returns *the item that makes it possible*: the weapon itself if
  it is already loaded or needs no ammunition, otherwise the first matching ammunition
  box found, otherwise nothing. Returning the item rather than a boolean is what lets the
  planner emit a "pick that up" step.
- `can_kill(items)` — the same over an arbitrary list, used when evaluating a corpse.
- `ready_to_kill()` — true only when the weapon is the carrier's active item, is not
  jammed, is idle or firing, and has rounds. This is the "can shoot *right now*" test,
  and the active-item check is what stops a stalker trying to fire a holstered rifle.

### `hit_probability`

**Contract** — the weapon's authored chance of hitting, per difficulty. **It always
returns the novice entry regardless of the current difficulty**, which makes the other
three authored values dead data. The assertion immediately above confirms the current
difficulty is in range and then ignores it. Whether this is a bug or a deliberate
flattening is not recoverable from the source.

### `ef_main_weapon_type` / `ef_weapon_type`

**Contract** — two coarse integers from the section that the planner uses to pick a
weapon for a situation (pistol vs rifle vs launcher, and a finer class). Both assert they
were authored; a weapon reaching a stalker without them is a data error.

### `modify_holder_params`

**Contract** — a scoped weapon multiplies its carrier's *sight range and field of view*.
The modifiers are cached on the weapon at reload time rather than read from the scope,
because the scope object does not exist once attached — only its section name survives.
That caching is the reason the two fields exist at all.

## Input

### `Action`

**Contract** — the base weapon handles four bindings; everything else falls through to
the subclasses.

| Binding | Behaviour |
|---|---|
| fire | refused while pending; on press, start firing (and buzz the gamepad triggers if the carrier is the actor and this is not a knife); on release, end firing |
| next ammo | cycle the ammo type |
| zoom | see below |
| zoom in / zoom out | step the dynamic zoom, only while already zoomed |

Zoom has two authored behaviours selected by a console variable: *toggle*, where a press
alternates; and *hold*, where a press aims and a release un-aims. In both, entering zoom
first forces the weapon to idle if it is in any other state, and is refused entirely
while pending.

## Rendering

### `need_renderable` · `renderable_Render` · `render_hud_mode`

**Contract** — a weapon skips its ordinary rendering when it is fully scoped: zoomed, no
longer swinging into the pose, and carrying a scope overlay — because in that case the
overlay *replaces* the first-person model. The render path recomputes the transform,
draws the muzzle light, then draws the first-person model with a flag saying whether the
weapon body itself should appear.

**Notes** — the render path takes a lock. The weapon's transform can be recomputed from
the render thread while the simulation is also touching it; the lock is the seam where
that race was patched. A rebuild should instead publish an immutable per-frame transform
from the simulation and let the renderer read it without synchronization.

### `render_item_ui_query` / `render_item_ui`

**Contract** — the scope overlay (and the binocular vision's marks over it) is drawn as a
full-screen item UI, but only when this weapon is the active item, is fully scoped, and
hides the crosshair. Two further predicates govern the rest of the heads-up display:
the crosshair is shown when the weapon is not pending and is either not zoomed or does
not hide the crosshair; the ammunition indicators are hidden whenever a scope overlay is
up.

### The scope overlay document

**Contract** — every scope picture in the game is one node of a single shared layout
document, loaded once, rebuilt on a UI reset and dropped at shutdown. A weapon's overlay
is that document's node named by the scope's `scope_texture`. One of the three shipped
games is excluded from this path entirely (its scope pictures are drawn another way), so
the loader returns immediately in that mode.

## Small surfaces

- `IsValid` — has rounds. `IsUpdating` — is the active item or is currently firing;
  decides whether the weapon is worth replicating.
- `IsNecessaryItem(section)` — is this section one of my ammunition types; used by the
  AI's "should I pick this up" logic.
- `unlimited_ammo` — true in single player when the owner has the unlimited-ammo flag
  *and* the cartridge type permits it, and in deathmatch under the same cartridge
  condition. The per-cartridge permission is what keeps special ammunition finite even in
  unlimited modes.
- `ActivationSpeedOverriden` / `SetActivationSpeedOverride` — a one-shot override for the
  velocity a weapon is thrown with when it becomes independent; consumed on read.
- `GetConditionToShow` — the condition as the inventory draws it; identity today, with a
  commented-out fourth power that used to exaggerate wear.
- `reinit` / `reload(section)` — re-read the section without respawning. `reload` is
  where the hand and strap offsets are built from authored position/orientation pairs
  (degrees to radians, rotation then translation), where strapping is enabled only if
  *all four* strap lines exist, and where the scope's holder modifiers are cached.
- `AllowBore` — may the idle fidget play; always yes here, overridden by weapons whose
  animation set lacks it.
- `MovingAnimAllowedNow` — walking animations are suppressed while zoomed.
- `Hit` — a weapon takes damage like any object; nothing weapon-specific happens.
- `DumpActiveParams` / `GetAnticheatSectionName` — the multiplayer integrity check's view
  of a weapon's effective parameters.
