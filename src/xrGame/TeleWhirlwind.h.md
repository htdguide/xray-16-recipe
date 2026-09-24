# src/xrGame/TeleWhirlwind.h

> Declares the whirlwind grip and the per-object entry it holds, implemented in [`TeleWhirlwind.cpp`](TeleWhirlwind.cpp.md).

**Needs** — [`ai/monsters/telekinesis.h`](ai/monsters/telekinesis.h.md) · [`ai/monsters/telekinetic_object.h`](ai/monsters/telekinetic_object.h.md) · [`xrPhysics/PHImpact.h`](../xrPhysics/PHImpact.h.md)
**Used by** — [`Mincer.cpp`](Mincer.cpp.md) · [`Mincer.h`](Mincer.h.md) · [`TeleWhirlwind.cpp`](TeleWhirlwind.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the whirlwind's two halves — the grip that owns a centre, a hold radius, a throw
power and a queue of pending debris impulses, and the per-captured-object entry that runs
the force field. Substance is in [`TeleWhirlwind.cpp`](TeleWhirlwind.cpp.md).

Exported units:

- `CTeleWhirlwind` — the grip, a specialization of the shared telekinesis mechanism.
  - `SetCenter` / `Center` — the point everything is pulled toward, written every frame by
    the anomaly that owns the grip.
  - `SetOwnerObject` / `OwnerObject` — the anomaly; destruction and particles are
    attributed to it.
  - `keep_radius`, `set_throw_power` — the two tuning values.
  - `set_destroing_particles` / `destroing_particles` — the effect played when a held
    object breaks.
  - `add_impact`, `reserve_impact`, `draw_out_impact`, `clear_impacts` — the pending debris
    impulse queue.
  - `activate` — capture an object.
  - `clear`, `clear_notrelevant` — drop all entries, or only stale ones.
  - `alloc_tele_object` — supplies the entry type, which is how the shared mechanism is
    specialized.
  - `play_destroy` — a hook with no behaviour.
- `CTeleWhirlwindObject` — one captured object's entry.
  - `init` — capture, refusing a second grip on the same object.
  - `raise`, `raise_update` — the per-step force field and an empty timeout hook.
  - `keep` — hold and spin at the eye.
  - `release` — throw or destroy.
  - `destroy_object` — break the object and queue its debris impulses.
  - `can_activate` — anything.
  - `fire` (two forms) — deliberately empty; a whirlwind has no target.
  - `switch_state`, `set_throw_power`.
