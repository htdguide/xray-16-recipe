# src/xrGame/SimpleDetector.cpp

> The cheapest artefact detector: it beeps and blinks faster the closer the nearest artefact is, and shows nothing else.

**Needs** — [`SimpleDetector.h`](SimpleDetector.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`Artefact.h`](Artefact.h.md) · [`player_hud.h`](player_hud.h.md) · [`ui/ArtefactDetectorUI.h`](ui/ArtefactDetectorUI.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a distance search, a period interpolation and a few bone and light writes

## Purpose

There is a family of artefact detectors of increasing capability — this one, one that
shows a direction, one that shows a map. The generic detector already does everything they
share: maintain the set of artefacts within range, track each one's detection parameters
by its rank, own a first-person model, and drive a display. This file supplies only what
the cheapest of them does differently, which is: reduce *all* of that information to one
number — the distance to the nearest artefact — and express it as a beep rate and a blink.

The reduction is the design. The device is deliberately bad at its job, and the rebuild
must reproduce that: it finds *an* artefact, not *the* artefact, and it cannot tell you
which way to walk.

It also carries the implementation of its own display, which is a small piece of hardware
rather than a screen: two lights and two bones on the held model.

## State

`Stateless` beyond the generic detector's, with one value set at construction: this device
detects only artefacts of **rank one**, the commonest. Rarer artefacts are invisible to
it, which is what the more expensive detectors are for.

## `UpdateAf`

**Contract** — called each frame while the detector is held and switched on. Scans the
tracked artefacts for the nearest one that is lying loose in the world, converts its
distance into a beep period and a sound pitch, and emits a beep and a display flash when
the period has elapsed. Does nothing when nothing is tracked. As a side effect, it reveals
artefacts that are close enough and were hidden.

```text
FUNCTION UpdateAf()
  IF tracked_artefacts IS empty THEN RETURN

  nearest <- none ; min_dist <- infinity
  FOR EACH (artefact, info) IN tracked_artefacts
    IF artefact HAS a parent THEN CONTINUE      # carried by someone: not lying in the world
    d <- distance(self.position, artefact.position)
    IF d < min_dist THEN min_dist <- d ; nearest <- (artefact, info)
    IF artefact CAN be invisible AND d < reveal_radius THEN artefact.become_visible()

  info <- nearest.info
  kind <- info.current_kind                     # the detection parameters for this artefact's rank

  closeness <- clamp(min_dist / detect_radius, 0, 1)   # 0 at the artefact, 1 at the edge

  info.period <- kind.period_near
               + (kind.period_far - kind.period_near) * closeness * closeness   # see note
  pitch <- 0.9 + (1.4 - 0.9) * (1 - closeness)

  IF info.since_last_beep > info.period THEN
    info.since_last_beep <- 0
    play the artefact kind's detection sound, attached to this device
    display.flash(on, strength: closeness)
    set the playing sound's pitch to `pitch`
  ELSE
    info.since_last_beep <- info.since_last_beep + frame_delta
```

**Invariants**

- Only loose artefacts count. One already in somebody's rucksack must not make the
  detector chatter, and the parent test is the whole of that rule.
- The revealing of hidden artefacts happens for **every** tracked artefact in range, not
  only the nearest. A detector held near a cluster uncovers the cluster.
- The nearest artefact is recomputed from scratch every frame and no hysteresis is
  applied, so two artefacts at nearly equal distance make the beep rate flicker between
  their two kinds. In practice they are rarely the same rank, so it is rarely visible.

**Notes** — the beep period interpolates against the **square** of the normalized
distance, not against the distance. The effect is that the rate barely changes across most
of the detector's range and then climbs sharply in the last fraction of it. That is the
whole feel of the device: it tells you nothing useful until you are almost on top of
something, and then it becomes frantic. A linear interpolation makes it a usable ranging
instrument and ruins it.

Pitch moves the opposite way, from a low tone far out to a high one close in, and it moves
*linearly*, so the two cues are deliberately out of step: pitch gives a coarse continuous
sense of range while rate gives the sharp final cue.

The pitch is set on the sound *after* it has been started, so the first instant of each
beep plays at the sound's authored rate. At these durations it is inaudible. A rebuild
should set the rate at start.

## `CreateUI`

**Contract** — builds this detector's display object and binds it to this device. Requires
that no display exists yet; a second call is a programming error, not a condition to
recover from.

## `ui`

**Contract** — the display, narrowed to this detector's own type. The generic detector
holds the display as the shared base type; every concrete detector needs its own.

## The display: `construct`, `setup_internals`, `update`, `Flash`

The "user interface" of this device is not a screen. It is two indicator lights on the
held model, and this class drives them.

**`construct`** — attaches the display to its detector and marks both indicator bones as
unresolved, then puts the flash indicator out. Nothing on the model is touched yet: the
first-person model does not exist until the device is actually equipped.

**`setup_internals`** — runs once, the first frame after the first-person model exists.
Creates the two lights, finds the two indicator bones by name in the model's skeleton,
sets the initial bone visibility, and looks up the two colour animations by name.

```text
FUNCTION setup_internals()
  REQUIRE neither light exists yet AND neither bone is resolved yet
  FOR EACH (light, range_key) IN ((flash_light, "flash_light_range"),
                                  (power_light, "onoff_light_range"))
    light <- renderer.create_light()
    light.casts_shadows <- false          # an indicator lamp must not cost a shadow map
    light.kind          <- point
    light.range         <- section[range_key]
    light.first_person  <- true           # see note

  flash_bone <- model.bone_named("light_bone_2")
  power_bone <- model.bone_named("light_bone_1")
  flash_bone.visible <- false             # the flash lamp is off until a beep
  power_bone.visible <- true              # the power lamp is on while the device is held

  on_off_animation <- colour_animation_named("det_on_off")
  flash_animation  <- colour_animation_named("det_flash")
```

**Invariants** — the bone names, the two configuration key names and the two colour
animation names are **data contracts** with the shipped model and configuration. A rebuild
must use the same strings or reauthor the assets.

The first-person flag on both lights is load-bearing: a first-person light is rendered in
the held-weapon pass with its own depth range, so the lamp lights the device in the
player's hands without spilling into the world or being clipped by it.

**`Flash`** — turns the flash indicator on or off. On means: make the flash bone visible,
activate the flash light, and schedule the indicator to go out after a delay proportional
to the closeness the caller passed — a near artefact produces a longer, brighter-looking
pulse than a far one. Off means: hide the bone, deactivate the light, cancel the schedule.
Does nothing at all when the device is not currently held in first person, which is the
only state in which its model exists.

**`update`** — each frame, while held: resolve the indicator bones on first use, extinguish
the flash indicator if its scheduled time has passed, and move both lights onto their
attachment points on the current pose. The power indicator is kept lit permanently and its
colour is driven by the "on/off" colour animation sampled at the global clock, which is
what makes it pulse slowly. The flash light is only repositioned while it is active.

**Invariants** — the lights follow the model's *fire-dependency* points, the same named
attachment points a weapon's muzzle uses, because that is the mechanism the first-person
model already has for "a point on this model in world space".

## Notes

The flash indicator's linger is computed as the closeness value read as a whole number of
milliseconds — that is, a value between zero and one is scaled by a thousand. At the far
edge of the range that is one second and at the artefact itself it is zero, which is
**backwards** from the interpolation's own sense: the nearer the artefact, the *shorter*
the flash. Combined with a beep rate that climbs steeply when near, the visible result is
a long dim glow far out and rapid short blinks close in, which reads correctly even though
the arithmetic reads inverted. **Could not recover**: whether the inversion was intended.
