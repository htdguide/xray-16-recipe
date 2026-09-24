# src/xrGame/xrServer_sls_clear.cpp

> Empties the world: destroys every entity, children before parents, announcing each destruction as a backdated event.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`ai_space.h`](ai_space.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: recursive teardown of an ownership tree with wire announcements

## Purpose

Changing level, ending a session or loading a save all need the current world gone. Gone means
every entity destroyed *and* every client told, and the ownership tree makes the order
non-obvious: destroying a backpack before the items in it leaves orphans.

## State

`Stateless.`

## `Perform_destroy`

**Contract** — destroy one entity and everything it contains. **Requires the entity to have no
parent** — it must be a root. Releases each child from it, destroys that child recursively, then
destroys the entity itself and broadcasts a destruction event for it.

```text
FUNCTION perform_destroy(entity, send_mode)
  REQUIRE entity.parent is none                   # only roots may be destroyed directly

  WHILE entity has children
    child := entity.children.last
    REQUIRE child exists in the table             # a registered child that is missing is fatal
    perform_reject(child, entity, backdate: 2 * assumed_latency)   # child becomes a root
    perform_destroy(child, send_mode)             # now legal: it has no parent

  id := entity.id
  entity_destroy(entity)
  broadcast EVENT(time: now - 2 * assumed_latency, DESTROY, id)
```

**Invariants** — the recursion is **release-then-destroy**, never destroy-in-place. Releasing the
child first makes it a root, which is the precondition the recursive call asserts. That is why the
function can assert its precondition at all, and the assertion is what catches an inconsistent
tree at the moment it matters.

Children are taken from the **back** of the list, and the release removes from wherever the entry
is, so the loop terminates regardless of list order.

**Every timestamp here is backdated by twice the assumed network latency.** The same technique as
in [`xrServer_perform_transfer.cpp`](xrServer_perform_transfer.cpp.md): the destruction is dated
before anything a client could still have in flight, so a client's pending message about the
entity is ordered *after* its own destruction and discarded rather than applied to nothing.

**Notes** — a child registered in a parent's list but absent from the entity table is a hard
failure with the missing identifier named. That is the one place the parent/child invariant from
[`xrServer.h`](xrServer.h.md) is checked outside the debug sweep, and it is checked here because
this is where violating it would silently leak an entity.

## `SLS_Clear`

**Contract** — destroy every entity. Repeatedly finds any root and destroys it recursively, until
the table is empty. When entities remain and none is a root — a cycle, or an entity naming a parent
that does not exist — logs every survivor with its parent and **abandons the table**, leaking them
rather than looping forever.

```text
FUNCTION clear_level_state()
  WHILE entities is non-empty
    root := any entity with no parent
    IF root EXISTS
      perform_destroy(root, reliable)
    ELSE
      log every remaining entity, its identifier and its parent
      log that the world could not be fully destroyed
      abandon the table
      BREAK
```

**Invariants** — scanning for a root each iteration, rather than collecting roots once, is
necessary: destroying one root releases its children, which become roots, and the table is mutated
throughout.

**Notes** — the scan-from-the-start-every-time makes this quadratic in the entity count, which for
a level of a few thousand entities at a level change is acceptable and would not be in a hot path.
A rebuild can collect roots into a work list and push newly orphaned children onto it.

**The failure branch is a deliberate leak.** Its only alternative — destroying entities in an
arbitrary order — would run destructors against dangling parent references, and the diagnostic
listing is worth more than the memory. A rebuild reaching this branch has a bug in its ownership
bookkeeping and should treat the log as the bug report it is.
