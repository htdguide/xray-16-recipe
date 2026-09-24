# src/xrGame/ai/monsters/psy_aura.cpp

> Refreshes the field's membership from the creature's current position, but only while the ability's charge allows it.

**Needs** — [`psy_aura.h`](psy_aura.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a scheduled position-and-radius query

## Purpose

Twelve lines that make the composition described in [`psy_aura.h`](psy_aura.h.md) actually
follow its creature around. Like the header, it is instantiated nowhere.

## `schedule_update`

**Contract** — called on the creature's scheduled (rate-degraded, not per-frame) update.
Advances the energy budget first, then — only if the budget says the ability is currently
active — re-centres the touch query on the creature's present position with the configured
radius and runs the subclass's effect over whatever the query admitted. No return value; the
effect is whatever the filling does.

```text
FUNCTION schedule_update()
  energy.schedule_update()          # drains while active, recovers while not; may flip active
  IF energy.is_active()
    touch.refresh(owner.position, radius)   # arrivals and departures are reported by the sense
    process_objects_in_aura()
```

**Invariants** — the budget is advanced unconditionally and the field is refreshed
conditionally, so a field that runs out of charge stops *updating* but keeps its last
membership set. Whether that is intentional is not recoverable; a rebuild should clear the set
when the field deactivates, since a stale set is otherwise visible to the next activation.

**Notes** — running on the scheduled update rather than per frame is the load-bearing choice:
the membership query is a spatial lookup, and the aura therefore costs less on a distant
creature exactly as the scheduler intends. The radius default of one world unit is never
overridden anywhere, which is further evidence the type was left unfinished.
