# src/xrPhysics/IColisiondamageInfo.h

> The description of an impact that physics hands the game so the game can turn it
> into a hit.

**Needs** — [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)
**Used by** — [`PHMovementControl.cpp`](../xrGame/PHMovementControl.cpp.md) · [`movement_manager_physic.cpp`](../xrGame/movement_manager_physic.cpp.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md)
**Tier floor** — T3: a read-only record behind an interface.

## Purpose

Physics knows how hard two things hit each other; only the game knows what that costs in
health. This interface is the handover. It is deliberately read-only from the game's side
and is filled in by whichever physics object recorded the impact — a character
([`PHSimpleCharacter.h`](PHSimpleCharacter.h.md)) or a shell element.

The *latched* design is the load-bearing part: a collision fills the record and marks it
un-initiated; the game reads it at its own pace, once, and the read clears the mark. That
decouples the physics step rate from the game's update rate without losing impacts or
double-counting them.

## `ICollisionDamageInfo`

**Contract** — reports the impact:

- **Contact velocity** — the closing speed at the contact, already scaled by the object's
  collision-damage factor. Zero means "no impact worth reporting". This is the single
  number the damage formula is built on.
- **Hit direction** and **hit position** — where and which way, in world space, for
  directional damage, for the hit's visual effect, and for knockback.
- **Damage initiator** — the id of the object that caused the impact, and the object itself.
  Needed because being crushed by a crate someone threw is that someone's kill. When the
  impact was against static geometry there is no initiator and the id is the no-object value.
- **Hit type** — which damage channel this impact belongs to (the game's own hit taxonomy),
  settable by the physics side because different physical causes map to different channels.
- **Hit callback** — the per-object hook that converts the kinematics into damage.

**Invariants** — the *initiated* flag is the latch:

```text
FUNCTION consume(info) -> optional<Impact>
  IF NOT info.get_and_reset_initiated()      # test and clear in one operation
    RETURN none                              # nothing new since the last read
  RETURN Impact( info.contact_velocity, info.hit_dir, info.hit_pos, info.initiator )
```

`Reinit` clears the record back to "no impact" and is called when the object is respawned or
its physics rebuilt, so that a stale impact from a previous life cannot fire.

**Notes** — the destructor is protected: holders of this interface are never its owners. The
record always lives inside the physics object that recorded it. In a rebuild this is simply
a borrowed reference with the physics object's lifetime.
