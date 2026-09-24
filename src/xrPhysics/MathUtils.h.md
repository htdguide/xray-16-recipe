# src/xrPhysics/MathUtils.h

> The small vector, angle and ballistics helpers the physics module needs and the
> shared math layer does not have.

**Needs** — [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — [`CarDoors.cpp`](../xrGame/CarDoors.cpp.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`Missile.cpp`](../xrGame/Missile.cpp.md) · [`WeaponMagazinedWGrenade.cpp`](../xrGame/WeaponMagazinedWGrenade.cpp.md) · [`WeaponRG6.cpp`](../xrGame/WeaponRG6.cpp.md) · [`ik_foot_collider.cpp`](../xrGame/ik_foot_collider.cpp.md) · [`ik_object_shift.cpp`](../xrGame/ik_object_shift.cpp.md) · [`imotion_position.cpp`](../xrGame/imotion_position.cpp.md) · [`interactive_animation.cpp`](../xrGame/interactive_animation.cpp.md) · [`CalculateTriangle.h`](CalculateTriangle.h.md) · [`ElevatorState.cpp`](ElevatorState.cpp.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`MathUtils.cpp`](MathUtils.cpp.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · _and 15 more_
**Tier floor** — T2: arithmetic on three-component vectors; the only thing anchoring it
lower is that half the operations are phrased against a *raw run of three floats* rather
than a vector type, so that the same code serves both the engine's vectors and the dynamics
library's.

## Purpose

This is the physics module's own math annex. It exists because the engine's math layer
speaks in vector types while the dynamics library speaks in bare float triples, and every
contact, body position and joint axis crosses that line. Rather than convert at each
crossing, the module works on the raw layout directly.

The file's substance is in three groups: the **layout bridge**, the **ground-plane
operators**, and the **ballistics solver**. Everything else is one-line convenience.

## the layout bridge

**Contract** — a vector and a run of three consecutive reals are the same bytes, and this
file says so once instead of at every call site. Nothing is copied or converted.

**Notes** — this is the sharpest instance in the chapter of the
[explicit-layout requirement](../../SYSTEM-REQUIREMENTS.md#2-tier): the dynamics library
writes body positions into memory the engine then reads as vectors, and a rebuild must
either reproduce that layout or accept a copy on every crossing. The copy is affordable —
this is not an inner loop — so a rebuild in a language that will not alias should convert
and not contort itself.

## ground-plane operators

**Contract** — dot product, magnitude and normalized dot restricted to the horizontal plane
(the two axes that are not the gravity axis).

**Notes** — the *why* is the character controller. Almost every decision it makes — am I
facing the ladder, am I walking or sliding, how fast am I moving for the animation blend —
is about horizontal motion only, with the vertical component deliberately discarded because
it is dominated by gravity and by step-climbing jitter. A rebuild that computes these in
three dimensions and then projects will get subtly different answers when the character is
on a slope.

The gravity axis is **hard-coded as the second component** throughout this file and the rest
of the chapter. There is no up-vector parameter anywhere in the physics module.

## `restrict_vector_in_dir`

**Contract** — removes from a vector any component pointing along a given direction, leaving
the component pointing against it untouched.

```text
FUNCTION restrict_in_dir(v, dir)
  projection = dot(dir, v)
  IF projection > 0
    v = v - dir * projection      # only motion INTO the constraint is removed
  RETURN v
```

**Notes** — this is a one-sided constraint, and the asymmetry is the point: a character
pressed against a wall may still move away from it. Used wherever a desired motion has to be
clipped against a surface the character is touching.

## ballistics

**Contract** — given a displacement to cover, a throw speed and gravity, find the launch
directions that land on the target.

```text
FUNCTION throw_directions(transference, speed, gravity) -> list<direction>
  # transference: target minus origin, in world space
  horizontal_sq = transference.x^2 + transference.z^2
  speed_sq      = speed^2
  # discriminant of the standard projectile-range quadratic in tan(angle)
  d = 1 - gravity / speed_sq^2 * (2 * transference.y * speed_sq + gravity * horizontal_sq)
  IF d < 0            RETURN []          # target is out of range at this speed
  horizontal = sqrt(horizontal_sq)
  scale = speed_sq / (gravity * horizontal)
  IF d == 0           RETURN [ dir_with_tangent(scale) ]        # exactly at max range
  RETURN [ dir_with_tangent(scale * (1 - sqrt(d))),             # the flat shot
           dir_with_tangent(scale * (1 + sqrt(d))) ]            # the lobbed shot
```

Two solutions are returned in a fixed order — **flat first, lobbed second** — and callers
rely on that order to prefer the flat trajectory when both are clear. Alongside it, two
inverse helpers: converting a displacement and a flight time into the launch velocity that
achieves it, and reporting the flight time of the minimum-energy throw.

**Notes** — this is how creatures and the grenade-throwing logic aim. It is exact, not
iterative, and it deliberately ignores drag — the simulation has none.

## `SInertVal`

**Contract** — a scalar with a first-order low-pass filter: each new sample is blended with
the held value by a fixed ratio set at construction and never changed. The ratio must lie
strictly between zero and one.

**Notes** — the smoothing constant is *per instance and immutable* because every user of it
tunes a different quantity (camera bob, reported speed) and a shared constant would couple
them. The filter is frame-rate dependent by construction — it blends per *update*, not per
second — which is acceptable only because every caller updates on the fixed physics step.

## rotation-matrix validation

**Contract** — a determinant computation plus a debug-only check that a matrix being handed
to the solver is still a rotation. The check warns at a 15% deviation and fails at 80%.

**Notes** — those two thresholds are the file's clearest recovered decision. A rotation
matrix drifting by 15% is a bug worth logging but not worth stopping for — it happens when a
bone's transform has been scaled by animation. At 80% the body will explode, so the run
stops. A rebuild wanting one threshold should take the warning one; the fatal one only
catches what the solver would catch a step later anyway.

## Notes

Three families of comparison macros (largest-of-three, smallest-of-three, sort-three) exist
to pick and order components without branching into function calls. They are pure C++
plumbing; a rebuild sorts three values however its language prefers.

`phInfinity` is the module's saturation value for "no limit" — used for unbounded joint
stops and for initializing distance searches.
