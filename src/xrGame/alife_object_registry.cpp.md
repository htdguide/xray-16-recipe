# src/xrGame/alife_object_registry.cpp

> The master table of every alife server object in the world, and the save/load of that table as a parent-first tree of spawn-plus-update packets.

**Needs** — [`alife_object_registry.h`](alife_object_registry.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`ai_debug.h`](ai_debug.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: owns entity lifetime and writes a frozen byte stream; the stream format is the only T1-adjacent part and it is delegated to the packet writer

## Purpose

Exactly one registry holds every server object the alife simulation knows about, on every
level, online or offline, keyed by entity identifier. Every other alife component —
the graph registry, the smart-terrain registry, the schedule — stores identifiers and
resolves them here. That indirection is deliberate: an entity's identity is a 16-bit
handle that survives a save and a network hop, whereas its address in memory does not.

The file also owns the save and load of that table, which is the bulk of a saved game.

## State

```text
RECORD ObjectRegistry
  objects : map<EntityId, owned ALifeDynamicObject>   # sorted by identifier
```

Invariants:

- the registry **owns** every object in it; destroying the registry destroys them all;
- an identifier maps to at most one object, and an object appears at most once — both
  directions are checked on insert, because a duplicate is the failure that
  conformance §6 names first ("no entity is registered twice");
- iteration order is by identifier, and save relies on that only for reproducibility, not
  for correctness.

## Destruction

**Contract** — two passes over the table: every object is told it is being unregistered,
and only then is any object destroyed.

**Invariants** — the split is load-bearing. An object's unregister hook may reach other
objects through the registry (a child detaching from a parent, a creature deregistering
from a smart terrain), so no memory may be released until every hook has run. A single
combined pass would hand a live object a reference to a freed one. A rebuild with
automatic memory management still needs the two phases, because the hook is about
*logical* deregistration, not about freeing.

## `save`

**Contract** — writes the whole registry into a chunk of the save stream. Allocates a
packet buffer per object; blocking; called from the save path only.

```text
FUNCTION save(stream)
  open chunk OBJECT_CHUNK_DATA
  count_position = stream.position
  stream.write_int32(placeholder)       # patched after the walk; the count is unknown until then

  count = 0
  FOR EACH (id, object) IN objects
    IF NOT object.can_save()      -> CONTINUE
    IF object.redundant()         -> CONTINUE
    IF object.parent_id is set    -> CONTINUE      # children are written by their parent
    save_subtree(stream, object, count)

  patch count into count_position
  close chunk
```

**Invariants** — the three filters are a single rule stated three ways: *write each
savable root exactly once*. `can_save` is the entity kind's own veto (projectiles,
temporary effects); `redundant` is the simulation's veto for an object that will be
recreated rather than restored; the parent test is what makes the walk a forest rather
than a flat list. Dropping any one of them produces either a duplicated entity on load or
an orphan with a parent identifier pointing at nothing.

The count is written as a placeholder and patched afterwards rather than counted first.
A rebuild is free to count first instead; the reader needs the count *before* the objects,
so the only requirement is that it precede them in the stream.

## `save_subtree` — one object and its descendants

**Contract** — writes one object followed by its savable children, depth first, parents
before children. Recursion depth is the containment depth (a stalker holding a backpack
holding a weapon holding a magazine), which is small and bounded by authored data.

```text
FUNCTION save_subtree(stream, object, count)
  count = count + 1

  packet = new packet
  object.write_spawn(packet, as_save = true)      # identity, class, position, parent link
  stream.write_int16(packet.length)
  stream.write_bytes(packet)

  packet = new packet tagged UPDATE
  object.write_update(packet)                     # the mutable half: health, inventory, timers
  stream.write_int16(packet.length)
  stream.write_bytes(packet)

  FOR EACH child_id IN object.children
    child = registry.lookup(child_id, tolerate_missing = true)
    IF child is absent        -> CONTINUE
    IF NOT child.can_save()   -> CONTINUE
    save_subtree(stream, child, count)
```

**Invariants** — this is the single most order-sensitive routine in the save path.

- Each object occupies **two** length-prefixed packets, spawn then update, in that order,
  and the reader depends on exactly that pairing. The split is not an accident of
  convenience: the spawn packet is the same bytes the network protocol sends to create an
  entity on a client, and the update packet is the same bytes that later mutate it, so
  saving is literally "record the creation message and the latest state message".
- Each length is a 16-bit prefix, which caps a single entity's serialized state at 65535
  bytes. That cap is frozen by the format and is not generous: an entity with a very
  large inventory is the realistic way to exceed it.
- The spawn write is told it is a *save* rather than a network send, which is what makes
  it emit the fields that only matter across a reload.
- A child whose identifier is present in a parent's child list but absent from the
  registry is skipped silently rather than treated as corruption. That tolerance is what
  lets a level transition drop objects without invalidating every parent.
- A child that vetoes saving is skipped *together with its own subtree*, so a
  non-savable container takes its contents with it.

## `load`

**Contract** — replaces the registry's contents from a save stream. Fails hard if the
chunk is missing or if any record is not an alife object. Blocking.

```text
FUNCTION load(stream)
  REQUIRE stream has chunk OBJECT_CHUNK_DATA   ELSE FAIL WITH missing chunk
  objects.clear()
  count = stream.read_int32()
  REPEAT count TIMES
    object = read_object(stream)
    add(object)
```

**Invariants** — the stream is a flat sequence of `count` objects, *not* a tree: the
parent/child structure is reconstructed from the identifiers inside each spawn packet,
not from the nesting. The depth-first write order therefore guarantees a parent is added
before any of its children, which is what lets a child's registration resolve its parent
immediately.

Clearing the table before the read does not destroy the previous objects — the load path
runs on a registry whose contents were already disposed of elsewhere. A rebuild should
make that explicit rather than inherit it; leaking here is invisible because a load is
usually preceded by a full teardown.

**Notes** — the original stages the objects into a scratch array sized by the count before
inserting them, using a stack allocation proportional to the entity count. That is a
detail with no consequence — the array is written and never read — and a rebuild should
simply insert as it goes. It is worth naming only because a save with a very large entity
count makes a stack allocation proportional to that count, which is a latent limit
nothing else in the format imposes.

## `read_object`

**Contract** — reads one object's two packets and materializes it. A pure function of the
stream plus the class factory; does not register the result. Fails hard on a malformed
stream.

```text
FUNCTION read_object(stream) -> ALifeDynamicObject
  length = stream.read_int16()
  packet = stream.read_bytes(length)
  tag = packet.read_message_tag()
  REQUIRE tag == SPAWN                ELSE FAIL WITH wrong packet tag

  class_name = packet.read_string()
  entity = entity_factory.create(class_name)
  REQUIRE entity exists               ELSE FAIL WITH unknown entity class
  REQUIRE entity is an alife object   ELSE FAIL WITH non-alife object in save
  entity.read_spawn(packet)

  length = stream.read_int16()
  packet = stream.read_bytes(length)
  tag = packet.read_message_tag()
  REQUIRE tag == UPDATE               ELSE FAIL WITH wrong packet tag
  entity.read_update(packet)

  RETURN entity
```

**Invariants** — the class is selected by the **name string stored in the spawn packet**,
read before the factory is consulted; the factory is the registration table filled at
startup. This is why a save from a build with an unknown entity class cannot be loaded
rather than partially loaded: the engine refuses mismatches instead of guessing, exactly
as §5 says.

The two message tags are verified rather than assumed. They are the only integrity check
in the format — there is no checksum on the object chunk — so a rebuild should keep them
even though they are redundant with the length prefixes in a well-formed stream.

The routine is a free function of the registry rather than a method, because the network
path needs the same "packet pair to entity" decoding without a registry to put the result
in.
