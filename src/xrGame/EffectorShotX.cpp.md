# src/xrGame/EffectorShotX.cpp

> Dead file: an abandoned recoil variant that drove the character's camera angles directly instead of publishing an offset. Nothing is compiled.

**Needs** — [`EffectorShotX.h`](EffectorShotX.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: nothing is built

## Purpose

The file contains no live code. Its entire body is a commented-out subclass of the recoil
effector ([`EffectorShot.cpp`](EffectorShot.cpp.md)) from an earlier design, retained in
the tree as a record of an approach that was tried and dropped.

It is worth one page rather than none because the approach it abandons is a real fork in
the road for a rebuild.

## State

`Stateless.`

## The abandoned design

The variant inverted the direction of control. Rather than accumulating a recoil offset
and letting the character *read* it each frame, it pushed each shot's incremental kick
straight into the first-person camera's pitch and yaw as the shot happened, and published
a zero offset so that nothing read it twice. Its camera step did nothing at all.

Two consequences follow, and they are why the design was dropped:

- writing into the camera's angles means the writer must also reproduce the camera's own
  angle limits — wrapping pitch into range before adding, then clamping both axes against
  the camera's configured bounds. The commented code does exactly this, duplicating logic
  that belongs to the camera;
- there is nothing left to relax. The kick becomes part of the player's aim the instant it
  is applied, so the automatic return the live model offers has no state to return.

The live model's split — accumulate an offset, publish both the offset and its per-frame
delta, let the character decide which to apply where — keeps both behaviours available and
keeps the angle limits in one place.

## Notes

A rebuild should not carry this file forward. It is recorded here so that the absence of
a "shot X" effector is understood as a decision rather than an omission.
