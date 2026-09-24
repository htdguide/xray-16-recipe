# src/xrGame/ai/monsters/telekinesis.cpp

> Owns a set of levitated objects and drives them on two different clocks: a coarse phase machine on the behaviour tick, and a force application on every physics step.

**Needs** — [`telekinesis.h`](telekinesis.h.md) · [`telekinetic_object.h`](telekinetic_object.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`telekinesis.h`](telekinesis.h.md)
**Tier floor** — T2: owned records, a physics-step callback, and lifetime rules that must be exact

## Purpose

The load-bearing decision in this file is the **two-clock split**.

*When* an object stops rising and starts hovering, and *when* it has hovered long enough to
be dropped, are decided on the creature's scheduled update — a coarse, distance-degraded
tick. *How much force* is applied to keep it there is decided on every physics step, which
runs at the physics rate regardless of how far away the creature is. Separating them means
a poltergeist across the level still throws things convincingly while paying almost nothing
in AI budget, and it means a levitated object never stutters because the creature's update
was deferred.

The second decision is that this controller injects **accelerations, not impulses**,
between collision detection and the constraint solve. Gravity is switched off on a
levitated object and this controller supplies a substitute. Everything a rebuild does here
must land in the same place in the step, or objects will sink through floors or jitter.

## State

```text
RECORD Telekinesis
  objects : list<TelekineticObject>   # owned; each is destroyed when removed
  active  : bool                      # true while at least one object is held

  # invariant: `active` and `objects` are consistent only at the ends of the public
  # operations. The controller's physics subscription is enabled exactly when the list
  # is non-empty, and the subscription is what makes the per-step force run at all.
```

## `activate`

**Contract** — allocates a per-object record through the overridable hook, initialises it
with the strength, target height, hold duration and rotate flag, and adopts it. Returns the
record so the caller can attach sounds or read it back; returns nothing and destroys the
record when the object has no physics body. Marks the controller active and enables the
physics subscription. Note it sets `active` *before* knowing whether the object was
acceptable, so a failed activation leaves the controller marked active with an unchanged
set — harmless, because the per-step loop iterates the set.

## `schedule_update`

**Contract** — the coarse clock. Advances every held object's own phase machine, and
removes any object that has released itself. Does nothing when the controller is inactive.

```text
FUNCTION schedule_update()
  IF NOT active THEN RETURN
  i = 0
  WHILE i < objects.count
    objects[i].update_state()          # rising -> hovering -> dropped, or throw expiry
    IF objects[i].is_released()
      remove_object(i)                 # destroys the record; may deactivate the controller
    ELSE
      i = i + 1
```

**Invariants** — removal is by index into a list that the removal itself shortens. The
original advances the index unconditionally, which means the object that slides into a
removed slot is skipped for one tick. Harmless at the coarse clock, but a rebuild should
not reproduce the off-by-one by accident and then be surprised by a different one.

## `PhDataUpdate` — the per-step force

**Contract** — called once per physics step, before the solve. For each held object,
applies the rising force or the hovering force according to its phase. Objects in the
thrown phase and released objects are skipped: once thrown, the object is ballistic and the
physics library owns it.

## `PhTune` — the per-step wake-up

**Contract** — called once per physics step, and first prunes objects that have become
irrelevant: destroyed, or with their physics body gone or deactivated. Then it wakes the
physics body of every rising or hovering object, because a body held at a steady height by
balanced forces will otherwise be put to sleep by the physics library and stop responding.

**Notes** — the pruning step removes entries from the list *without destroying the records
it removes*. Those records are leaked. It is the only place in the file that forgets an
object rather than releasing it, and the asymmetry is almost certainly unintended.

## `deactivate` / `clear_deactivate` / `clear` / `clear_notrelevant`

**Contract** — four different degrees of forgetting, and they are not interchangeable:

| Operation | Physics restored? | Records destroyed? | Subscription ended? |
|---|---|---|---|
| deactivate (all) | yes — each object released, gravity restored, small downward nudge | yes | yes |
| clear-and-deactivate | no — each object is merely switched to the released phase | yes | yes |
| clear | no | no — the list is emptied, records leaked | no |
| clear-not-relevant | not applicable — the objects are already gone | no — leaked | no |

The second form exists for the case where the *owner* is being destroyed and the objects
are about to be destroyed too, so restoring their gravity is pointless. The third is an
internal helper used after an explicit destruction loop; called on its own it leaks.

## `fire_all` / `fire` / `fire_t`

**Contract** — `fire_all` throws every held object at one target with unit power and then
deactivates the whole controller. `fire` finds one object and throws it with a caller's
power factor, leaving it in the set until its throw phase expires. `fire_t` finds one object
and throws it so that it arrives at the target after a given flight time — the ballistic
form, used when the throw must lead a moving target.

## `deactivate(object)` / `remove_object`

**Contract** — `deactivate(object)` releases the named object (gravity back on, downward
nudge) and then removes it. `remove_object` destroys the record, erases the entry, and —
if that emptied the set — ends the physics subscription and marks the controller inactive.
Everything that shrinks the set must go through it, or the subscription outlives the last
object.

## `get_objects_count` / `get_objects_total_count` / `is_active_object`

**Contract** — the first counts only objects in the rising or hovering phases, which is what
an owner asks when deciding whether it has enough ammunition to throw; the second counts
everything including objects mid-flight. `is_active_object` is set membership.

## `remove_links`

**Contract** — the engine-wide "this entity is being destroyed" notification. Removes the
named object from the set without releasing it, because there is nothing left to release.

**Notes** — the whole class is a physics-step subscriber, which is the only way it can
inject accelerations at the right point in the step. A rebuild whose physics seam does not
expose a between-detection-and-solve hook must find another way to hold objects at a height
— per-step velocity assignment is the usual substitute, and it changes how levitated objects
respond to being shot.
