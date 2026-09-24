# src/xrGame/MercuryBall.h

> Declares the rolling artefact implemented in [`MercuryBall.cpp`](MercuryBall.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md) · [`MercuryBall.cpp`](MercuryBall.cpp.md)
**Used by** — [`MercuryBall.cpp`](MercuryBall.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CMercuryBall`, the artefact that shoves itself along the ground on a timer.
Substance is in [`MercuryBall.cpp`](MercuryBall.cpp.md).

Exported units:

- `CMercuryBall` — the artefact, carrying the last-decision time, the decision interval and
  the impulse range.
- `Load` — reads the interval and impulse range from the section.
- `UpdateCLChild` — the periodic random horizontal shove, or the carried-item transform
  follow.

**Notes** — the header carries a commented-out earlier design in which the mercury ball was a
plain game object with a lifetime: it appeared after an emission, persisted briefly and
evaporated, and had to be kept in a shielded container. None of that survives in the shipped
class, which is an ordinary artefact that rolls. The abandoned design is recorded here because
its vocabulary (post-emission spawning, containment) reappears in the configuration data.
