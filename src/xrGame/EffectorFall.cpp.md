# src/xrGame/EffectorFall.cpp

> Two one-shot camera effectors: the knee-bend dip after a landing, and a timed override of the depth-of-field parameters.

**Needs** — [`EffectorFall.h`](EffectorFall.h.md) · [`CameraEffector.h`](CameraEffector.h.md) · [`GamePersistent.h`](GamePersistent.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: one scalar curve per frame, plus a render-parameter handoff

## Purpose

Two unrelated effectors share the file because both are short-lived entries in the camera
chain that retire themselves. Their common shape is the important thing: a transient
effector runs until it decides it is finished, then marks its own lifetime expired and the
camera manager removes and destroys it on the next pass. Nothing outside holds a handle.

The fall dip translates a landing impact into a single downward camera excursion and back.
The depth-of-field effector does not move the camera at all — it pushes a render parameter
set when it is created, and exists solely to restore the previous set after a delay.

## State

```text
RECORD FallEffector
  power : real    # in [0,1]; the squared, clamped landing severity
  phase : real    # 0 at the landing, 1 when the dip has fully returned

RECORD DofEffector
  restore_at : real    # absolute clock time at which the previous parameters return
```

## `CEffectorFall`

**Contract** — constructed with a landing severity and a lifetime. The severity is clamped
into the unit range and then **squared**, so a light landing produces almost no dip and a
hard one nearly the whole excursion. The effector declares itself not to affect the
first-person weapon model — the hands do not dip with the head, because the weapon has its
own landing animation and doubling them looks wrong.

## `ProcessCam` *(fall)*

**Contract** — advances the phase at a fixed rate and lowers the camera by one half-period
of a sine, which starts at zero, reaches full depth at the midpoint and returns to zero.
Past the end of the curve the effector expires itself rather than clamping at zero, so it
stops costing anything.

```text
FUNCTION process_cam(camera)
  phase = phase + fall_rate * frame_seconds     # fall_rate 3.5 -> the dip lasts ~0.29 s
  IF phase < 1 THEN
    # sin(pi*phase + pi) runs 0 -> -1 -> 0, so this subtraction dips and returns
    camera.p.height = camera.p.height - max_dip * power * sin(pi * phase + pi)
  ELSE
    expire self
  RETURN keep
```

**Notes**

- The dip's duration is fixed by a compiled-in rate and its depth by a compiled-in
  maximum; only the severity varies. So every landing takes the same time and differs only
  in how far the camera drops — which is why a fall from any height reads as the same
  motion, harder.
- The phase drives a curve whose own lifetime is shorter than the effector's declared
  lifetime, and the curve is what actually ends it. The declared lifetime is a backstop.

## `CEffectorDOF`

**Contract** — constructed with a depth-of-field parameter triple and a duration.
Construction immediately pushes the parameters into the persistent game state, where the
renderer reads them; the effector then merely waits. It never touches the camera.

## `ProcessCam` *(depth of field)*

**Contract** — once the wall clock passes the stored deadline, restores the previous
parameter set and expires. Until then it does nothing at all.

**Invariants** — the push at construction and the restore at expiry must pair exactly, and
the persistent state keeps only *one* saved set: two of these alive at once would leave
the renderer with the wrong parameters when the first one expires. Nothing enforces this,
and it is the kind of invariant a rebuild should make structural by making the saved set a
stack.

**Notes** — building this as a camera effector is a convenience, not a design: it is
riding the camera chain purely because the chain already provides "run me every frame and
destroy me when I say so". A rebuild with a general timed-callback facility should use
that and keep the pairing invariant.
