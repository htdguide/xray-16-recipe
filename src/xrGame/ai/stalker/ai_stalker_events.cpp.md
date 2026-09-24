# src/xrGame/ai/stalker/ai_stalker_events.cpp

> Inventory events and the pick-up reflex: what happens when an item is given to a stalker, taken from it, or merely walked past.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`Inventory.h`](../../Inventory.h.md) · [`xrServerEntities/xrMessages.h`](../../../xrServerEntities/xrMessages.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: event handling plus one authoritative round trip

## Purpose

Ownership of an item is *authoritative state*, not local state: it lives in the server-side
record even in single player, where both sides run in one process. So a stalker never simply
takes something. It **asks**, by emitting an event, and the take happens when the event comes
back. This file is both ends of that loop, and getting the loop right is what keeps the
server's picture of who owns what from drifting from the client's.

The second idea here is the **ignored-touched-objects list**, which is how a stalker
remembers that it has already decided not to pick something up.

## `OnEvent` — take, drop, buy, sell

**Contract** — handles the same four events as the trader, with one difference that matters:
a rejected take is *sent back* rather than dropped silently.

```text
FUNCTION on_event(packet, type)
  base.on_event(...) ; inventory_owner.on_event(...)

  SELECT type
    trade_buy, ownership_take:
      item = find_object(packet.read_id())      # required to exist
      IF inventory can take it
        reparent it to me and take it
        # if I now hold nothing and a script is driving me and the thing is a
        # weapon, tell the weapon-handling planner to idle WITH that item -
        # otherwise a scripted stalker handed a rifle stands there empty-handed
        IF no active item AND under script control AND it is a shooting object
          weapon_planner.set_goal(idle, item)
        run the post-take conflict resolution
      ELSE
        # send an ownership rejection so the authority's record is corrected.
        # Without this the server believes the stalker holds an item it does not.
        send ownership_reject naming the item

    trade_sell, ownership_reject:
      item = find_object(packet.read_id())
      IF the item no longer exists THEN BREAK    # see Notes
      just_before_destroy = packet has more AND packet.read_flag()
      dont_create_shell   = (type == trade_sell) OR just_before_destroy
      mark the item pre-destroy
      on_ownership_reject(item, dont_create_shell)
```

**Notes** — the missing-object case in the sell branch carries a standing question in the
source about how it can happen at all. It is guarded rather than explained. A rebuilder
should guard it too and should not assume the object is always present.

## `on_ownership_reject` — letting go of something

**Contract** — drops an item, and takes three steps before doing so that are not obvious.

```text
FUNCTION on_ownership_reject(item, just_before_destroy)
  physics.frame_update()
  invalidate the skeleton's bone transforms
  recompute the bones, forced

  # the three steps above exist because the dropped item's position and
  # orientation are derived from the HAND BONE. Dropping on a stale pose puts
  # the item where the hand was last frame, which is visibly wrong when the
  # stalker is moving, and can put it inside geometry.

  IF the inventory refuses to drop it THEN RETURN
  IF the item was destroyed by the drop THEN RETURN

  # refuse to notice this item again for two seconds, so that the touch reflex
  # below does not immediately pick up what was just put down
  deny touch on it for 2000 milliseconds
```

**Invariants** — the two-second touch denial is what breaks the obvious loop: drop item,
touch item, take item, drop item. A rebuild that omits it will have stalkers juggling.

## `feel_touch_new` — the pick-up reflex

**Contract** — called when an object first enters the stalker's touch range. Decides,
once, whether to want it.

```text
FUNCTION feel_touch_new(object)
  IF dead, or not the authority                     THEN RETURN
  IF the object is not flagged VISIBLE-TO-AI        THEN RETURN
      # objects can opt out of being noticed; phantoms do exactly this

  IF NOT wounded AND NOT critically wounded
     AND it is an inventory item flagged useful to NPCs
     AND I am allowed to take it
    send an ownership_take event
    RETURN

  # everything else goes on the ignored list, so that the perception layer
  # does not re-ask about it every time it re-enters range
  record it in the ignored-touched-objects list
```

**Invariants**

- A wounded or crippled stalker picks up nothing. That is what stops a downed stalker from
  looting while it lies there.
- The ignored list is a positive record of "I decided against this", and
  [`ai_stalker_feel.cpp`](ai_stalker_feel.cpp.md) removes entries from it when the object
  leaves touch range. A rebuild must pair the two or the list grows without bound and the
  stalker never reconsiders an item whose usefulness changed.
- The take is *asked for*, not performed. The item is not in the inventory when this returns.

## `generate_take_event` / `DropItemSendMessage`

**Contract** — the two one-line emitters. Taking names the item and the stalker; dropping
first checks that the stalker is in fact the item's parent, so that a script cannot make a
stalker drop something it does not hold.

## `UpdateAvailableDialogs`

**Contract** — a pure delegation to the dialogue mixin, present so that the stalker's
override chain is complete.

## Notes

**Every diagnostic in this file is compiled out** by a silence switch defined at the top.
Six messages — trying to take, took, could not take, dropping — exist behind it. The
equivalent handler on the trader has no such switch and logs unconditionally; the asymmetry
is worth knowing if a rebuild is comparing the two.
