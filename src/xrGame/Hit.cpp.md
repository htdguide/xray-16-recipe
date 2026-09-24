# src/xrGame/Hit.cpp

> One damage event, as it travels from the thing that caused it to the thing that receives it — and, unchanged, across the network.

**Needs** — [`Hit.h`](Hit.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`Level.h`](Level.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md)
**Used by** — reached through its declarations in [`Hit.h`](Hit.h.md); callers name that, not this file.
**Tier floor** — T1: the wire encoding is a frozen bit-packed layout with quantized directions and conditional fields

## Purpose

Every source of damage in the game — a bullet, an explosion, a claw, an anomaly, a fall —
produces one of these, and every recipient consumes one. It is the single vocabulary of
damage, and the reason it is worth its own file is that it is simultaneously three things:

- a **value** passed by reference through a dozen virtual hit handlers;
- a **network message body**, with its own header, written and read in a frozen order;
- a **self-describing validity marker** — a hit can exist in a state that means "no hit",
  and consumers assert against reading one.

The frozen wire order is the part a rebuild cannot choose freely: the same bytes must be
produced and consumed by both ends, and two fields are conditional on the values of
earlier ones.

## State

```text
RECORD Hit
  # header, written only by the full-packet form
  time          : int (32-bit)   # server time the hit was generated
  packet_type   : int (16-bit)   # which event this is: an ordinary hit, or a scored one
  dest_id       : int (16-bit)   # the entity being hit

  # body
  who_id        : int (16-bit)   # the attacker's entity identifier
  weapon_id     : int (16-bit)   # the weapon's, for statistics and for kill messages
  direction     : vec3           # quantized on the wire; see Notes
  power         : real           # the damage, before any immunity or armour is applied
  bone_id       : int (16-bit)   # which bone was struck; selects the damage multiplier
  point_in_bone : vec3           # where on that bone, in the bone's own space
  impulse       : real           # the physical push, independent of the damage
  aim_bullet    : bool           # single player only; see Notes
  hit_type      : HitType        # bullet, explosion, burn, shock, chemical, radiation, …
  armor_piercing: real           # present on the wire ONLY for bullet hits
  bullet_id, sender_id : int     # present ONLY for the scored-hit event

  # local only, never transmitted
  who           : optional<Object>   # the attacker, when it is still alive locally
  add_wound     : bool               # whether the recipient should grow a visible wound
```

Invariants:

- **Damage and impulse are separate.** A heavy slow object pushes hard and damages
  little; a small fast one the reverse. Nothing derives one from the other.
- A hit is **valid** exactly when its damage type is not the out-of-range sentinel.
  Invalidating sets every numeric field to a large negative value so that a hit read
  after invalidation is obviously wrong rather than plausibly zero.
- The attacker is carried *twice*: as an identifier, which always survives, and as a live
  reference, which does not cross the network and may be gone by the time the hit is
  handled. Consumers that need the object must check.
- `who_id` is zero when there is no attacker. Zero is also the player's identifier in
  single player, which is a latent ambiguity the code does not resolve.

## `SHit` — construction and `invalidate`

**Contract** — the full constructor takes damage, direction, attacker, bone, the point in
bone space, impulse, damage type, armour piercing and the aimed-shot flag, and defaults
the wound flag to true and every network-only field to zero. The default constructor
produces an *invalid* hit.

**Invariants** — a default-constructed hit is invalid, not empty. That distinction is what
lets a recipient hold "the last hit I took" in a field and ask whether there was one.

## `is_valide` / `damage` / `direction` / `initiator` / `bone` / `bone_space_position` / `phys_impulse` / `type`

**Contract** — the accessors, each asserting validity before answering. They exist so that
reading an invalidated hit is caught at the read rather than producing a large negative
damage that some consumer silently applies.

## `GenHeader`

**Contract** — stamp the hit with its destination, its event type and the current **server**
time, making it ready to transmit. Server time, not local time, because the recipient
compares it against its own view of the server clock.

## `Write_Packet` / `Write_Packet_Cont` / `Read_Packet` / `Read_Packet_Cont`

**Contract** — the wire encoding, in two halves. The full form writes an event header —
time, event type, destination — and then the body; the continuation form writes only the
body, for use when the header has already been framed by a containing message. Reading
mirrors exactly.

**Invariants** — the field order is frozen and two fields are **conditional on earlier
fields in the same message**, which means the reader cannot be written as a fixed layout:

- the armour-piercing value is present only when the damage type is *bullet*;
- the bullet and sender identifiers are present only when the event type is the
  *scored hit* variant, which multiplayer uses to attribute kills.

A third conditional is worse: the aimed-shot flag is written **only in single player** and
read back the same way, so the message length depends on the *game mode*, which is not in
the message. Both ends must already agree on the mode. A rebuild should make it
unconditional or move it into a flags word.

```text
FUNCTION write(packet)
  packet.begin_event()
  write time, packet_type, dest_id
  write_body(packet)

FUNCTION write_body(packet)
  write who_id, weapon_id
  write direction as a quantized unit vector       # not three floats; see Notes
  write power, bone_id, point_in_bone, impulse
  IF single player THEN write aim_bullet
  write hit_type
  IF hit_type is bullet THEN write armor_piercing
  IF packet_type is the scored-hit event THEN write bullet_id, sender_id
```

**Notes** — the direction is written through the packet's *direction* encoding, which
quantizes a unit vector into far fewer bits than three floats. That is a deliberate
bandwidth decision and it means the direction that arrives is not bit-identical to the one
sent; nothing downstream may depend on exact equality.

The damage type is written as sixteen bits with an explicit mask, and so is the event
type. The masks are redundant with the field widths and exist because the in-memory types
are wider; a rebuild simply declares them at the wire width.

The read form takes its packet **by value**, so reading does not advance the caller's
packet. Every caller relies on that, which makes it a contract rather than an oversight —
but it is an expensive way to express "peek", and a rebuild should read from an explicit
cursor.

## `_dump`

**Contract** — write every field to the log. Debug-only diagnostics; a rebuild may omit
it.

## Notes

The wound flag is set at construction and never transmitted. A hit that crosses the
network therefore always arrives with the flag *false*, because the receiving side
reconstructs the hit from the wire and the wire does not carry it. Whether that is
intended — remote clients not growing wounds — or an oversight is not recoverable from
the source.
