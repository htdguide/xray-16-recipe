# src/xrGame/CameraEffector.cpp

> Empty: the camera-effector types are entirely declared in [`CameraEffector.h`](CameraEffector.h.md).

**Needs** — [`CameraEffector.h`](CameraEffector.h.md)
**Used by** — reached through its declarations in [`CameraEffector.h`](CameraEffector.h.md); callers name that, not this file.
**Tier floor** — T4: nothing compiles from this file

## Purpose

The file contains only the precompiled-header include. It exists because the build system
expects a compilation unit beside the header; nothing in it is a decision.

The substance is in [`CameraEffector.h`](CameraEffector.h.md), which declares the effect-type
enumeration and the game's effector base classes. A rebuild should delete this file.

## State

`Stateless.`
