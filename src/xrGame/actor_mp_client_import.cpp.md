# src/xrGame/actor_mp_client_import.cpp

> Applies a received player update: authoritative values are set, the view is snapped to the sender's aim, and the rest is queued for interpolation rather than applied at once.

**Needs** — [`actor_mp_client.h`](actor_mp_client.h.md) · [`actor_mp_state.h`](actor_mp_state.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level.h`](Level.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`xrPhysics/phvalide.h`](../xrPhysics/phvalide.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: consumes the wire record directly

## Purpose

The receive half. The central design decision is the split: **values the server is
authoritative over are applied immediately, and values that describe motion are queued**.
Motion queued and interpolated looks smooth at the cost of being a fraction of a second
behind; health applied late means shooting a player who is already dead.

## State

None owned. Feeds two bounded queues on the player: logical updates and physics updates.

Invariant: each queue holds at most five entries. Five updates at the server's send rate
is a few hundred milliseconds of buffer — enough to interpolate across a lost packet,
short enough that the visible lag stays inside what a shooter tolerates. The number is
the interpolation budget and is the same on both queues.

## `net_Import`

**Contract** — reads the wire record, validates the position, then applies it in two
parts.

```text
FUNCTION net_import(packet)
  state = read_wire_record(packet)
  REQUIRE state.position is a valid coordinate

  IF running as a client THEN
    apply_health(state.health)          # see the invincibility rule below

  IF this player is currently a ragdoll THEN
    RETURN                              # physics owns the body; nothing below applies

  IF running as a client THEN
    radiation = state.radiation * 100
    IF inventory.active_slot != state.inventory_active_slot THEN
      inventory.active_slot = state.inventory_active_slot

  logical.timestamp = state.time
  logical.position  = state.position
  logical.movement_flags = state.body_state_flags
  logical.body_yaw  = state.model_yaw
  logical.torso     = state.camera_yaw, camera_pitch, camera_roll
  IF logical.torso.roll > half turn THEN logical.torso.roll = logical.torso.roll - full turn
  logical.acceleration = state.logic_acceleration

  IF this is a demo playback, or we are the server, or this player is remote THEN
    player.unaffected_torso = logical.torso
    player.camera.yaw   = -logical.torso.yaw     # the camera's yaw runs opposite to the torso's
    player.camera.pitch =  logical.torso.pitch

  queue_logical(logical)
  queue_physics(state.physics_*)
```

**Invariants**

- **Health is only ever raised unconditionally; lowering it respects an invincibility
  flag.** A server-sent heal always applies, but a server-sent wound is ignored for a
  player the match has marked invincible — a spawn protection or a game-mode rule. The
  asymmetry is deliberate: the invincibility flag must not block healing.
- A ragdolling player is skipped entirely below the health step. The rigid-body simulation
  owns the body in that state and applying a network position would fight it.
- The roll angle is un-normalized from the wire's single-turn range back into a signed
  range around zero, because the camera expects a signed lean and the wire carries an
  unsigned turn. Yaw and pitch are not adjusted; only roll crosses zero in normal play.
- The camera's yaw is set to the *negation* of the torso's. The two use opposite
  conventions for which way is positive, and this is the only place that disagreement is
  reconciled.
- The view snap applies when watching someone else, when acting as the server, or during
  demo playback — never for the locally controlled player, whose aim is its own and must
  not be overwritten by a round trip.

## `process_packet` (the logical queue)

**Contract** — appends a logical update to the interpolation queue, with ordering rules:
an update older than the newest queued one is discarded; one with the same timestamp
*replaces* it; otherwise it is appended and the queue trimmed from the front to its bound.
A locally controlled player on a client queues nothing — its own motion is predicted, not
received. A living player is made visible and enabled on the first update that reaches it,
except while the first-person heads-up view is showing it.

**Invariants** — replacing on an equal timestamp rather than appending is what keeps the
queue's timestamps strictly increasing, which the interpolator relies on to divide by a
non-zero interval.

## `postprocess_packet` (the physics queue)

**Contract** — timestamps the physics update from the newest logical update if there is
one and from the server clock otherwise, seeds its previous-position and
previous-orientation fields from the current ones, and applies the same ordering rules to
the physics queue. A locally controlled player on a client, and any dead player, queues
nothing. Queueing anything switches the player into interpolated motion and registers it
with the level's prediction-correction pass, resetting that pass's progress.

**Invariants**

- Borrowing the logical update's timestamp keeps the two queues aligned on one clock, so
  the interpolator can sample both at the same instant. Falling back to the server clock
  covers the first update, where there is no logical entry yet.
- Seeding the previous pose from the current one means a newly queued physics state
  interpolates from *itself* — no motion — rather than from whatever stale pose the
  record happened to carry. The first frame after a queue is therefore still, not a jump.

**Notes** — the two queues are maintained by near-identical code with the same bound and
the same ordering rules, differing only in what they hold. A rebuild should write one
bounded ordered queue and instantiate it twice.
