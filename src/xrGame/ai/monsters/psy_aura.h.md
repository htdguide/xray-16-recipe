# src/xrGame/ai/monsters/psy_aura.h

> A creature-carried field that keeps a live set of whatever is standing inside it — declared, complete, and instantiated by nothing in the shipped engine.

**Needs** — [`energy_holder.h`](energy_holder.h.md) · [`xrEngine/Feel_Touch.h`](../../../xrEngine/Feel_Touch.h.md)
**Used by** — [`psy_aura.cpp`](psy_aura.cpp.md)
**Tier floor** — T3: a radius, a membership set and a scheduled refresh

## Purpose

A psi aura is the composition of two things the engine already has: the *touch* sense, which
maintains the set of objects within a radius of a point and reports arrivals and departures,
and the energy-holder mixin, which gives an ability a charge that drains while active and
recovers while not, so the ability cannot run continuously. Attaching the first to a creature's
position and gating it on the second gives "a field around a monster that affects whatever
stands in it for as long as the monster can sustain it".

The type is complete and the base implementation is in [`psy_aura.cpp`](psy_aura.cpp.md), but
**nothing in the engine derives from it or constructs one.** The psi dog's aura, which is the
behaviour this type was evidently written for, is a separate and unrelated type. Treat this
page as a description of an unused design, not of running behaviour.

## State

```text
RECORD PsyAura
  owner   : reference to the creature the field is centred on
  radius  : real       # default 1, and no shipped code ever sets it
  # plus the touch sense's membership set and the energy holder's charge
```

## The demands on a filling

**`process_objects_in_aura`** — the whole point of the type, and the base does nothing. A
filling reads the membership set the touch sense maintains and applies whatever the aura does.
This is the one method a rebuild must supply.

**`feel_touch_contact`** — the admission test: which objects the field considers at all. The
base *refuses everything*, so an aura that does not override it has an empty membership set
and `process_objects_in_aura` sees nothing. A filling must override both or neither.

**`set_radius` / `get_radius`** — the field's extent, in world units.

**`init_external`** — binds the aura to the creature whose position it follows. Separate from
construction because the aura is a member of the creature and cannot take the creature's
address until the creature exists.

## Notes

The two-mixin shape — a sense interface plus an energy budget — is the pattern the chapter uses
for every sustained special ability, and it is worth keeping even though this particular
instance is dead: the budget is what makes an ability episodic without any state in the brain
tracking it.
