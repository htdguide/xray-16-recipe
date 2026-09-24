# src/xrGame/actor_mp_state_inline.h

> Initializes a networked player state holder to a state that is zero everywhere a zero is meaningful and to an identity rotation where it is not.

**Needs** — [`actor_mp_state.h`](actor_mp_state.h.md)
**Used by** — [`actor_mp_state.h`](actor_mp_state.h.md)
**Tier floor** — T2: one initialization

## Purpose

Two trivial members, one of which encodes a real decision.

## State

Adds nothing.

## Construction

**Contract** — zeroes the entire held state, then sets the physics orientation's third
imaginary component to one.

**Invariants** — an all-zero quaternion is not a rotation, and using one produces a
degenerate transform. The holder is constructed before any state has been received, and
anything that reads it in that window — a player object spawned but not yet updated —
must get a usable orientation. Setting one component to one gives a valid unit
quaternion, which happens to be a half-turn about that axis rather than the identity; the
distinction does not matter because the value is overwritten by the first update, and only
its *validity* is being guaranteed.

**Notes** — zeroing the record wholesale rather than field by field is the idiom for a
record that is also a wire image; the point is that no padding byte is ever uninitialized,
since padding would otherwise be sent. A rebuild whose serializer writes fields
individually does not need this.

## `state`

**Contract** — reads the held state.
