# src/xrGame/ai/stalker/ai_stalker_feel.cpp

> The perception filters: what a stalker's eyes bother to consider, what its touch accepts, and the mutual consent a touch contact requires.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`visual_memory_manager.h`](../../visual_memory_manager.h.md) · [`sight_manager.h`](../../sight_manager.h.md) · [`stalker_movement_manager_smart_cover.h`](../../stalker_movement_manager_smart_cover.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: filter predicates

## Purpose

Perception in this engine is a shared machine — a frustum, a range, a raycast against the
collision database — and each entity narrows it with a few predicates. This file is the
stalker's narrowing, and it is tiny, which is the finding: **a stalker's vision filter is
two type tests.**

## `feel_vision_isRelevant`

**Contract** — decides whether an object is worth running the expensive visibility test
against. A dead stalker sees nothing. A living one considers exactly two kinds of thing:
living entities and inventory items. Everything else in the world — doors, vehicles, debris,
anomaly volumes — is invisible to a stalker's eyes.

```text
FUNCTION worth_looking_at(object) -> bool
  IF dead THEN RETURN false
  RETURN object is a living entity OR object is an inventory item
```

**Invariants** — this is a *pre-filter*, not the visibility test. Everything it admits still
goes through the frustum, the range, the transparency accumulation and the raycast. Its only
job is to keep the per-frame candidate set small, and a rebuild should keep a filter of the
same cheapness — the set it runs over is every spatial object near the stalker.

**Notes** — a disabled line would have made a stalker ignore members of its own team
entirely. It is not compiled, and it must not be: the squad telepathy in
[`ai_stalker_misc.cpp`](ai_stalker_misc.cpp.md) depends on a stalker being able to *see* its
squadmates in order to inherit their enemy. Re-enabling that line would silently break squad
coordination.

## `feel_touch_contact` — may I notice this at all

**Contract** — the first half of touch acceptance. Refuses inventory items outright when the
stalker's take-items switch is off (a script flag used for characters who must not loot),
refuses itself, then defers to the base rule, and finally asks the *other object* whether it
consents.

```text
FUNCTION feel_touch_contact(object) -> bool
  IF take-items is disabled AND object is an inventory item THEN RETURN false
  IF object is me                                           THEN RETURN false
  IF the base rule refuses                                  THEN RETURN false
  RETURN object.accepts_touch_from(me)
```

**Invariants** — touch is **mutual consent**: both sides must agree before a contact is
registered. That is what lets an object exclude itself from being touched by AI without the
AI knowing anything about it.

## `feel_touch_on_contact` — do I consent to being touched

**Contract** — the other half, from the stalker's side. A stalker consents to touch only
from objects flagged visible to AI, then defers to the base rule. Symmetric with the test
in [`ai_stalker_events.cpp`](ai_stalker_events.cpp.md)'s pick-up reflex, so that an object
the stalker would refuse to notice cannot enter its touch set by the other direction either.

## `feel_touch_delete`

**Contract** — when an object leaves touch range, remove it from the ignored list. This is
the other half of the pick-up reflex's bookkeeping: an item the stalker decided against is
reconsidered the next time it comes back into range, because the decision was recorded as
"ignored while in range", not "ignored forever".

## `bfCheckForNodeVisibility`

**Contract** — asks the visual memory whether a given navigation cell is within the
stalker's field of view, using the *head's current yaw* rather than the body's. Used by the
planner's positional evaluators to ask "can I see that spot from here". Note it tests the
field of view only, not occlusion.

## `Exec_Look`

**Contract** — a one-line delegation to the sight manager, called once per frame from the
frame update. It exists because the base entity declares looking as a virtual step and the
stalker's looking lives in a separate manager.

## `renderable_Render`

**Contract** — chains the base entity's and the inventory owner's rendering, the second of
which is what draws the weapon in the stalker's hands. In debug builds it also emits
animation statistics when the corresponding diagnostic flag is set.
