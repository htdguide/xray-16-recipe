# src/xrGame/ai/monsters/controller/controller_psy_hit_effector.h

> Dead file: an abandoned post-process and camera effector for the psi attack, entirely commented out.

**Needs** — [`../../../pp_effector_custom.h`](../../../pp_effector_custom.h.md) · [`xrEngine/Effector.h`](../../../../xrEngine/Effector.h.md)
**Used by** — [`controller_psy_hit_effector.cpp`](controller_psy_hit_effector.cpp.md)
**Tier floor** — T4: nothing is declared

## Purpose

The file declares nothing. Two class declarations — a post-process effector driven by the
distance between the creature and the player, and a camera effector interpolating an angular
offset over a fixed duration — are present in full as comments and neither is compiled.

It is named here for completeness of the mirror. The psi attack's actual camera effector is
declared elsewhere and is reached by the identifier named in
[`controller_psy_hit.cpp`](controller_psy_hit.cpp.md); the forward declarations at the top of
that file's header point at these two dead classes and are themselves unused.

## State

Stateless.

## What the abandoned design was

Worth one paragraph, because it is the only record of it. The post-process effector would
have had a factor rising as the player approached the creature, computed from the distance
against an authored radius and a pair of minimum and maximum fractions of it, with authored
attack and release portions — that is, a *proximity aura* the player would feel before the
attack began, rather than an effect that starts with the attack. The camera effector would
have applied an interpolated angular offset rather than moving the camera's position, which
is a shake rather than a drag.

Neither survives. A rebuild should reproduce the shipped behaviour described in
[`controller_psy_hit.cpp`](controller_psy_hit.cpp.md) and may drop this file entirely.
