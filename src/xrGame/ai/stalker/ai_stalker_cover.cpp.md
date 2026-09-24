# src/xrGame/ai/stalker/ai_stalker_cover.cpp

> Cover selection and its cache: a two-radius search whose acceptable distance band is chosen by the weapon in hand, and an invalidation set that is most of the behaviour.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`ai_stalker_space.h`](ai_stalker_space.h.md) · [`cover_evaluators.h`](../../cover_evaluators.h.md) · [`cover_manager.h`](../../cover_manager.h.md) · [`cover_point.h`](../../cover_point.h.md) · [`smart_cover.h`](../../smart_cover.h.md) · [`stalker_movement_restriction.h`](../../stalker_movement_restriction.h.md) · [`agent_member_manager.h`](../../agent_member_manager.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a cached search plus an invalidation set

## Purpose

"Take cover" is the most visible thing a stalker does, and this file is where it is decided.
Three ideas carry it.

**The weapon decides the range band.** A stalker does not look for the safest place; it
looks for the safest place *at a distance its weapon works at*. A shotgunner wants to be
within five metres of the enemy and will take a worse position to get there; a sniper
refuses anything closer than twenty; a pistol holder wants ten. Everyone else works at
twenty. That one table is why a mixed squad spreads out around a firefight instead of
piling into the same wall.

**The search is two concentric attempts.** Ten metres first, thirty if that fails. A stalker
prefers a mediocre cover it can reach in a moment to a better one across the street, and
only widens the net when it has no choice.

**The answer is cached, and the invalidation set is the real design.** Searching is
expensive, so the chosen cover is kept and only re-examined when something specific happens.
Six events invalidate it, and each one corresponds to a way the world can make a cover
wrong. Getting this set right is what makes a stalker look like it is holding a position
rather than twitching.

## State

The cache lives on the stalker; see [`ai_stalker.h`](ai_stalker.h.md). One module-level
constant is shared with other files:

```text
CONSTANT minimum_suitable_enemy_distance = 3 metres
```

It is the floor of every range band *and* the radius inside which a cover is abandoned
because the enemy has walked up to it.

## `compute_enemy_distances`

**Contract** — produces the acceptable distance band to the thing being taken cover from,
from the class of the best weapon in hand. Pure.

```text
FUNCTION compute_enemy_distances() -> (minimum, maximum)
  minimum = 3 metres
  maximum = 170 metres
  IF there is no best weapon THEN RETURN      # unarmed: any distance will do

  SELECT weapon_class(best_weapon)
    pistol:       maximum = 10
    shotgun:      maximum = 5
    sniper_rifle: minimum = 20                # maximum stays at 170
    otherwise:    maximum = 20
  minimum = min(minimum, maximum)             # a shotgunner's band collapses to 3..5
```

**Invariants** — the sniper case sets only the floor, so a sniper's band is 20 to 170
metres: it will accept almost anything as long as it is not close. Every other class sets
only the ceiling. The clamp that follows exists for the shotgun case, whose ceiling of five
is above the floor of three but below what the other classes assume.

**Notes** — a second clamp follows the first and is a no-op given it. Harmless.

## `find_best_cover`

**Contract** — the search. Decides whether smart covers are admissible, configures the
general-purpose cover evaluator with the band, then asks the cover registry for the best
cover within ten metres of the stalker, subject to the stalker's movement restrictions. If
that finds nothing, repeats at thirty metres. Returns nothing when both fail.

```text
FUNCTION find_best_cover(cover_from) -> optional<Cover>
  (near, far) = compute_enemy_distances()

  # smart covers - the authored, animated ones with named firing loopholes - are
  # only usable with a weapon in the primary weapon slot. A stalker holding a
  # pistol or a grenade is restricted to ordinary navigation-mesh cover, because
  # the smart cover's animations assume a rifle.
  evaluator.allow_smart_covers = (best weapon exists
                                  AND it is a weapon
                                  AND its base slot is the primary weapon slot)

  evaluator.setup(cover_from, near, far, near)
  cover = cover_registry.best_cover(own position, radius = 10, evaluator, restrictions)
  IF cover EXISTS THEN RETURN cover

  evaluator.setup(cover_from, near, far, near)
  RETURN cover_registry.best_cover(own position, radius = 30, evaluator, restrictions)
```

## `best_cover` — the cached entry point

**Contract** — the only function callers use. Re-examines the cached answer, returns it if
still good, otherwise searches afresh. Always publishes the result to the squad's member
registry, so that other squad members can avoid claiming the same cover.

```text
FUNCTION best_cover(cover_from) -> optional<Cover>
  update_best_cover_actuality(cover_from)
  IF cache is still valid
    publish cached cover to the squad
    RETURN cached cover

  cache is now valid
  new_cover = find_best_cover(cover_from)
  IF new_cover != cached cover
    notify subscribers (new, old)
    cached cover = new_cover
    clear the advance-search marker
  cached value = new_cover EXISTS ? evaluate(new_cover) : infinity
  publish cached cover to the squad
  RETURN cached cover
```

## `update_best_cover_actuality` — when a cover stops being good

**Contract** — the re-examination. Four tests, in order; any failure invalidates the cache.
When all four pass, it then performs an *advance* search.

```text
FUNCTION update_best_cover_actuality(cover_from)
  IF cache already invalid THEN RETURN
  IF there is no cached cover THEN invalidate ; RETURN

  # a smart cover is only as good as the loophole it offers against this threat;
  # a threat that has moved so that no loophole bears on it makes the whole
  # smart cover worthless even though the cover itself has not changed
  IF the cached cover is a smart cover
    IF it has no loophole bearing on `cover_from` THEN invalidate ; RETURN

  # the enemy has walked up to my cover: it is no longer cover
  IF distance(cached cover, cover_from) < 3 metres THEN invalidate ; RETURN

  # the cover's score has degraded by more than one unit since it was chosen.
  # The threshold is ABSOLUTE, not relative, which means a good cover tolerates
  # proportionally less degradation than a poor one.
  IF evaluate(cached cover) >= cached value + 1 THEN invalidate ; RETURN

  # -- the advance search ------------------------------------------------
  IF the advance marker already names this cover THEN RETURN
  mark this cover as advanced-from
  re-run the ten-metre search with the WIDEST band (3 to 170 metres) and adopt
  whatever it returns as the cached cover
```

**Invariants**

- The advance search runs **once per cover**, not once per tick: the marker is compared
  against the current cover, so a stalker that keeps the same cover does the advance search
  exactly once. That bound is what makes an otherwise-expensive extra search affordable.
- The advance search uses the widest possible band rather than the weapon's band. It is
  asking "is there somewhere better *at all*", which is how a stalker creeps forward from
  cover to cover across a firefight rather than settling permanently.
- It replaces the cached cover **without notifying the subscribers and without recomputing
  the cached value.** A rebuild reproducing this will have the same consequence: the stalker
  moves to a new cover while everything listening for a cover change still believes it is at
  the old one, and the degradation test that follows compares the new cover against the old
  one's score.

**Notes** — the advance search is meant to be gated by a "may I try to advance" flag that
callers set. **That gate is disabled**: the test that would have returned early is written
as an unconditional false, so the advance search runs on every re-examination that survives
the four tests, and the flag-setting entry point is inert. The stalker therefore advances
more eagerly than the design intended. Whether the gate was disabled for a reason is not
recoverable, but the shipped behaviour is the ungated one and a rebuild must match it.

## `best_cover_value`

**Contract** — scores the currently cached cover against a threat position using the widest
band, so that the degradation test is comparing like with like across weapon changes.
Requires a cached cover.

## The invalidation hooks

**Contract** — six events drop the cache, each for its own reason:

| Event | Why the cover is now suspect |
|---|---|
| the enemy changed | the whole threat geometry changed; also drops the item cache |
| the restrictions changed | the cover may no longer be reachable or permitted |
| a danger location was added within its radius of the cover | something is now dangerous there |
| a danger location was removed within its radius | the position that was forbidden may now be the best |
| the cover was blocked | somebody else claimed it |
| an explicit invalidate | script or another subsystem knows better |

The danger-removal hook has a second branch: when the stalker has *no* cover, it invalidates
if the removed danger was within its radius of the **stalker's own position** rather than of
a cover — because a stalker with no cover was probably refused one by that very danger.

## `subscribe_on_best_cover_changed` / `unsubscribe` / `on_best_cover_changed`

**Contract** — a list of callbacks notified when the cached cover is replaced, with the new
and the old cover. Subscription refuses duplicates and unsubscription requires membership.
The notification is what lets the smart-cover animation planner and the movement manager
react to a cover change without polling.

## `use_smart_covers_only`

**Contract** — a switch on the general-purpose evaluator that restricts it to authored smart
covers, ignoring ordinary navigation-mesh cover entirely. Set from script for set-piece
encounters where the designer wants stalkers using specific authored positions.
