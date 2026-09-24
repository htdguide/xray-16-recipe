# src/xrGame/cta_game_artefact.h

> Declares the capture-the-artefact objective artefact, implemented in [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md) · [`game_base.h`](game_base.h.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md)
**Used by** — [`artefact_script.cpp`](artefact_script.cpp.md) · [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the artefact subclass used as the objective in the capture-the-artefact mode.
Substance is in [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md).

Exported units:

- `CtaGameArtefact` — the objective. Holds the mode's client rules, its own home point and
  its own team, the last two resolved lazily from the mode.
- `Action` — intercepts use: activation is blocked when the mode forbids it or when the
  artefact belongs to the other team.
- `UpdateCLChild` — per frame: follow the carrier, resolve the home point, settle at base.
- `CreateArtefactActivation` — repurposes activation into "return to base": emit an
  ownership rejection naming the home point as the drop position.
- `OnAnimationEnd` — guarded against the carrier dying mid-animation.
- `OnStateSwitch` / `CanTake` / `PH_A_CrPr` — overrides that add nothing live; each wraps
  disabled code described in the implementation twin.
- `InitializeArtefactRPoint` / `IsMyTeamArtefact` — the two team-resolution helpers.
