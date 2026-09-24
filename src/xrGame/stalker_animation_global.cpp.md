# src/xrGame/stalker_animation_global.cpp

> The whole-body channel: it yields to any subsystem that has claimed it, and otherwise plays a critical-wound stagger chosen by wound site and weapon, or a panic run.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_manager_impl.h`](stalker_animation_manager_impl.h.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`ai/stalker/ai_stalker_space.h`](ai/stalker/ai_stalker_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a three-branch selection and one table index

## Purpose

The second tier of the priority ladder. A global animation claims every bone group, so
choosing one is a decision to suppress the head, torso and legs entirely — which is right
for exactly three situations: another subsystem is driving the body, the stalker is
staggering from a crippling wound, or it is running in blind panic.

## `assign_global_animation`

**Contract** — return the whole-body motion for this frame, or nothing to let the trio play,
and report whether the motion needs a movement controller to carry the body along with it.

```text
FUNCTION assign_global_animation() -> (optional<motion>, wants_movement_controller)
  # 1. Another subsystem has claimed the channel. It answers for itself, including
  #    whether its motion moves the body.
  IF a global selector hook is installed
    RETURN selector()

  wants_movement_controller = false

  # 2. Not panicking: a critical wound may still take the body.
  IF mental state IS NOT panic
    RETURN global_critical_hit()

  # 3. Panicking and actually moving: the panic run.
  IF the stalker's physical speed is essentially zero
    RETURN none                       # panicking but pinned: let the trio play
  RETURN a motion from the whole-body table for the current body state, family 1
```

**Invariants** — the panic branch and the critical-wound branch are mutually exclusive by
construction: a panicking stalker never plays a wound stagger. That reads oddly until you
see what it buys — panic is a full-body flight animation and a stagger would interrupt it
every time a bullet landed, so a panicking wounded stalker keeps running.

A panicking stalker at zero speed falls back to the trio rather than playing the panic run
in place, which is the same "asked to move but pinned" case the leg channel handles: playing
a run cycle against a wall is the visible failure.

**Notes** — the hook branch returning immediately, before the movement-controller flag is even
defaulted, means a claiming subsystem owns that decision too. That is what lets a smart
cover play an authored entry animation that carries the stalker's body into the cover, which
no engine-chosen motion does.

## `global_critical_hit`

**Contract** — the stagger motion for a crippling wound, selected by wound site and by the
weapon in hand. Returns nothing unless the stalker is critically wounded. Once a stagger is
playing it keeps playing: the same motion is returned until it ends. Hard-fails if a
critically wounded stalker has no weapon.

```text
FUNCTION global_critical_hit() -> optional<motion>
  IF NOT critically wounded          RETURN none
  IF the global channel already has an animation
    RETURN that same animation        # do not restart or reselect mid-stagger

  weapon = the item in hand                   # must be a weapon
  slot   = weapon.animation slot              # 1, 2 or 3
  family = wound_type + 6 * (slot - 1)
  RETURN a weighted choice among the whole-body STANDING motions of that family,
         using the stalker's per-wound weights
```

**Invariants**

- The stagger is always selected from the **standing** body state, never the crouching one,
  regardless of the stalker's actual posture. A crippling wound stands the stalker up; there
  is no crouching stagger authored.
- The already-playing check is what makes this stable. Without it the weighted choice would
  be re-rolled every frame and the stalker would flicker between staggers.
- The family index packs two axes into one flat table: six wound sites — head, torso, each
  hand, each leg — times three weapon slots. A weapon slot outside one to three indexes past
  the table, which is why it is checked.

**Notes** — the weights are per-stalker rather than global, which is how a particular
character can be made to favour one reaction; they come from its configuration section.

The six-wide stride is the count of wound sites and is therefore fixed by the wound
vocabulary rather than chosen. A rebuild should index by the pair rather than by the packed
sum, which makes the stride disappear.

## `global_play_callback`

**Contract** — the renderer's notification that the global channel's motion ended. Tells the
channel, and raises the deferred-callback flag if a subsystem has registered interest.
Called from inside the skeleton's blend update, so it must not do work.

**Invariants** — it only sets a flag. The actual callback runs at the top of the next frame's
update, by which time the channel set is no longer being mutated. See
[`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md).

**Notes** — the callback receives the stalker through the blend's opaque parameter, which is
the renderer's only channel back to the game layer. That is the problem being solved: the
pose is computed inside the renderer, and the game layer has to be told about an event the
renderer noticed.
