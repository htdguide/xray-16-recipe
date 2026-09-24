# src/xrGame/MPPlayersBag.cpp

> The bag a killed multiplayer player's belongings drop into: a container that is itself an inventory item, and that removes itself once the match's item-lifetime rule says so.

**Needs** — [`MPPlayersBag.h`](MPPlayersBag.h.md) · [`Level.h`](Level.h.md) · [`game_base_space.h`](../xrServerEntities/game_base_space.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — [`MPPlayersBag.h`](MPPlayersBag.h.md)
**Tier floor** — T3: an entity with two event cases and a lifetime predicate

## Purpose

When a player dies in a multiplayer match their gear must go somewhere other players can
take it, and it must not accumulate forever. This class is that somewhere: an inventory
item that can also own inventory items, so a bag can be picked up whole or looted piece by
piece, and which volunteers for destruction when the match's weapon-removal rule expires.

## State

`Stateless` beyond what it inherits. The only constant it introduces:

```text
BAG_REMOVE_TIME = 60000 ms   # how long a bag survives after being dropped, when the
                             # match is configured to remove items on a timer
```

## `CMPPlayersBag::OnEvent`

**Contract** — handles the two ownership events on top of the inherited item behaviour.
Taking an item into the bag reparents it and snaps it to the bag's own position; rejecting
one clears its parent, passing through an optional flag that the sender may or may not have
included.

**Invariants** — an item entering the bag must not already belong to an inventory. This is
asserted, not handled: an item in two containers is a state the rest of the engine has no
way to express.

```text
FUNCTION OnEvent(message, kind)
  inherited OnEvent(message, kind)
  SELECT kind
    CASE OWNERSHIP_TAKE
      id = read entity identifier
      item = find object(id)
      REQUIRE item belongs to no inventory
      item.parent = this
      item.position = this.position       # the contents are AT the bag, not where they fell
    CASE OWNERSHIP_REJECT
      id = read entity identifier
      item = find object(id)
      # The trailing flag says whether the item should be dropped actively (given the
      # bag's motion) rather than simply released. Older senders omit it entirely, so
      # its absence must read as false rather than as a malformed message.
      item.detach(active: message has more bytes AND that byte is non-zero)
  END SELECT
```

**Notes** — snapping a taken item to the bag's position matters because the item keeps a
world position even while owned, and that position is what the physics world and the
renderer would otherwise use if the item were briefly visible during a transfer.

The optional trailing byte is a wire-compatibility accommodation: the message grew a field
and the reader must tolerate both lengths. A rebuild versioning its own protocol should
make the field mandatory.

## `CMPPlayersBag::NeedToDestroyObject`

**Contract** — a predicate the object's own update asks before volunteering for destruction.
Pure; no side effects.

```text
FUNCTION NeedToDestroyObject() -> bool
  IF single player           THEN RETURN false   # single player never sheds loot
  IF this is a remote object THEN RETURN false   # only the authority may destroy it
  IF the bag has a parent    THEN RETURN false   # someone is carrying it
  IF removal rule == -1      THEN RETURN false   # "never remove"
  IF removal rule == 0       THEN RETURN true    # "remove immediately once dropped"
  RETURN time since it became independent > BAG_REMOVE_TIME
```

**Invariants** — the guards are ordered from cheapest and most absolute to most conditional,
and the remote check must come before anything else that could return true: a client
destroying an object the server still owns desynchronizes the entity registry.

**Notes** — the removal rule is a single tunable with three meanings (never, immediately,
on a timer), which is why it is an integer rather than a flag. The timer's length is not
part of that tunable — it is fixed at one minute here — so a server operator can turn the
behaviour on and off but cannot change how long a bag lasts. That looks like an oversight
and a rebuild should make the interval configurable alongside the mode.
