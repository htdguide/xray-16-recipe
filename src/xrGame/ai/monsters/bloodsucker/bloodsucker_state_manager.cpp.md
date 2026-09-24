# src/xrGame/ai/monsters/bloodsucker/bloodsucker_state_manager.cpp

> The bloodsucker's brain: nine registered states, a selector that puts feeding above everything else a live enemy could provoke, and a hard bypass that hands the creature to a scripted seize whenever one is running.

**Needs** — [`bloodsucker_state_manager.h`](bloodsucker_state_manager.h.md) · [`bloodsucker.h`](bloodsucker.h.md) · [`bloodsucker_vampire.h`](bloodsucker_vampire.h.md) · [`bloodsucker_predator.h`](bloodsucker_predator.h.md) · [`bloodsucker_state_capture_jump.h`](bloodsucker_state_capture_jump.h.md) · [`bloodsucker_attack_state.h`](bloodsucker_attack_state.h.md) · [`monster_state_attack.h`](../states/monster_state_attack.h.md) · [`monster_state_rest.h`](../states/monster_state_rest.h.md) · [`monster_state_panic.h`](../states/monster_state_panic.h.md) · [`monster_state_eat.h`](../states/monster_state_eat.h.md) · [`monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`bloodsucker_state_manager.h`](bloodsucker_state_manager.h.md)
**Tier floor** — T2: a priority selector over perception facts, plus one call into the physics layer to attach a body to a bone

## Purpose

Every creature's brain is the same machine — a composite state owning global states and re-picking one each update — and every creature differs in which states it owns and in what order it prefers them. This file is the bloodsucker's answer, and it differs from the plainer creatures in two ways.

First, the selector is *two-level*: before any of the usual reasoning runs, the brain asks whether a scripted seize-and-drag owns the creature, and if so it hands over entirely. Second, feeding is checked ahead of the danger assessment, so a bloodsucker that can feed will feed even on an enemy it would otherwise flee from.

## State

The brain owns no data of its own. It holds nine registered global states and the active/previous pair every composite state has.

| Global state | Implementation | Note |
|---|---|---|
| custom | [seize-and-leap hold](bloodsucker_state_capture_jump_inline.h.md) | also the script-forced slot |
| rest | generic | |
| panic | generic | chosen when the enemy is assessed as strong |
| attack | generic | **not** the creature's own attack composite — see Notes |
| eat | generic | |
| heard an interesting sound | generic | |
| heard a dangerous sound | generic | |
| was hit | generic | |
| feed | [the vampire tree](bloodsucker_vampire_inline.h.md) | registered here as a *global* state, not only as an attack substate |

**Notes** — registering the feed at the top level rather than inside the attack composite is what lets it outrank the danger assessment. If it were a substate of attack, a bloodsucker facing a strong enemy would panic and never get the chance.

The line that would register the creature's own attack composite is commented out beside the generic one. Everything reachable only through it — [`bloodsucker_attack_state_inline.h`](bloodsucker_attack_state_inline.h.md), [`bloodsucker_attack_state_hide_inline.h`](bloodsucker_attack_state_hide_inline.h.md), [`bloodsucker_predator_lite_inline.h`](bloodsucker_predator_lite_inline.h.md) and the back-approach leaf — is therefore dead in the shipped build. That is the single largest piece of unreachable behaviour in this chapter, and it is unreachable by an edit somebody made on purpose, not by an oversight.

## `execute` — the selector

**Contract** — choose exactly one global state, switch into it if it differs from the active one, execute it, and record it. Two disjoint paths, decided before any perception is read.

**Invariants** — on the ordinary path exactly one state is always selected; there is no route that leaves the brain without an active state. On the scripted path the brain may deliberately do *nothing at all* — see the Notes.

```text
FUNCTION execute()
  IF a drag jump is pending OR a scripted animation owns the creature
    IF a drag jump is pending
      select(custom) ; custom.execute() ; previous = custom
      seize_corpse()
    # otherwise: no state is selected and nothing runs this update
    RETURN

  IF an enemy is known
    IF check_feeding()             -> feed
    ELSE IF the enemy is assessed strong -> panic
    ELSE IF the enemy is assessed weak   -> attack
  ELSE IF hit memory is fresh            -> was_hit
  ELSE IF heard an interesting sound     -> hear_interesting
  ELSE IF heard a dangerous sound        -> hear_danger
  ELSE IF a usable corpse is near        -> eat
  ELSE                                   -> rest

  select(chosen) ; chosen.execute() ; previous = chosen
```

**Notes** — the "scripted animation owns the creature but no drag jump is pending" case leaves the selection untouched: no state is chosen and no state is executed. That is correct rather than sloppy. A scripted animation is already driving the body, and the previously active state keeps its place so that the brain resumes where it left off when the animation ends.

The danger assessment has no default branch. If the enemy is assessed as neither strong nor weak, no state is chosen this update and the previously active one simply continues. The assessment is a two-valued judgement in practice, so the gap is unreachable; a rebuild with a third value must decide what it means.

The corpse branch consults both "is a corpse near" and "will the eat state accept it", so a bloodsucker does not walk to a body it cannot use.

A disabled pair of branches beside the selector would have started and stopped the stalking cloak around the "heard an interesting sound" state, making a curious bloodsucker fade out while it investigated. It was removed; the creature now investigates visible.

## `check_feeding`

**Contract** — answers whether the feed should be selected, using the same latch shape as every other "should I switch to this" test in the chapter: if we are not already in it, ask whether it will start; if we are, ask whether it has *not* finished.

```text
FUNCTION check_feeding() -> bool
  IF previous is not feed  RETURN feed.will_start()
  RETURN NOT feed.is_complete()
```

**Invariant** — the latch is what stops a completed feed being re-entered on the very next update. Without it the feed's own start test, which is about opportunity rather than recency, would fire again immediately.

## `seize_corpse`

**Contract** — attach a captured entity to the creature so it can be dragged: look up the drag bone on the victim's skeleton by the name the creature carries, claim the body from the pack so no packmate tries to eat it, ask the physics layer to capture the body at that bone, and start the drag animation if the capture succeeded. Does nothing when there is no captured entity, when the victim has no skeleton, or when the capture failed.

```text
FUNCTION seize_corpse()
  victim = the entity the creature captured
  IF victim is none  RETURN
  IF victim has no skeleton  RETURN

  IF the creature is in a pack
    claim victim as this creature's corpse

  bone = victim.skeleton.bone_named(creature.drag_bone_name)
  physics.capture(victim, bone)

  IF the capture succeeded AND a scripted animation is running
    start the drag animation
```

**Notes** — the pack claim is what keeps a dragged body from being the target of another creature's eat state; it is taken before the capture, so a failed capture still leaves the body claimed. That asymmetry is in the source and is probably an oversight, but it is harmless: the claim is released when the creature's own bookkeeping next runs.

The bone is named by a string carried on the creature and set from the scripted encounter that created the drag jump. A body whose skeleton lacks that bone cannot be dragged, and the failure is silent.

## `update`

**Contract** — the scheduled tick. A plain delegation to the shared creature brain, present only because the declaration overrides it.
