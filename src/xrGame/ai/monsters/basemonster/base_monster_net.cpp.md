# src/xrGame/ai/monsters/basemonster/base_monster_net.cpp

> The creature's wire form and its save relevance.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`xrAICore/Navigation/game_graph.h`](../../../../xrAICore/Navigation/game_graph.h.md) · [`CharacterPhysicsSupport.h`](../../../CharacterPhysicsSupport.h.md) · [Seam: Networking transport](../../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: a byte layout written and read field by field into a packed stream

## Purpose

What crosses the network for a creature, and what a save must contain. Both are thin — the
creature's whole brain is reconstructed from its section and its alife record, so only its
body state travels.

The file is worth reading for what it records about the protocol's history rather than for
its logic: several fields are written twice, and the redundancy is load-bearing only in that
both ends must agree on the field count.

## `save_to_packet` / `save_relevant`

**Contract** — appends the physics support's saved state to the inherited save. A creature
is save-relevant if the inherited rules say so **or** it currently has a physics shell — a
ragdolled creature must be saved even when it would otherwise be skipped, because its pose
is not reconstructible.

## `export_state`

**Contract** — writes the creature's authoritative state for a remote client. Requires the
creature to be locally owned. Sends the most recent recorded update rather than the live
state, so that what goes out matches what the local simulation last committed.

```text
FUNCTION export_state(out)
  FAIL WITH not-local IF this creature is not locally simulated
  FAIL WITH no-history IF no update has been recorded

  u = the most recent recorded update
  out.write real    health
  out.write int     u.timestamp
  out.write int8    0                  # a flags byte, always zero
  out.write vec3    u.position
  out.write real    u.model_heading
  out.write real    u.torso.yaw
  out.write real    u.torso.pitch
  out.write real    u.torso.roll
  out.write int8    team, squad, group

  v = my game-graph vertex
  out.write v ; out.write v            # written twice
  IF v is a valid game-graph vertex THEN
    d = distance from my position to that vertex's world point
    out.write d ; out.write d          # written twice
  ELSE
    out.write 0 ; out.write 0
```

**Invariants** — angles are written as full-width reals. The original's commented-out
alternative wrote them as single bytes, which is the quantisation the protocol was designed
for; it was turned off and the field widths changed with it. A rebuild defining its own
protocol should quantise, and must then version-gate.

**Notes** — the doubled game vertex and doubled distance are the file's one real puzzle. The
receiving side reads both into the same variable, so the second overwrites the first and the
value is identical anyway. The most likely reading is that the pair once carried two
different quantities — the commented-out lines beside them wrote a movement speed twice in
the same shape — and the fields were never removed because both ends would have to change
together. The distance is computed twice from the same inputs, which is pure waste.

The flags byte is written as zero and read into a variable that is never consulted.

## `import_state`

**Contract** — reads the above into a fresh update record, sets health directly, and appends
the record to the interpolation history **only if it is newer than the last one held**.
Marks the creature visible and enabled. Requires the creature to be remotely owned.

**Invariants** — out-of-order packets are dropped rather than reordered: an update whose
timestamp is not strictly newer than the newest held is discarded entirely. That is the whole
of the ordering discipline, and it is why the timestamp is the second field.

The game vertex is read into a local that is then used to decide whether to compute the
distances — it is never stored on the creature. So the navigation state the sender took the
trouble to transmit is discarded on receipt. That is either dead protocol or an unfinished
feature; nothing recoverable says which.
