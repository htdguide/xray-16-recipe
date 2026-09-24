# src/xrGame/zone_effector.cpp

> Fades a full-screen post-process over the player as they walk into an anomaly, in proportion to how deep in they are and how much their suit protects them.

**Needs** — [`zone_effector.h`](zone_effector.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`PostprocessAnimator.h`](PostprocessAnimator.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md)
**Used by** — [`zone_effector.h`](zone_effector.h.md)
**Tier floor** — T2: per-frame scalar work plus a registration with the camera's effect stack

## Purpose

A zone's danger has to be legible before it hurts. This attaches a named post-process
animation to the player's camera while they are inside a zone's outer radius, and drives
its strength from the distance to the centre, so the screen distorts more the further in
they walk.

One of these belongs to each zone that wants the effect. The zone owns it, calls `Update`
with the distance every frame it is close enough to matter, and the effector handles
attaching, detaching and the strength curve itself.

## State

```text
RECORD ZoneEffector
  effect_name : text                 # the post-process animation file to play
  min_percent : real                 # fraction of the zone radius at which strength is 1
  max_percent : real                 # fraction of the zone radius at which strength is 0
  factor      : real                 # current strength, in [0.01, 1]; read by the camera
  effector    : optional<ref post-process effector>   # present iff attached
  actor       : optional<ref player>  # the player the effector is attached to
```

**Invariants** — `min_percent <= max_percent`, checked at load. The effector reference and
the player reference are attached and detached together: both present or both absent.
Strength is clamped to a *positive* floor, never to zero — see `Update`.

## `Load`

**Contract** — reads the effect's file name and the two radius fractions from a
configuration section. Accepts two alternative spellings of the name key, preferring the
first; a section naming neither is a hard failure. The two fractions are mandatory.

**Notes** — the two key spellings are the two games' data speaking: the shipped
configurations of the different titles name the same thing differently, and supporting both
is what lets one build load all three. A rebuild must accept both names.

## `Update`

**Contract** — called with the player's distance to the zone centre, the zone's radius, and
the damage type the zone deals. Attaches or detaches the effector as the player crosses the
outer radius, and recomputes the strength. Does nothing meaningful when the camera is not on
the player.

```text
FUNCTION update(distance, radius, hit_type)
  min_r := radius * min_percent
  max_r := radius * max_percent
  on_player := the level's current viewpoint entity is the player

  IF attached
    IF distance > max_r OR NOT on_player OR the player is dead
      stop()
  ELSE
    IF distance < max_r AND on_player
      activate()

  protection := 0
  IF actor EXISTS AND actor wears an outfit
    protection := outfit.protection_against(hit_type)

  IF attached
    factor := (max_r - distance) / (max_r - min_r) - protection
    factor := clamp(factor, 0.01, 1)
```

**Invariants** — after the call, the effector is attached exactly when the player is the
viewpoint, alive, and inside the outer radius.

**Notes** — **Protection is subtracted from the strength, not multiplied into it.** A suit
that protects at half strength does not halve the distortion; it shifts the whole curve down
by half, so the effect does not appear at all until the player is halfway from the outer to
the inner radius. That is why a good suit makes an anomaly feel *invisible* until you are
nearly on top of it, rather than merely dimmer throughout — and a rebuild that multiplies
will change how readable anomalies are at every protection level.

**The floor is 0.01, not 0.** A strength of exactly zero would let the camera's effect stack
consider the effector finished and drop it; keeping it barely positive keeps it alive so it
can rise again without a re-attach. The effect at that strength is imperceptible. A rebuild
whose effect stack does not cull on zero can use zero.

The strength is delivered to the camera as a *callback into this object* rather than as a
value pushed each frame. That matters for lifetime: the camera holds a reference to this
object for as long as the effector is attached, so the effector must be detached before the
zone is destroyed. That is what the destructor's stop is for.

Detach also happens when the player dies, which is how a death inside an anomaly does not
leave the death camera distorted.

## `Activate`

**Contract** — attach. Resolves the current viewpoint entity as the player and does nothing
if it is not one. Creates a cyclic post-process effector, loads the named animation into it,
wires the strength callback to this object, and pushes it onto the player's camera effect
stack.

**Notes** — the effector is registered under an identifier derived from this object's own
memory address, truncated to the type's width. That is the file's one genuinely ugly line:
it is a "unique enough" identity for a slot in a keyed stack, and it is neither stable
across runs nor guaranteed unique after the truncation. A rebuild should allocate effector
identifiers from a counter. Nothing observable depends on the values themselves — only on
`Activate` and `Stop` agreeing on one.

Cyclic means the animation loops rather than running once and ending; the effect's duration
is decided by the player's presence, not by the animation's length.

## `Stop`

**Contract** — detach. Removes the effector from the player's camera stack by the same
derived identifier, and clears both references. Safe to call when not attached.

**Notes** — the removal is by identifier, and the identifier is recomputed from the object
address at removal time, so it matches only because the object has not moved. A rebuild that
stores the allocated identifier avoids the whole class of problem.

## `GetFactor`

**Contract** — the current strength. This is the callback the camera's effect stack invokes
each frame; it must be cheap and must not touch the world.
