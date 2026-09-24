# src/xrPhysics/DisablingParams.h

> Declares the thresholds that decide when a body has stopped moving enough to be
> put to sleep.

**Needs** — [`DisablingParams.cpp`](DisablingParams.cpp.md) · [`xrCore`](../xrCore/README.md)
**Used by** — [`DisablingParams.cpp`](DisablingParams.cpp.md) · [`PHDisabling.cpp`](PHDisabling.cpp.md) · [`PHDisabling.h`](PHDisabling.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md)
**Tier floor** — T3: four numbers and a loader.

## Purpose

Declares the record whose values and loading are described in
[`DisablingParams.cpp`](DisablingParams.cpp.md).

## Exported units

- **`SOneDDOParams`** — one axis of the test: a velocity threshold and an acceleration
  threshold, with a scaling operation.
- **`SAllDDOParams`** — a whole object's settings: the translational pair, the rotational
  pair, and the long-window length. Can reset to the world defaults and load overrides from
  a configuration section.
- **`SAllDDWParams`** — the world's settings: the default object settings plus the
  re-enable margin.
- **the world defaults** — one shared instance every object starts from.
