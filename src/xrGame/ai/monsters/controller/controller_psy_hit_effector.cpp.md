# src/xrGame/ai/monsters/controller/controller_psy_hit_effector.cpp

> Dead file: the bodies of the abandoned psi-attack effectors, entirely commented out.

**Needs** — [`controller_psy_hit_effector.h`](controller_psy_hit_effector.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: the translation unit is empty

## Purpose

Compiles to nothing. Every definition in it is commented out, matching the declarations in
[`controller_psy_hit_effector.h`](controller_psy_hit_effector.h.md).

## State

Stateless.

## What the abandoned bodies said

The post-process effector's factor would have been computed as the player's distance mapped
linearly across the band between an authored minimum and maximum fraction of a radius, and
clamped to a hundredth at the far end and one at the near end — so the effect would fade in
with proximity rather than switching on. Its start condition would have been the player
entering the outer fraction of the radius.

The camera effector's body was never written; only a section banner for it survives.

A rebuild should delete this file. It is recorded only so the mirror is complete and so the
abandoned proximity-aura idea is not mistaken for a missing feature.
