# src/xrGame/actor_mp_client_export.cpp

> Gathers a networked player's state from the live simulation into the wire record, and decides whether it is worth sending at all.

**Needs** — [`actor_mp_client.h`](actor_mp_client.h.md) · [`actor_mp_state.h`](actor_mp_state.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`Inventory.h`](Inventory.h.md) · [`xrPhysics/phvalide.h`](../xrPhysics/phvalide.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: assembles the record whose byte layout is the protocol

## Purpose

The send half of a networked player. It answers two questions per update: *what is this
player's state right now*, and *should we send it*.

The file's header comment is an accounting of a deliberate size-reduction pass on the
player update — 138 bytes reduced to 27 — itemized by what each change saved. That
accounting is the most valuable thing in the file for a rebuilder, because it says which
fields a shooter can afford to drop or crush and roughly what each is worth.

## State

None owned; fills the wire-state holder declared in
[`actor_mp_state.h`](actor_mp_state.h.md).

## `fill_state`

**Contract** — reads the player's current state out of the live simulation into the wire
record. Takes the rigid-body half from the character's physics synchronization item
(orientation, angular and linear velocity, force, torque, position and whether the body is
awake); the logical half from the player itself (position, the intent-driven acceleration,
the body's facing, the torso's aim, the active inventory slot, the movement flag word,
health, radiation); and the timestamp from the *server's* clock rather than the local one.

**Invariants**

- Every angle is normalized into a single turn before being stored, because the wire form
  quantizes over exactly one turn and an un-normalized angle would alias.
- The torso aim is read from the *unaffected* torso rotation — the aim before weapon sway,
  recoil and breathing are applied. Sending the affected one would transmit a jitter the
  receiver reproduces on top of its own, doubling it.
- Health below a very small epsilon is snapped to exactly zero. The wire form's packing
  deliberately never rounds a non-zero value down to zero (see
  [`actor_mp_state.cpp`](actor_mp_state.cpp.md)), so a player with a sliver of health
  would transmit as alive forever; the snap is the sending side's half of that contract.
- Radiation is scaled to the unit interval by dividing by one hundred, because the wire
  form carries it as a normalized scalar. One hundred is the radiation scale's maximum and
  is the constant the two sides agree on.
- The movement flag word is masked to its low sixteen bits, matching the fifteen-bit wire
  field plus its clearance. Flags above that are local-only.
- The timestamp is the server's clock. Ordering on the receiving side compares timestamps
  across senders, so a local clock would make them incomparable.

```text
FUNCTION fill_state() -> state
  body = physics_sync_item(0).state        # exactly one sync item: a player is one body
  state.physics_* = body.orientation, angular_velocity, linear_velocity,
                    force, torque, position
  state.physics_state_enabled = body.awake
  state.position = player.position
  state.logic_acceleration = player.last_sent_acceleration
  state.model_yaw   = normalize_angle(player.body_yaw)
  state.camera_*    = normalize_angle(player.unaffected_torso.yaw/pitch/roll)
  state.time = server_clock
  state.inventory_active_slot = inventory.active_slot
  state.body_state_flags = player.movement_flags AND 0xffff
  state.health = player.health ; IF state.health < epsilon THEN state.health = 0
  state.radiation = player.radiation / 100
```

## `net_Relevant`

**Contract** — decides whether this player is sent this update. A player whose character
physics has been removed is never sent. Otherwise the current state is gathered and
offered to the holder, which stores it and (in the shipped configuration) always answers
yes.

**Notes** — a commented-out rule would also have stopped sending a dead player entirely,
which the size accounting credits with two bytes of average saving. It is disabled, so
dead players are still synchronized.

## `net_Export`

**Contract** — writes the previously gathered state to the packet. Asserts the position is
a finite, in-range coordinate before sending; a corrupted position propagated to every
client is far worse than a dropped update, so the check is on the sending side and is
fatal.

**Invariants** — `net_Relevant` must have run first, because it is what gathers the state
that this writes. The two are one operation split across the engine's relevance-then-send
pass, and a rebuild that reorders them sends stale data.
