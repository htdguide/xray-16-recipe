# src/xrGame/alife_trader_abstract.cpp

> What it means, on the server side, for an entity to carry an inventory: how its contents are created, and the two hand-offs that move a container's whole inventory across the online/offline boundary.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`specific_character.h`](../xrServerEntities/specific_character.h.md) · [`xrServer.h`](xrServer.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai_debug.h`](ai_debug.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — [`alife_trader.cpp`](alife_trader.cpp.md)
**Tier floor** — T2: registry hand-offs and identifier reassignment; the spawn message it emits is a frozen byte stream, delegated to the packet writer

## Purpose

The word *trader* is historical. What this file actually implements is the server-side
behaviour of **any entity that owns other entities** — a trader, a stalker, a corpse, a
box. Three separate concerns live here because all three are about the parent/child
relationship between server objects:

- **spawning a character's starting inventory** from its authored profile;
- **attach and detach**, the only two operations that change the parent/child link;
- **promotion and demotion of a container's contents**, which is where the interesting
  decisions are.

The file is split from [`alife_trader.cpp`](alife_trader.cpp.md) only because one is the
abstract behaviour and the other is the concrete trader class. The split is arbitrary
from a rebuild's point of view; merge them.

## State

`Stateless.` Every function here operates on state owned by the server object it is
handed. The two things it reads that are not obvious:

```text
# on the container (a dynamic server object)
  children     : list<entity identifier>   # invariant: a child's parent field names this container
  position, level vertex, game vertex      # a child's copy of these is meaningless while attached

# on the container-as-trader
  specific_character : text                # the authored character profile, cleared after use
  community_index    : int                 # cleared with it
```

## `spawn_supplies`

**Contract** — give a newly created character its authored starting inventory. Runs once,
at spawn. Allocates entities into the world.

```text
FUNCTION spawn_supplies()
  spawn a personal data device as a child of this entity
  set the device's original owner to this entity
  resolve the character profile                  # picks one profile out of the authored set
  clear the specific-character and community fields
  copy the resolved profile name onto the device

  IF a profile was resolved THEN
    supply = true
    IF this entity's spawn record carries per-instance configuration text THEN
      parse that text as a configuration file
      IF it declares a "dont_spawn_character_supplies" section THEN supply = false
    IF supply THEN
      load the profile and spawn every item its supply list names
```

**Invariants** — the profile name is **consumed**: it is copied onto the data device and
then cleared on the entity. The device becomes the entity's identity from that moment on,
which is why a character's name and faction survive being killed and looted — they live on
an item, not on the corpse.

**Notes** — the opt-out is a *section that need only exist*, with no keys read. That is the
cheapest possible flag in the configuration format and it is how level designers place a
named character who must arrive empty-handed. A rebuild needs the same escape hatch,
addressed by the same name, because it appears in shipped level data.

The item spawn order is fixed: the data device first, then the profile's list. Anything
that scans a character's inventory expecting the device at a known place depends on it.

## `attach`

**Contract** — record that an inventory item now belongs to this container. Refuses to
assert twice on the same item.

```text
FUNCTION attach(item, alife_requested, add_to_children)
  IF not alife_requested THEN RETURN      # the client side is only mirroring a decision
  item.parent = this.id
  IF not add_to_children THEN RETURN
  REQUIRE item.id is not already in children
  append item.id to children
```

**Notes** — the two flags exist because the same call arrives from three directions: the
alife simulation deciding an item changed hands, the client side replaying that decision,
and the load path rebuilding a tree whose child list has already been read from the save.
Each needs a different subset of the work. A rebuild with separate entry points for
"transfer ownership", "mirror a transfer" and "restore a link" will be clearer and must
still do exactly these steps for each.

The duplicate check is a hard assertion, not a tolerated condition: an item in a container
twice is counted twice by every mass, volume and value computation in the game.

## `detach`

**Contract** — remove an item from this container and give it a place in the world.

```text
FUNCTION detach(item, known_position_in_children, alife_requested, remove_from_children)
  # the item inherits the container's place in the world — it had none while carried
  item.position    = this.position
  item.level vertex = this.level vertex
  item.game vertex  = this.game vertex
  item.distance     = this.distance

  IF not alife_requested THEN RETURN
  item.parent = none
  IF a position in children was supplied THEN erase at that position; RETURN
  IF not remove_from_children THEN RETURN
  REQUIRE item.id is in children
  erase it
```

**Invariants** — a carried item's own position is **stale by design** and is only made
meaningful here. Nothing may read an attached item's position; the container's is the
truth. This is the reason a dropped item appears exactly where its owner stood rather than
where it was picked up, and it is why the four fields are copied together rather than
only the position: the navigation vertices must agree with the position or the item is
unreachable by the pathfinder.

The caller may pass the item's known position in the child list, because the common caller
is already iterating that list and erasing by search inside its own loop would invalidate
its cursor. That is an incidental detail of one iteration style, but the *contract* — the
caller may hand back where it found the item — is worth keeping.

## `add_online` — promoting a container's contents

**Contract** — the container has just been promoted to online. Its children must become
live client objects too. Emits one spawn message per child; frees the previous server-side
instance of each.

```text
FUNCTION add_online(container, update_registries)
  FOR EACH child_id IN container.children
    IF child_id is the actor THEN CONTINUE          # the actor is never anyone's cargo
    child = objects().object(child_id)
    IF none OR child is not an inventory item THEN CONTINUE

    mark the child's spawn record as carrying an update payload
    destroy the child's existing server-side entity
    child.position     = container.position
    child.level vertex = container.level vertex
    process a spawn for the child, as the server's own client
    clear the update-payload mark
    child.online = true

  IF not update_registries THEN RETURN
  remove the container from the offline schedule
  remove the container from its game-graph vertex
```

**Invariants** — the spawn-plus-update mark is set before the spawn and cleared after. It
makes the emitted message carry not just the item's spawn record but its *current* state,
which is the only way a half-used medkit or a partly-loaded magazine survives promotion.
Conformance criterion 11 is exactly this.

Each child is re-placed at the container's position before its spawn is processed, for the
same reason `detach` does it: the child's own stored position is stale.

**Notes** — the actor is skipped explicitly. A container can legitimately have the player
as a child — the player sitting in a vehicle is the vehicle's child — and the player is
already online by definition; spawning them again would register an entity twice.

The registry removals are conditional because promotion also happens as part of a larger
operation (a level change, a save restore) that has already taken the container out of
the offline registries. Doing it twice is the error the flag prevents.

## `add_offline` — demoting a container's contents

**Contract** — the container is going offline. Its children must become records again.
Takes the list of children the container had *while online*, which may differ from what it
has now.

```text
FUNCTION add_offline(container, saved_children, update_registries)
  FOR EACH child_id IN saved_children
    child = objects().object(child_id, tolerate_missing)
    REQUIRE child exists
    child.online = false
    REQUIRE child is an inventory item

    child.id = server.reassign_identifier(child.id)    # see below

    IF the child must not be saved THEN
      release it entirely and skip it
    ELSE
      discard its client-side data
      register it on its game-graph vertex
      attach it to the container in the graph registry

  IF not update_registries THEN RETURN
  add the container to the offline schedule
  add the container to its game-graph vertex
```

**Invariants** — every child's **entity identifier is reissued** on the way offline. That
looks gratuitous and is not: identifiers are a scarce 16-bit resource shared between the
server's own objects and the identifiers handed out to clients, and an item that was live
holds one from the client-facing part of the space. Demotion returns it. A rebuild with a
wider identifier space can skip this — but the identifier width is frozen by the save
format and the network protocol, so it probably cannot.

An item that reports it must not be saved is **destroyed here, not preserved**. That is
how ammunition already fired, a temporary quest token, or a client-only effect object
fails to come back: going offline is the garbage collection point for the inventory.

**Notes** — the *saved* child list is passed in rather than read from the container
because demotion runs after the client side has already been torn down, and the container's
own child list at that moment reflects the server's view, which the online period may have
diverged from. The caller captures the list at the right moment; this function must not
second-guess it.

The two registry operations at the end are the mirror image of `add_online`'s removals,
and the order is reversed: children first, then the container. A container placed back on
its graph vertex before its children exist is briefly a container of nothing, which the
offline simulation will happily act on.
