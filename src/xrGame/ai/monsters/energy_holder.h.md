# src/xrGame/ai/monsters/energy_holder.h

> Declares the rechargeable ability budget, implemented in
> [`energy_holder.cpp`](energy_holder.cpp.md).

**Needs** — [`energy_holder.cpp`](energy_holder.cpp.md)
**Used by** — [`energy_holder.cpp`](energy_holder.cpp.md) · [`poltergeist.cpp`](poltergeist/poltergeist.cpp.md) · [`poltergeist.h`](poltergeist/poltergeist.h.md) · [`psy_aura.h`](psy_aura.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the surface of the stamina mixin that creature *fields* — always-on abilities with no
target and no animation — draw on. A creature inherits it and overrides the two notification
hooks; see [`energy_holder.cpp`](energy_holder.cpp.md) for the rules and
[`psy_aura.h`](psy_aura.h.md) for the canonical override.

## `CEnergyHolder`

The lifecycle:

- **reinit** — budget full, ability on, clock anchored now
- **reload(section, prefix, suffix)** — read the five tuning numbers from a configuration section
- **schedule_update** — advance the budget and, if armed, act on the thresholds

The switch, and the hooks a creature overrides:

- **activate** / **deactivate** — flip the ability, firing the hook only on a real transition
- **on_activate** / **on_deactivate** — do nothing by default; this is where the subclass makes
  the ability real
- **is_active** — is the ability on

The predicates and the arming flags:

- **can_activate** — the budget is above the activation threshold
- **should_deactivate** — the budget is below the critical threshold
- **set_auto_activate** / **set_auto_deactivate** — arm each direction of automatic switching
  independently; both start disarmed

Suspension and mode:

- **enable** / **disable** — resume or freeze the simulation; resuming re-anchors the clock
- **set_aggressive** — select the faster of the two refill rates
- **get_value** — read the budget; used by the debug overlay
