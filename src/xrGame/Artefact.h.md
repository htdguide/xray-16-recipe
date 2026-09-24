# src/xrGame/Artefact.h

> Declares the artefact base class and its detector-support helper, both implemented in [`Artefact.cpp`](Artefact.cpp.md).

**Needs** — [`hud_item_object.h`](hud_item_object.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`xrPhysics/PHUpdateObject.h`](../xrPhysics/PHUpdateObject.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`AdvancedDetector.cpp`](AdvancedDetector.cpp.md) · [`Artefact.cpp`](Artefact.cpp.md) · [`BastArtifact.cpp`](BastArtifact.cpp.md) · [`BastArtifact.h`](BastArtifact.h.md) · [`BlackDrops.cpp`](BlackDrops.cpp.md) · [`BlackDrops.h`](BlackDrops.h.md) · [`CustomDetector.cpp`](CustomDetector.cpp.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`CustomZone.cpp`](CustomZone.cpp.md) · _and 37 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CArtefact`, the base of every artefact in the game, and
`SArtefactDetectorsSupport`, the optional behaviour that makes an artefact invisible and
mobile until a detector finds it. Substance is in [`Artefact.cpp`](Artefact.cpp.md).

Its own load-bearing content is the **shape of the class**: an artefact is simultaneously a
held first-person item, a physics-step participant, and a carrier of passive effects. Those
three roles have different update paths and different lifetime rules, and a reader needs
that before the implementation makes sense.

Exported units:

- `CArtefact` — the artefact. Five restoration rates and an immunity table (the carry
  effect), a light and a particle effect (the loose presentation), an optional activation
  object and an optional detector support.
- `Load` / `net_Spawn` / `net_Destroy` — the lifecycle.
- `OnH_A_Chield` / `OnH_B_Independent` — the ownership transitions that start and stop the
  presentation.
- `UpdateCL` / `shedule_Update` / `UpdateWorkload` — the two update paths and the shared body
  they both call, selected by the fast-mode flag.
- `UpdateCLChild` — the subclass hook; empty here. Every behavioural artefact overrides it.
- `ActivateArtefact` / `StopActivation` / `CreateArtefactActivation` / `CanBeActivated` — the
  use-it-up-to-spawn-an-anomaly path.
- `PhDataUpdate` — the physics-step participation, used only during an activation. Its
  sibling pre-solve hook is deliberately empty.
- `FollowByPath` / `CanBeInvisible` / `SwitchVisibility` — the detector-support surface,
  reachable from script and from a detector.
- `GetAfRank` — which detector grade can see this artefact.
- The five restoration accessors and mutators, and `AdditionalInventoryWeight`.
- `StartLights` / `StopLights` / `UpdateLights` / `SwitchAfParticles` — the presentation
  switches.
- `Action` / `OnStateSwitch` / `OnAnimationEnd` / `PlayAnimIdle` / `Hide` / `Show` /
  `IsHidden` — the held-item state machine, with one state added to the base set:
  *activating*.
- `UpdateXForm` — place a carried artefact in the owner's hands from the two weapon bones.
- `MoveTo` / `ForceTransform` — teleport, through the physics body.
- `Interpolate` — explicitly none: keep only the newest received state.
- `CanTake` — refuses while an activation is running.
- `renderable_ShadowGenerate` / `renderable_ShadowReceive` — an artefact receives shadows but
  never casts one. That is a fixed cost decision, not a per-artefact option.
- `cast_artefact` — the capability query by which the rest of the game recognizes an
  artefact.
- `o_switch_2_fast` / `o_switch_2_slow` — the update-rate switch.
- `SArtefactDetectorsSupport` — the hidden-and-wandering behaviour: a patrol path, a
  destination, a drift force, a visibility timer and a reveal sound.

## Notes

**The fast/slow switch once activated and deactivated the unconditional per-frame update
path**; both calls are commented out, so the artefact now stays on that path permanently and
the flag only decides which of the two paths does the work. A rebuild should restore the
registration or drop the flag; the present arrangement pays for a per-frame callback that
usually does nothing.
