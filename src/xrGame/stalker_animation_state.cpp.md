# src/xrGame/stalker_animation_state.cpp

> Resolves one body state's worth of animation names into motion handles, once, at model load.

**Needs** — [`stalker_animation_state.h`](stalker_animation_state.h.md) · [`stalker_animation_names.h`](stalker_animation_names.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`stalker_animation_state.h`](stalker_animation_state.h.md)
**Tier floor** — T2: builds a tree of handles once and then only indexes it

## Purpose

Turns fragment tables into resolved motion handles. Every name the AI will ever ask for is
built and looked up here, at load time, so that the per-frame animation selector can be
nothing but array indexing. The cost of a missing animation is therefore paid once, as an
invalid handle sitting in a slot, rather than as a failed lookup in the middle of a frame.

## State

```text
RECORD StalkerAnimationState
  global    : list<list<Motion>>        # [global_name][variant]
  torso     : list<list<list<Motion>>>  # [animation_slot][weapon_action][variant]
  movement  : list<list<list<Motion>>>  # [walk|run][direction][variant]
  in_place  : list<Motion>              # [in_place_name], owned separately
```

**Invariants**

- The innermost level of the global, torso and movement sets is a *variant list*: the same
  logical animation authored several times, so the selector can pick one at random or by
  weight. The variant list may be empty when the model does not ship that animation, and
  every consumer must tolerate that — the engine's own consumers mostly do not, which is
  why a missing animation shows up as a crash on a modded model rather than a diagnostic.
- The torso nest is built under an extra `"torso_"` name component. That component exists
  because a model's torso and leg animations share the weapon-action vocabulary and would
  otherwise collide.
- The in-place set is the only member held by reference rather than inline, and the only
  one that is deep-copied when one animation state is copied from another. Nothing else
  about the type needs that; it is a consequence of the in-place set being shared with the
  leg animator and needing a stable address.

## `Load`

**Contract** — given a model's animation interface and a base name, resolve every name in
the four sets. Each level concatenates its fragment onto the accumulated prefix and
recurses; the leaf looks the resulting string up in the model's motion bank and stores the
handle, or an invalid handle when the model has no such motion. Allocates; called once per
model; does not block.

```text
FUNCTION load(skeleton, base_name)
  global.load   (skeleton, base_name)
  torso.load    (skeleton, base_name + "torso_")
  movement.load (skeleton, base_name)
  in_place.load (skeleton, base_name)
```

**Notes** — the order of the four calls is not load-bearing; the split of the base name is.
The fact that this is four lines is the point: all the structure lives in the type, and the
loader is the same recursive walk at every level.

## Lifecycle

**Contract** — construction allocates the in-place set; copy construction deep-copies it;
destruction releases it. In a language with value semantics none of this exists — it is the
cost of the in-place set being a separately-addressed object, and a rebuild that makes it
an inline member deletes the constructor, the copy constructor and the destructor together.
