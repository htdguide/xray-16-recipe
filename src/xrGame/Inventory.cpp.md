# src/xrGame/Inventory.cpp

> One entity's carried belongings: three storage areas, a slot that is currently in the hands, and the rules that move an item between them.

**Needs** — [`Inventory.h`](Inventory.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Actor.h`](Actor.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`Weapon.h`](Weapon.h.md) · [`Grenade.h`](Grenade.h.md) · [`eatable_item.h`](eatable_item.h.md) · [`Level.h`](Level.h.md) · [`player_hud.h`](player_hud.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`Inventory.h`](Inventory.h.md); callers name that, not this file.
**Tier floor** — T2: containers, an authority check, and a state machine over item handles

## Purpose

Every entity that can carry things owns one of these. It is not a bag: it is **three
storage areas with different rules**, plus a one-item "in hands" position that is a state
machine rather than a field.

- **Slots** — a fixed, configuration-defined array of typed positions (knife, pistol,
  rifle, grenades, binoculars, bolt, outfit, detector, torch, artefact, helmet, …). At most
  one item per slot, and an item's *base slot* is a property of its section. Some slots are
  *persistent* (worn, never traded away as loose goods) and some are *activatable* (can be
  the thing in the hands).
- **Belt** — a small ordered set of quick-reach positions whose capacity is not fixed: it
  comes from the worn outfit in the two newer games and from configuration in the oldest.
- **Ruck** — the unbounded backpack.

An item is in exactly one of those at any moment and it also appears in a fourth list, `all`,
which is every carried item regardless of area. That redundancy is deliberate: weight,
lookup by identity and "do I have one of these" all want a single list, while the placement
rules all want the three separate ones.

The file is large because placing an item is not a container operation. It is a transaction
that touches the item's recorded place, the scheduler registration of the item's object,
the active-slot state machine, the owner's notification hooks, the weight total, the dirty
frame stamp, and — if this inventory belongs to whoever the player is looking through — the
inventory screen.

## State

```text
RECORD InventorySlot
  item        : optional<InventoryItem>
  persistent  : bool     # worn rather than carried; excluded from the tradeable list
  activatable : bool     # may become the item in the hands

RECORD Inventory
  slots            : list<InventorySlot>   # index 0 is the "no slot" sentinel, never filled
  last_slot        : int                   # highest configured slot index, inclusive
  all              : list<InventoryItem>   # every carried item
  belt, ruck       : list<InventoryItem>
  owner            : InventoryOwner
  active_slot      : int                   # what is in the hands now
  next_active_slot : int                   # what will be, once the current item finishes hiding
  prev_active_slot : int                   # what to restore when a block lifts
  slots_useful     : bool                  # this owner uses slots at all
  belt_useful      : bool                  # this owner has a belt at all
  max_weight       : real                  # configured carry limit
  total_weight     : real                  # cached sum; -1 means "never computed"
  max_belt         : int
  modify_frame     : int                   # frame of the last change, for UI invalidation
  blocked_slots    : list<int>             # per slot, a *count* of active blocks
  available_grenade_types : list<text>      # sorted, distinct sections of grenades held
```

**Invariants**
- An item is in exactly one of slots / belt / ruck, and simultaneously in `all`. The item
  itself records its own place, and that record must agree with which container holds it —
  most of the placement code exists to keep those two in step.
- `active_slot` and `next_active_slot` are equal when no switch is in flight. A switch is in
  flight for as long as the outgoing item is playing its hide animation.
- A slot is blocked when its counter is above zero. It is a *counter*, not a flag, because
  several independent reasons to block the same slot overlap — climbing a ladder, riding a
  vehicle, having the inventory screen open — and each must be able to lift only its own
  block.
- Index 0 of the slot array is a sentinel meaning "no slot" and never holds an item, which
  is why the first real slot index is 1 and iteration is inclusive of `last_slot`.
- `total_weight` is a cache recomputed on every placement change; reading it before any
  placement has happened is an error rather than zero.

## Construction

**Contract** — the slot layout is *data*, not code. The constructor reads the carry limit
and belt capacity, then walks configuration keys of the form "slot persistent N" from 1
upward until one is missing; that stopping point defines how many slots exist. Each slot
takes its persistent and activatable flags from configuration.

```text
FUNCTION construct()
  max_weight = config "inventory"."max_weight"
  max_belt   = config "inventory"."max_belt", default 5
  declared   = config "inventory"."slots_count" or "slots", default the built-in count

  slots = [ the sentinel slot ]
  i = 0
  WHILE true
    i = i + 1
    IF config has no "inventory"."slot_persistent_<i>"
      i = i - 1
      BREAK
    persistent  = config "inventory"."slot_persistent_<i>"
    # the oldest game's data predates the per-slot activatable key, so its
    # defaults are compiled in; the newer games must declare each one
    activatable = config "inventory"."slot_active_<i>",
                  default (built_in_default[i] IF game IS shadow_of_chernobyl ELSE false)
    slots.append(InventorySlot{ none, persistent, activatable })
  last_slot = i
  # a declared count that disagrees with the walk is a warning, not an error:
  # the walk wins
```

**Notes** — the compiled-in activatable defaults for the oldest game — knife, pistol, rifle,
grenade, binoculars, bolt and artefact yes; outfit, pda, detector, torch and helmet no — are
the one place where a data property had to be hard-coded, because that game's shipped
configuration does not carry the key and cannot be edited. The list is a compatibility
table, not a design decision.

## `Take`

**Contract** — accepts an object into this inventory and finds it a home. The caller must
already have established that the item is takeable; this routine does not re-decide. Always
succeeds: if nothing else fits, the item lands in the ruck. Side effects reach the item, the
owner, the weight cache, the UI and the network prediction list.

```text
FUNCTION take(object, do_not_activate, strict_placement)
  item = object as inventory item
  item.inventory = this
  item.drop_manual = false
  item.allow_trade()
  # an item that just gained a parent must leave the client-side prediction set:
  # prediction assumes a parentless object and corrupts a carried one
  level.remove_from_correction_prediction(object)
  all.append(item)

  IF NOT strict_placement
    item.place.type = undefined      # ignore the item's recorded place; re-decide it

  # honour a recorded place first (this is how a loaded save restores layout)
  SWITCH item.place.type
    belt: IF NOT put_in_belt(item, strict) THEN item.place.type = undefined
    ruck: IF NOT put_in_ruck(item, strict) THEN item.place.type = undefined
    slot: IF NOT put_in_slot(item.place.slot_id, item, do_not_activate, strict)
            THEN item.place.type = undefined

  IF item.place.type IS undefined
    prefer_slot = config compatibility "default_to_slot",
                  default (game IS shadow_of_chernobyl)
    IF prefer_slot OR NOT item.ruck_default
      IF can_put_in_slot(item, item.base_slot)   -> put_in_slot(...)
      ELSE IF NOT item.ruck_default AND can_put_in_belt(item) -> put_in_belt(...)
    IF still not placed
      put_in_ruck(item)            # cannot fail
      IF item is a magazine weapon AND NOT item.ruck_default AND auto_ammo_unload
        unload its magazine

  owner.on_item_take(item)
  recompute weight; stamp modify frame
  object.unregister_from_scheduler()   # a carried item is updated by its holder
  REQUIRE item.place.type IS NOT undefined
  notify the inventory screen IF this inventory is the one being looked at,
    or IF it is the corpse currently being searched
```

**Notes** — the two-stage placement (honour the recorded place, then fall back to preference
order) is what makes save/load and "picked up off the ground" the same code path. A save
restores an item with its place already set and `strict_placement` on, so it goes back
exactly where it was even into a slot that the normal rules would refuse.

`default_to_slot` differing per game is a behavioural difference between the titles, not a
bug: the oldest game puts a picked-up weapon straight into your hands, the newer ones put it
in the ruck.

## `DropItem`

**Contract** — removes an item from whichever area holds it and detaches it from its owner,
optionally creating a physics shell so it falls to the ground. If the item is currently in
the hands, the hands are emptied first — and forcibly so when the item is about to be
destroyed, because a destroyed item will never deliver the "I am hidden now" message the
normal path waits for.

```text
FUNCTION drop_item(object, just_before_destroy, dont_create_shell) -> bool
  item = object as inventory item
  object.register_with_scheduler()     # it is about to be a world object again

  SWITCH item.place
    belt: remove from belt; object.unregister_from_scheduler()
    ruck: remove from ruck
    slot:
      IF this slot is the active one AND the owner is not a dead actor
        activate(no_slot, forced = just_before_destroy)
      clear the slot; object.unregister_from_scheduler()

  remove from all
  item.inventory = none
  owner.on_item_drop(item, just_before_destroy)
  recompute weight; stamp modify frame
  drop_happened_this_frame = true
  notify the inventory screen IF this inventory is the one being looked at
  object.set_parent(none, dont_create_shell)
```

**Notes** — the scheduler registration is toggled on at the top and off again in two of the
three branches. What it means: a belted or slotted item is driven by its holder and must not
also be updated on its own; a rucked item is not driven at all. The asymmetry reads as an
oversight but is load-bearing — a rucked item stays registered and that is how it continues
to age, decay and tick down its timers in the backpack.

Removal is by search rather than by remembered position, and a missing item logs instead of
failing. That tolerance covers a real ordering hazard: several paths can drop the same item
in one frame.

## `Slot`

**Contract** — places an item into a numbered slot, evicting it from wherever it was and
possibly putting it straight into the hands. Refuses when the target holds the item already,
when the slot is occupied and placement is not strict, or — in multiplayer only — when the
item's actual parent object is not this owner. Returns whether the placement happened.

```text
FUNCTION put_in_slot(slot_id, item, do_not_activate, strict) -> bool
  IF slots[slot_id].item IS item
    RETURN false
  IF multiplayer AND item.object.parent IS NOT owner
    log a warning; RETURN false      # an ownership desync, tolerated on a client
  IF NOT strict AND NOT can_put_in_slot(item, slot_id)
    RETURN false

  slots[slot_id].item = item
  remove item from belt and from ruck
  # multiplayer additionally asserts the item was in one of those, or else
  # is genuinely parented to this owner; single player does not check
  IF item was already in a DIFFERENT slot
    IF that slot was active THEN activate(no_slot)
    clear that slot

  # put it in the hands when this slot is already the active one, or when
  # nothing at all is in the hands and the caller did not object
  IF slot_id == active_slot
     OR (active_slot == no_slot AND next_active_slot == no_slot AND NOT do_not_activate)
    activate(slot_id)

  previous = item.place
  owner.on_item_slot(item, previous)
  item.place = { slot, slot_id }
  item.on_move_to_slot(previous)
  item.object.register_with_scheduler()
  RETURN true
```

**Notes** — the owner is told *before* the item's place is updated, and the item is told
*after*. The owner's hook needs to see the old placement to undo any effect the item had
there; the item's hook needs its new placement to be true. Reversing the order breaks worn
equipment.

## `Belt`

**Contract** — moves an item onto the belt. Refuses when the belt is full, when the owner
has no belt, when the item is not belt-capable, or when the item is already on it. Emptying
the hands if the item was the active one is part of the move.

**Notes** — the removal from the ruck is skipped when the item came from a slot, because a
slotted item was never in the ruck. The scheduler registration is deactivated then
immediately reactivated on that path, which is a no-op with a bookkeeping cost; a rebuild
should register once.

## `Ruck`

**Contract** — moves an item into the backpack. Only ever refuses when the item is already
there, or on a multiplayer ownership mismatch. This is the fallback every other placement
falls back to, so it must not fail for capacity reasons — the ruck is unbounded and weight
is a *penalty*, not a limit.

As a side effect it maintains the sorted set of distinct grenade sections held, which is
what the grenade cycling key reads. Grenades are the only item type with that treatment
because they are the only type where the player switches between *sections* rather than
between slots.

## `Activate`

**Contract** — asks for a slot to become the one in the hands. It does not make it so: it
sets the *intent* and lets the per-frame update carry it out, because the outgoing item must
play its hide animation first. Server-authority only — a client never decides what is in the
hands, it is told. A blocked slot records the request as the slot to restore later and
returns.

```text
FUNCTION activate(slot, forced = false)
  IF NOT on_server
    RETURN
  target = ItemFromSlot(slot) IF slot IS NOT no_slot ELSE none

  IF target exists AND its slot is blocked AND NOT forced
    prev_active_slot = slot      # remember it, to restore when the block lifts
    RETURN
  IF slot == active_slot OR (slot == next_active_slot AND NOT forced)
    next_active_slot = slot
    RETURN
  REQUIRE slot <= last_slot
  IF slot IS NOT no_slot AND NOT slots[slot].activatable
    RETURN

  IF active_slot IS no_slot                       # hands are empty
    IF target exists
      next_active_slot = slot
    ELSE IF slot IS the grenade slot
      # asking for grenades with none in hand pulls one from the ruck first
      spare = same_slot(grenade_slot, none, search_ruck = true)
      IF spare THEN put_in_slot(grenade_slot, spare)
  ELSE IF slot IS no_slot OR target exists        # hands are full
    current = active_item
    IF current exists AND NOT forced
      current.send_deactivate()                   # begins the hide animation
    ELSE
      # forced, or the current item is being destroyed: switch instantly,
      # because no hide-finished message will ever arrive
      IF target THEN target.activate()
      active_slot = slot
    next_active_slot = slot
```

**Invariants** — after any call, `next_active_slot` holds the intent; `active_slot` changes
only in the forced path here, and otherwise only in the per-frame update.

**Notes** — the grenade special case is the one place where asking to activate a slot
*moves an item*. It exists because grenades are consumed: the slot empties itself on every
throw, and the player pressing the grenade key expects the next one to appear rather than
nothing to happen.

## `Update`

**Contract** — runs once per frame on the server side and is the only place a pending hand
switch completes. Everything about the switch is gated on the outgoing item having finished
hiding, and on the first-person presentation layer being willing to accept the incoming one.

```text
FUNCTION update()
  IF on_server AND active_slot != next_active_slot
    IF this inventory belongs to the entity the player sees through
       AND the incoming item exists
       AND the first-person presentation refuses it right now
      RETURN                                  # try again next frame

    IF the current item exists and is not yet hidden
      IF it is idle and staying idle
        it.send_deactivate()                  # nudge a stalled hide
      update drop tasks; RETURN               # the switch waits

    IF a deferred grenade switch is pending
      carry it out now
    IF next_active_slot IS NOT no_slot
      incoming = ItemFromSlot(next_active_slot)
      IF incoming exists
        IF its slot became blocked meanwhile
          activate(active_slot)               # abandon the switch
          RETURN
        incoming.activate()
    active_slot = next_active_slot

  # recovery: the hands hold an item that somehow ended up hidden
  IF next_active_slot IS NOT no_slot AND active_item is hidden
    active_item.activate()
  update drop tasks
```

**Notes** — the "nudge a stalled hide" branch resends the deactivate to an item that is idle
and will stay idle. That is a repair for a lost transition, not a normal step, and it is why
a switch that should take one animation sometimes visibly takes two.

The final recovery clause has no counterpart in the ordinary flow; it exists because several
paths can hide the active item without clearing the active slot.

## `UpdateDropTasks` / `UpdateDropItem`

**Contract** — walks every carried item, in every area, once per frame, looking for the
one-shot "drop me" flag an item can set on itself. An item that raises it is refused for
trade and then dropped through the ownership-transfer message. Consuming the flag is part of
the visit, so an item drops exactly once.

**Notes** — this is a *polled* channel where an event would do. Its cost is a full walk of
the inventory every frame per owner; its reason is that the flag can be raised from inside
the item's own update, where dropping immediately would destroy the object mid-update.

The per-frame walk also carries the "something was dropped last frame" notification to the
owner, delayed by one frame so the owner sees a settled inventory.

## `CanTakeItem`

**Contract** — whether this inventory would accept an item. Refuses a destroyed item, an
item that refuses to be taken, and an item that would exceed the carry limit — *except* for
the player, who is never weight-blocked from picking something up. Holding an item already
present is a hard error rather than a refusal.

**Notes** — the player being exempt is the decision: weight in this game slows you down and
eventually stops you walking, but it never prevents a pickup. Non-player carriers get a hard
cap because nothing gives them feedback about being overloaded.

## `CanPutInSlot` / `CanPutInBelt` / `CanPutInRuck`

**Contract** — the three placement predicates, each pure.

- A slot accepts when slots are in use at all, the owner agrees (owners have their own
  per-slot rules), the slot is empty, and — for the helmet slot specifically — the worn
  outfit permits a separate helmet. That last is why some outfits have integral headgear.
- The belt accepts when the owner has a belt, the item is belt-capable, the belt is below
  capacity, and the belt's two-dimensional packing can fit the item's footprint. Belt
  positions are not interchangeable cells; items occupy a width.
- The ruck accepts everything not already in it.

## `BeltWidth` / `BeltMaxWidth`

**Contract** — the belt's current capacity and its configured ceiling. In the oldest game
capacity is a constant from configuration. In the newer two it is a property of the worn
outfit, and a character with no outfit has **no belt at all**. That is the mechanic behind
artefact carrying: the outfit is what lets you carry artefacts on your belt where they take
effect.

## `Same` / `SameSlot` / `Get` / `GetAny` / `item` / `get_object_by_id` / `GetItemFromInventory`

**Contract** — the lookup surface, all linear scans over one of the containers. They differ
in key (section name, class identifier, entity identifier, base slot) and in which container
they search, and the name-keyed and class-keyed ones additionally require the item to be
*useful* — an exhausted or broken item is skipped, so "do I have a medkit" does not find an
empty one.

**Notes** — the identifier-keyed lookup does not check usefulness, and must not: it is asked
about a specific item, not about a capability.

One of these hashes the requested section name and compares hashes rather than strings. That
is a scan over the same list as its neighbours with a cheaper comparison, and the tree's only
such optimization — nothing suggests it was measured.

## `dwfGetSameItemCount` / `dwfGetGrenadeCount` / `bfCheckForObject` / `dwfGetObjectCount` / `tpfGetObjectByIndex`

**Contract** — the counting and indexing surface the scripts use. Counting by section and
counting grenades each take a "search everything or just the ruck" switch.

**Notes** — the grenade count ignores the section it is handed and counts by two hard-wired
class identifiers instead. That is a defect visible from script: asking for the count of one
grenade type returns the count of both. A rebuild should count by section like its neighbour.

Indexing by position walks the list counting rather than indexing it directly. The list is
random-access; the walk is not a decision.

## `Eat` / `ClientEat`

**Contract** — consuming an item. The authoritative half (`Eat`) verifies a chain of
ownership relations before anything happens — the item must be edible, the owner must be a
living creature with an inventory, the item's inventory must be *this* one, the owner's
inventory must be this one, and the object's parent must be the owner — then applies the
item's effect, gives a script hook a veto, fires the use callback, refreshes the screen, and
marks an emptied item for dropping. The client half (`ClientEat`) verifies the same chain
and then sends the request to the server instead of acting.

**Invariants** — the five-way ownership check is not defensive padding. Eating is reachable
from the inventory screen, from a script and from a network message, and each can arrive
after the item has moved.

**Notes** — the script hook runs *after* the item's effect has already been applied, so a
script returning a refusal stops the bookkeeping but not the effect. Treat the hook as an
observer; a rebuild wanting a real veto must move it ahead of the use.

An emptied item that declines to be deleted causes the whole operation to report failure
even though the item was consumed. That is how refillable containers stay in the inventory.

## `Action` / `ActiveWeapon` / `SendActionEvent`

**Contract** — the input funnel. A command arrives, and three things may happen to it in
order: the actor records a deterministic random seed for shot and zoom spread; a client
forwards the command to the server; and the item in the hands gets first refusal. Only what
the item does not consume reaches the inventory's own bindings — the six direct slot keys
and the artefact key.

**Invariants** — the shot and zoom seeds are captured on the client *before* the command is
forwarded and are sent with it, so the server reproduces the same spread the client drew.
That is the whole reason those two commands are special-cased.

**Notes** — pressing the key for the slot already in the hands means "put it away" in single
player and "cycle to the next item in this slot" in multiplayer. The difference exists
because multiplayer loadouts put several weapons in one slot.

The sixth direct slot key is refused outside single player. Nothing states why; the slot it
names is the bolt, which multiplayer does not have.

## `AddAvailableItems`

**Contract** — builds the list of items a trade or corpse-search screen may show: everything
in the ruck, everything on the belt if there is one, and from the slots only the
non-persistent ones — with grenades exempted from that rule. Optionally filters each
candidate through a script predicate.

**Notes** — the persistent-slot exclusion is what keeps worn equipment out of the trade
list, and the grenade exemption exists because grenades live in an activatable slot but are
ordinary goods.

The script predicate is looked up by name once per item rather than once per call. A rebuild
should hoist it.

## `SetSlotsBlocked` / `BlockSlot` / `UnblockSlot` / `IsSlotBlocked` / `TryActivatePrevSlot` / `TryDeactivateActiveSlot`

**Contract** — the blocking system: a bitmask of slots is blocked or unblocked as a unit,
each slot carrying an independent counter so overlapping reasons nest correctly. Blocking
then tries to empty the hands if what they hold has become blocked; unblocking tries to put
back whatever was displaced.

**Invariants** — a slot's counter never goes negative and is expected to stay below five,
which bounds how many simultaneous reasons the game actually has. Server authority is
required, with a replay of a recorded demo as the one exception.

```text
FUNCTION set_slots_blocked(mask, block)
  FOR EACH slot i IN first_slot .. last_slot
    IF mask has bit i
      IF block THEN blocked[i] = blocked[i] + 1 ELSE blocked[i] = blocked[i] - 1
  IF block THEN try_deactivate_active_slot() ELSE try_activate_prev_slot()

FUNCTION try_deactivate_active_slot()
  # the item in hand, or the one on its way in, may have just become unreachable
  IF the active item's slot is now blocked or unactivatable
    it.discard_state()          # abandon any animation in progress
    activate(no_slot)
    prev_active_slot = the slot it was in
  ELSE IF the incoming item's slot is now blocked or unactivatable
    activate(no_slot)
    prev_active_slot = that slot

FUNCTION try_activate_prev_slot()
  IF the hands are empty (now or pending) AND a slot was remembered
    IF its item exists, its slot is unblocked and activatable
      activate(that slot)
      forget the remembered slot
```

**Notes** — the remembered slot is a single value, so nested blocks from different sources
that displace different items lose all but the last. In practice only one item is ever in
the hands, which makes the single slot sufficient; a rebuild layering more sources should
make it a stack.

The blocking masks themselves are named constants: climbing a ladder and riding a vehicle
block the rifle and binocular slots, while the inventory screen and the buy menu block
everything.

## `TotalWeight` / `CalcTotalWeight`

**Contract** — the cached sum of carried item weights, and the recomputation that refreshes
it. Recomputation is a full walk and happens on every placement change; reading the cache
before any recomputation is an error, not zero.

## `Clear`

**Contract** — empties every container and every slot and drops the owner link, without
notifying anybody. It is teardown, not a gameplay operation — items are not dropped, just
forgotten, so the caller must already have disposed of them.

## `InvalidateState` / `ModifyFrame`

**Contract** — stamps the current frame number as the moment of last change. The inventory
screen compares its own stamp against this one to decide whether to rebuild, which turns
"did anything change" into an integer comparison instead of a structural diff.

## `Items_SetCurrentEntityHud`

**Contract** — called when this owner becomes or stops being the entity the player sees
through; re-initializes every carried weapon's attachments and their visibility, because
attachment visuals are bound to the first-person presentation and must be rebuilt for the
new viewpoint.

**Notes** — the parameter saying which direction the change went is ignored; the routine
does the same work either way.

## `isBeautifulForActiveSlot`

**Contract** — whether any item already in a slot declares the candidate item *necessary* to
it — ammunition for a held weapon, a battery for a held device. Used to decide whether
picking something up should draw attention to it. Always true outside single player.

## `InSlot` / `InBelt` / `InRuck`

**Contract** — which area holds an item. The slot test reads the item's own recorded place
and asserts the slot agrees; the other two search their container by entity identifier. The
asymmetry is why the item's recorded place must be kept in step with the containers.
