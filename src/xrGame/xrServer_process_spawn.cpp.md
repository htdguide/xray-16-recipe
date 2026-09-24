# src/xrGame/xrServer_process_spawn.cpp

> Turns a spawn record into a live entity: allocates its identifier, refuses it if the game mode will not have it, attaches it to its parent, and tells everybody — twice, differently.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`game_sv_mp_script.h`](game_sv_mp_script.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: reads a frozen record into a factory-constructed object and re-emits it as two wire packets

## Purpose

**Every entity in the world comes into existence here.** The authored spawn file, the save loader,
the respawn queue, the alife simulation promoting an offline entity, and a local client's request
all converge on this one function. That convergence is the most important structural fact in the
chapter: there is exactly one construction path, and it is the one that reads a spawn record.

## State

`Stateless.`

## `Process_spawn`

**Contract** — create and register one entity from a spawn record. Returns the entity, or nothing
on refusal. Four callers with four shapes, distinguished by two optional arguments: an existing
entity to adopt instead of constructing one, and a flag to parent the new entity to the requesting
client's own body.

```text
FUNCTION process_spawn(packet, sender, parent_to_clients_body, existing) -> optional<entity>
  client := client_for(sender)          # may be absent: the server spawns as nobody

  IF existing IS none
    class_name := packet.read_string()
    entity := factory.create(class_name)        # hard failure if unregistered
    entity.read_spawn(packet)

    # three independent vetoes, all before the entity enters any table
    IF entity's game-type mask does not admit this game mode
       OR entity's configuration does not match this build
       OR the rules layer refuses it in pre-create
      destroy entity
      RETURN none
  ELSE
    entity := existing                  # an alife-controlled entity being promoted

  # the parent must already exist
  IF entity names a parent
    parent := entities[entity.parent_id]
    IF parent is absent
      REQUIRE this was not a promotion    # a promotion must never lose its parent
      destroy entity
      RETURN none

  IF client is absent
    client := best client to simulate this entity      # in practice, the host

  ... identifier allocation and table insertion; see below ...

  IF the entity is flagged "spawn as the player"
    client.owner := entity

  place_at_respawn_point(entity)
  entity.respawn_selector := "use supplied coordinates"   # so a later respawn keeps the place

  IF this was not a promotion
    rules.on_create(entity.id)
    IF entity names a parent
      rules.on_touch(parent.id, entity.id)       # unless the mode is script-driven
      append entity.id to parent.children

  ... broadcast; see below ...

  IF this was not a promotion THEN rules.on_post_create(entity.id)
  RETURN entity
```

**Invariants** — **the three vetoes all run before the entity gets an identifier or enters the
table**, so a refused entity leaves no trace. The game-type mask is a bitset on the record: a
spawn authored for single player carries a mask that a deathmatch does not match, which is how one
level file serves several modes.

An entity may not be spawned before its parent. Unlike the join replay in
[`xrServer_CL_connect.cpp`](xrServer_CL_connect.cpp.md), this does *not* recurse — it refuses.
That places the ordering burden on the caller, which for a save load means the file order, and is
the fragility described in [`xrServer_perform_sls_load.cpp`](xrServer_perform_sls_load.cpp.md).

After placement the respawn selector is rewritten to "use the supplied coordinates", so an entity
that was placed at a chosen respawn point will, if respawned later, reappear where it was placed
rather than being re-chosen.

### Identifier allocation and the phantom

Three cases, and the first is the one worth understanding.

```text
IF entity has a respawn time AND no phantom yet
  # a respawning entity needs a placeholder that outlives its own destruction
  phantom := factory.create(same class name)
  phantom.read_spawn(the SAME packet)          # re-read, from the same position
  phantom.id := allocate_id(fresh)
  phantom.phantom_id := phantom.id             # self-linked; a phantom cannot breed
  phantom.owner := none
  phantom.flags.set(PHANTOM)
  insert phantom

  entity.id := allocate_id(preferring its recorded id)
  entity.phantom_id := phantom.id
  insert entity

ELSE IF entity is itself flagged a phantom
  # the respawn queue is bringing one back: it stops being a phantom
  entity.id := allocate_id(fresh)
  entity.flags.clear(PHANTOM)
  insert entity

ELSE
  IF parent_to_clients_body
    entity.parent_id := client.owner.id
  entity.id := allocate_id(preferring its recorded id)
  insert entity
```

**Invariants** — **a phantom is a dormant copy of a respawnable entity, held in the table so that
the original can be destroyed without losing the record needed to recreate it.** It is excluded
from update broadcasts and from the join replay. The self-link — a phantom's phantom reference
pointing at itself — is what stops a phantom from spawning a phantom of its own, and the source
says so.

The allocator is asked to honour the record's own identifier when there is one, and to invent one
otherwise. That is how a saved entity keeps the identifier every other saved record refers to it
by, while a freshly spawned one gets a free slot.

The phantom re-reads **the same packet from the same position**, which works only because reading
a spawn record does not consume from a shared cursor in the way the code implies — both reads
start from where the record's payload begins. This is the file's most delicate line and a rebuild
should copy the record rather than re-parse it.

### The broadcast: two different packets

**Contract** — the owner and everyone else receive *different* spawn records.

```text
IF an owning client exists
  send to that client:            spawn record WITH client data, + update if flagged
  broadcast to everyone else:     spawn record WITHOUT client data, + update if flagged
ELSE
  broadcast to everyone:          spawn record WITHOUT client data, + update if flagged
```

**Invariants** — **the client-data blob goes only to the client that will simulate the entity.**
It is the client-side state — the part only a live client object can interpret — and sending it to
a spectator would be both wasted and a leak. This is the same single-delivery rule as in the join
replay, and the two must agree or a joining client and a spawning client end up with different
state.

The update record rides in the same packet as the spawn only when the record's flag says so. The
flag is set by whoever composed the record; the receiver reads it to know whether more follows.

**Notes** — the rules layer gets three hooks around creation — before, on, and after — and the
split matters: the pre-hook may veto, the on-hook runs before the entity is attached to its parent
and before it is broadcast, and the post-hook runs after everything. An entity that needs to see a
fully formed world uses the last.

The touch notification to the rules layer is **skipped for script-driven game modes**, because
those modes handle containment in script and a double notification would double-count. That is a
mode-specific exception hard-coded here, which is the wrong place; a rebuild should let the mode
decline the notification.

A promotion — an alife-controlled entity coming online — skips every rules hook and every veto.
It is not being created; it already exists and is merely becoming visible. That distinction is the
online/offline transition and getting it wrong duplicates entities.
