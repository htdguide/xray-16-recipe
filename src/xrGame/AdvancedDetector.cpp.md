# src/xrGame/AdvancedDetector.cpp

> The directional artefact detector: a hand-held device whose needle points at the nearest hidden artefact and whose beeping speeds up as you close on it.

**Needs** — [`AdvancedDetector.h`](AdvancedDetector.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`Artefact.h`](Artefact.h.md) · [`ui/ArtefactDetectorUI.h`](ui/ArtefactDetectorUI.h.md) · [`player_hud.h`](player_hud.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame bone transform override on the first-person model

## Purpose

There are three grades of artefact detector and this is the middle one. Where the basic
detector only tells you an artefact is *near*, this one tells you **which way**, by
rotating a needle bone on the first-person model toward the artefact and by modulating the
pitch and rate of a beep.

The interesting decision is that the needle is not a widget: it is a bone on the device's
own model, rotated by a callback during skeleton evaluation. The *interface* for this
device is geometry.

## State

```text
  artefact_rank    : int = 2          # which grade of artefact this detector can see
  target_direction : vector           # world direction to the nearest artefact; zero means none
  current_rotation : real             # the needle's present angle, eased toward the target
  needle_bone      : bone index       # resolved by name from the device's model
```

**Invariant** — the rank is fixed at construction and is the only thing distinguishing the
three detector grades' *detection*; everything else about this class is presentation. A
higher-ranked artefact is invisible to a lower-ranked detector.

## `UpdateAf`

**Contract** — the per-update sweep. Clears the needle first, so an emptied detection list
leaves the needle centred. Then finds the **nearest** artefact not already owned by anyone,
points the needle at it, and schedules the next beep.

```text
FUNCTION update_artefacts()
  needle = none
  IF nothing detected THEN RETURN

  nearest = the detected artefact with the smallest distance to the detector,
            skipping any that already has an owner
  ALSO, while scanning: any artefact that can be invisible and is closer than
       the reveal radius is made visible
       # the detector does not merely find artefacts — it UNHIDES them

  closeness = clamp(distance / detect_radius, 0, 1)   # 0 is touching, 1 is at the edge

  # The needle points from the CAMERA, not from the device.
  direction = normalize(artefact_position - camera_position)
  offset    = signed angle between that heading and the camera's heading

  # Beep interval: interpolated between the artefact type's two authored
  # frequencies, by the SQUARE of closeness — so the acceleration is
  # concentrated in the last part of the approach.
  period = min_period + (max_period - min_period) * closeness^2
  # Beep pitch: linear in closeness, from 1.4 down to 0.9.
  pitch  = 0.9 + 0.5 * (1 - closeness)

  IF the beep timer has exceeded the period THEN
    reset it and play the artefact type's detect sound at that pitch
  ELSE advance the beep timer

  needle = offset and direction
```

**Invariants** — the invisible-artefact reveal is a *side effect of scanning* and applies to
every candidate within the reveal radius, not only the nearest. That is the mechanic: the
detector makes artefacts appear.

**Notes**

- Measuring direction from the camera rather than from the device means the needle reads
  correctly as an on-screen instrument regardless of where the device is held. A rebuild
  measuring from the device will produce a needle that swings when the player lowers the
  gun.
- The squared interpolation of the beep interval is the whole feel of the device: linear
  would make the approach feel uniform, and the square makes the last few metres frantic.
- The pitch interpolation runs the opposite way to the period, so a near artefact beeps
  *faster* and *lower*. That inversion is deliberate and is what makes the two cues
  distinguishable.

## `CreateUI` · `ui`

**Contract** — constructs the device's presentation object and binds it to this detector.
Asserted to happen exactly once.

## `on_a_hud_attach` · `on_b_hud_detach`

**Contract** — installs and removes the needle bone callback when the device is raised into
and lowered out of the player's hands. The callback cannot exist while the first-person
model does not.

## the needle presentation

**Contract** — three pieces on the presentation object:

- **`SetBoneCallbacks`** — resolves the needle bone by a literal name on the device's
  first-person model, installs the rotation callback on it, and seeds the current angle from
  the bone's authored rest orientation so the needle does not jump on the first frame.
- **`update`** — hides the needle bone entirely when there is no target, and otherwise
  transforms the world-space target direction into the *device's* local frame and eases the
  needle toward it with an inertial approach that has a minimum speed, a maximum speed and
  an acceleration. A hand-rolled proportional controller is present in the source, commented
  out, and replaced by the shared inertial-angle helper.
- **`CurrentYRotation`** — the angle actually applied, **quantized to one twenty-fourth of a
  full turn**. The needle therefore snaps between twenty-four discrete positions rather than
  sweeping smoothly.

**Invariants** — the quantization is the device's character: it reads as a mechanical
instrument with detents rather than as a smooth gauge. A rebuild that omits it changes how
the device looks in a way players notice.

**Notes** — hiding the bone rather than centring the needle is how "no artefact" is
expressed. The bone-visibility toggle is applied only on a change, because the call
propagates to children.
