# src/xrGame/BastArtifact.h

> Declares the self-hurling artefact implemented in [`BastArtifact.cpp`](BastArtifact.cpp.md).

**Needs** — [`Artefact.h`](Artefact.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md)
**Used by** — [`BastArtifact.cpp`](BastArtifact.cpp.md) · [`artefact_script.cpp`](artefact_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CBastArtefact`, the artefact that charges when shot and throws itself at nearby
creatures. Substance is in [`BastArtifact.cpp`](BastArtifact.cpp.md).

Its load-bearing content is the mix-in: this artefact is also a **touch sense**, which is
how it knows who is nearby. That combination — an item that senses — is rare in the chapter
and worth noticing.

Exported units:

- `CBastArtefact` — the artefact. Holds the impulse threshold that wakes it, the sense
  radius, the per-strike impulse, the energy pool and its decay rate, the particle name, the
  armed flag, the nearby-creature list, the current victim and the last one struck.
- `Load` / `net_Spawn` / `net_Destroy` — tuning and the lifecycle; both lifecycle hooks
  simply clear the behavioural state.
- `Hit` — the trigger: above the impulse threshold, charge up and arm, and suppress the
  hit's own physical push.
- `UpdateCLChild` — choose a victim, steer toward it, emit particles, decay.
- `feel_touch_new` / `feel_touch_delete` / `feel_touch_contact` — the sense, filtered to
  living creatures.
- `shedule_Update` — refresh the sense at the configured radius.
- `Useful` — false while charged, so a live artefact cannot be picked up.
- `IsAttacking` — read by the contact callback to skip dormant artefacts cheaply.
- `ObjectContactCallback` / `BastCollision` — the physics contact hook and its handler.
- `setup_physic_shell` — install that hook and clear the generic one.

## Notes

**No reference-release hook is declared.** A creature destroyed while in the nearby list
leaves a dangling entry; the sibling class in
[`BlackGraviArtifact.h`](BlackGraviArtifact.h.md) does implement one. A rebuild should
either add it or use a representation that cannot dangle.
