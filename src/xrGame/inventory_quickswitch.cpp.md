# src/xrGame/inventory_quickswitch.cpp

> One key cycles the weapon in a slot through the backpack by authored priority, and one key cycles the grenade in hand through the types carried — the two "next thing" bindings.

**Needs** — [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`Grenade.h`](Grenade.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a ranked search over a container, plus a cycle over a sorted list

## Purpose

Two unrelated mechanics that share a shape — "give me the next one" — and a file.

**Quick switch** is the multiplayer weapon wheel. A player holding a rifle presses one key
and the best other rifle-class weapon in their backpack takes its place. "Best" is not
authored per weapon: it is a *class* priority (pistols, shotguns, assault rifles, sniper
rifles, heavy weapons) that differs depending on which of the two weapon slots is being
cycled, and within a class it is a value function over cost and loaded ammunition.

**Grenade cycling** is the single-player grenade key: the thrown weapon in hand advances to
the next grenade *type* carried.

Both live outside [`Inventory.cpp`](Inventory.cpp.md) because both are policy over the
inventory rather than inventory mechanics, and both can be replaced without touching
placement.

## State

```text
RECORD PriorityGroup
  sections : set<text>        # the weapon sections belonging to one class

RECORD QuickSwitchState        # fields of the inventory
  groups            : list<PriorityGroup>   # five, one per weapon class
  slot2_priorities  : list<PriorityGroup>   # the five groups in the order slot 2 prefers
  slot3_priorities  : list<PriorityGroup>   # likewise for slot 3
  null_priority     : PriorityGroup         # empty; returned for any other slot
  next_items_exceptions : set<InventoryItem>  # already offered this cycle
  next_item_iteration_time : int              # when the last offer was made
```

**Invariants**
- The priority arrays are permutations of the same five groups. Two slots therefore rank
  the same weapon classes differently, which is the whole point.
- The exception set always contains the currently held item, so a cycle never offers back
  what is already in hand.
- The exception set is cleared after a period of inactivity, which is what makes the cycle
  restart rather than run out.

## `InitPriorityGroupsForQSwitch`

**Contract** — reads the five weapon-class memberships from configuration and installs them
into the two per-slot priority orders. Run once when an inventory that uses quick switch is
constructed.

```text
FUNCTION InitPriorityGroupsForQSwitch()
  FOR EACH class IN (pistols, shotgun, assault, sniper_rifles, heavy_weapons)
    groups[class] = comma_list of sections from config "deathmatch_team0"."<class name>"

  # the two slots rank the same five classes differently
  slot2_priorities = [ pistols, shotgun, assault, heavy_weapons, sniper_rifles ]
  slot3_priorities = [ assault, sniper_rifles, shotgun, heavy_weapons, assault ]
```

**Notes** — the class memberships come from a *multiplayer team loadout section*. That is
not a general weapon taxonomy; it is the list of weapons one team may buy in one game mode,
reused as a classification because it is the only place in the shipped data where weapons
are grouped by class at all. The consequence is that a weapon not purchasable in that mode
is in no class and can never be quick-switched to. This is the mechanic's real limitation
and it is data, not code.

The two orders are the tuning. Cycling the *pistol* slot prefers pistols, then shotguns,
then assault rifles — a pistol-slot player who has run out of pistols gets the next smallest
thing. Cycling the *primary* slot prefers assault rifles, then sniper rifles, then shotguns.
Sniper rifles rank last for the pistol slot and second for the primary, which is the
statement that a sniper rifle is a specialist primary and never a fallback sidearm.

The slot-3 table has a hole: its first and last entries are both the assault group, and
the pistol group appears nowhere. The pistol entry was replaced by the assault group and
the original left commented out beside it. The effect is that cycling the primary slot never
offers a pistol — deliberate, if inelegantly expressed.

## `GetNextItemInActiveSlot`

**Contract** — finds the next weapon to offer for the currently held slot. Searches the
backpack only, one priority class at a time, and falls through to the next class when a
class yields nothing. Two full passes: the first requires loaded ammunition, the second
accepts an empty weapon. Returns nothing when both passes are exhausted, and resets the
cycle when it does. Recursive over the priority classes.

```text
FUNCTION GetNextItemInActiveSlot(priority, ignore_ammo) -> optional<InventoryItem>
  IF now - last_offer_time >= exception_clear_period
    exceptions = { the item currently in hand }      # idle long enough: start the cycle over

  best = best_in_class(ruck, priorities_for(active_slot)[priority], exceptions, ignore_ammo)
  IF best EXISTS
    exceptions.add(best)                              # do not offer it again this cycle
    last_offer_time = now
    RETURN best

  IF priority + 1 < class_count
    RETURN GetNextItemInActiveSlot(priority + 1, ignore_ammo)
  IF NOT ignore_ammo
    RETURN GetNextItemInActiveSlot(0, true)           # second pass, empty weapons allowed
  exceptions = { the item currently in hand }         # cycle exhausted; reset
  RETURN none
```

**Invariants** — the exception set is what makes repeated presses walk forward instead of
returning the same weapon. It is cleared on two conditions: the cycle running out, and two
seconds of not pressing the key. The timeout is the mechanic — the player holds a mental
model of "tap repeatedly to browse, pause to commit", and the reset is what makes the next
tap start from the best weapon again rather than from wherever browsing stopped.

**Notes** — the two-second window is a feel value. Nothing derives it.

The search covers the backpack only. A weapon already in the *other* weapon slot is not
offered, which is correct: quick switch replaces what is in hand from reserves, it does not
swap the two equipped weapons.

## Ranking within a class

**Contract** — a fold over the backpack that keeps the highest-scoring weapon of one class.
Skips anything not a weapon, anything already offered this cycle, anything outside the class,
and — on the first pass — anything with an empty magazine.

```text
FUNCTION best_in_class(items, group, exceptions, ignore_ammo) -> optional<InventoryItem>
  best = none
  FOR EACH item IN items
    IF item IS NOT a weapon                  THEN CONTINUE
    IF item IN exceptions                    THEN CONTINUE
    IF item.section NOT IN group.sections    THEN CONTINUE
    IF item.loaded_rounds IS 0 AND NOT ignore_ammo THEN CONTINUE
    score = item.cost + item.loaded_rounds * ammo_weight
    IF best IS none OR score > best_score
      best = item ; best_score = score
  RETURN best
```

**Notes** — `ammo_weight` is 3: one loaded round is worth three currency units of weapon
cost. It exists so that the choice between two similar weapons goes to the loaded one, and
so that a cheap full weapon beats an expensive empty one. Its magnitude relative to shipped
weapon prices is the tuning — three units against weapon costs in the thousands means ammo
only breaks ties between near-equal weapons, which is the intended behaviour. Not derived
anywhere; a feel value.

Using **cost** as the proxy for "better weapon" is the load-bearing shortcut. There is no
authored weapon quality; price is assumed to track it, and in the shipped data it does.

## `ActivateNextItemInActiveSlot`

**Contract** — performs the switch the search chose. Refuses if nothing is in hand, or if
this inventory does not belong to the entity the player is looking through. Moves the held
weapon to the backpack, moves the chosen one into the slot, and activates it — each step
announced as an authoritative event because a client may not move items on its own
authority. Reports whether a switch happened.

```text
FUNCTION ActivateNextItemInActiveSlot() -> bool
  IF nothing is in hand                                   THEN RETURN false
  IF owner is not the entity the player is viewing through THEN RETURN false
  new_item = GetNextItemInActiveSlot(0, false)
  IF new_item IS none THEN RETURN false        # only one weapon of any usable class

  IF something is in hand
    move it to the backpack   ; emit event ITEM_TO_RUCK
  move new_item into the active slot ; emit event ITEM_TO_SLOT
  emit event ACTIVATE_SLOT
  RETURN true
```

**Invariants** — the local placement and the event are both performed, in that order, for
each of the three steps. The local move is the client's prediction; the event is the
request. If the server disagrees the placement is corrected on the next import.

**Notes** — the viewing-entity check is how the mechanic stays inert for every inventory
except the one the player is currently driving. Spectating another player, or riding in a
vehicle, silently disables the key rather than switching somebody else's weapon.

## `GetNextGrenade` / `ActivateNextGrenade` / `ActivateNextGrenadeDeffered`

**Contract** — cycles the grenade in hand to the next *type* carried. The inventory
maintains a sorted, distinct list of grenade sections it holds; this walks that list one
position from whatever is in hand, wrapping, and then finds any grenade of that section in
the backpack. Reports nothing when only one type is carried.

```text
FUNCTION GetNextGrenade() -> optional<InventoryItem>
  IF fewer than two grenade types carried THEN RETURN none
  i = index of the held item's section IN grenade_types
  next = grenade_types[(i + 1) MOD count]
  RETURN any item in the backpack whose section is next
```

**Notes** — cycling is by *type*, not by item, which is why the list of carried grenade
types exists at all. Holding five of one grenade and one of another should be a two-position
cycle, not a six-position one.

The deferred variant exists because the switch cannot happen while the current grenade is
still being put away. It sets a pending flag and asks the hands to empty; the completion of
the hide animation is what triggers the actual swap, from the inventory's per-frame update.
This is the same in-flight-switch machinery the active slot uses — see
[`Inventory.cpp`](Inventory.cpp.md) — reused for one more case.

`ActivateNextGrenade` performs the swap once the hands are free: the held grenade goes to
the backpack and the chosen one takes its base slot. Unlike the weapon path it emits no
events, because grenade cycling is single-player only and the authoritative side is in the
same process.
