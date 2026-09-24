# src/xrGame/BlackGraviArtifact.h

> Declares the shockwave artefact implemented in [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md).

**Needs** — [`GraviArtifact.h`](GraviArtifact.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md)
**Used by** — [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CBlackGraviArtefact`: the hovering artefact, plus a touch sense, plus a radial
shockwave released when the artefact is struck hard. Substance is in
[`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md).

Exported units:

- `CBlackGraviArtefact` — the artefact. Holds the arming impulse threshold, the strike
  radius and impulse, the armed flag, the particle name, the list of nearby physical objects
  and a reusable ray-query buffer.
- `Load` / `net_Spawn` — tuning, and a decorative particle effect created at spawn.
- `Hit` — arm above the impulse threshold; suppress the hit's own push.
- `UpdateCLChild` — hover as the base class does, then release the shockwave if armed.
- `GraviStrike` — the radial effect: quadratic falloff, line-of-sight attenuation through
  the shared explosion routine, and a hit per affected bone. **See the implementation note:
  as shipped it sends no hits.**
- `feel_touch_new` / `feel_touch_delete` / `feel_touch_contact` — the sense, excluding other
  artefacts so a cluster cannot chain-react.
- `net_Relcase` — drop a destroyed object from the nearby list.

## Notes

**The reusable ray-query buffer is a member, not a local.** That is an allocation
optimization: the shockwave casts many rays and the buffer is reused across sweeps. A
rebuild on a tier with cheap allocation should make it a local and delete the field.
