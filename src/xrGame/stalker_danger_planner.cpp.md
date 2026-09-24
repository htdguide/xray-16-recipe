# src/xrGame/stalker_danger_planner.cpp

> The danger branch: one proposition per kind of threat, one sub-planner per kind, and the two reactions that run whatever kind it is.

**Needs** — [`stalker_danger_planner.h`](stalker_danger_planner.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_danger_unknown_planner.h`](stalker_danger_unknown_planner.h.md) · [`stalker_danger_in_direction_planner.h`](stalker_danger_in_direction_planner.h.md) · [`stalker_danger_grenade_planner.h`](stalker_danger_grenade_planner.h.md) · [`stalker_danger_by_sound_planner.h`](stalker_danger_by_sound_planner.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`sound_player.h`](sound_player.h.md)
**Used by** — [`stalker_danger_planner.h`](stalker_danger_planner.h.md)
**Tier floor** — T2: a four-operator dispatch rebuilt on setup.

## Purpose

A stalker reacts differently to a ricochet, to an incoming direction of fire, to a live
grenade, and to a suspicious sound. This planner is the dispatcher: its evaluators classify
the currently selected danger, and each of its four operators is the sub-planner for one
class. It owns nothing of the reactions themselves.

The design point worth carrying over is that the classification happens **once, here**,
rather than inside each reaction. A sub-planner never has to ask "is this danger mine"; the
precondition already guaranteed it.

## State

`Stateless.` The selected danger lives in the creature's danger memory; this planner reads
it through evaluators and never caches it, because the selection can change between cycles
and a cached copy would send the creature into the wrong reaction.

## `setup`

**Contract** — bind to the creature and the parent's storage, then discard and rebuild both
tables. Idempotent, like every planner's setup.

## `add_evaluators`

**Contract** — install five questions: one asking whether there is any danger at all, and
four classifying it.

```text
Danger             : is a danger currently selected
DangerUnknown      : ... and it is of a kind with no known direction
DangerInDirection  : ... and it has a known direction
DangerGrenade      : ... and it is a live grenade
DangerBySound      : ... and it is a sound
```

**Notes** — the classification is a partition over the danger's *type*, and the types are
defined by the danger memory rather than here; see
[`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) for
which type falls in which class and for the one class that is currently switched off.

## `add_actions`

**Contract** — install four sub-planners, each conditioned on its own class and each
claiming to clear the top-level danger proposition.

```text
DangerUnknownPlanner      requires DangerUnknown     = true   effects Danger = false
DangerInDirectionPlanner  requires DangerInDirection = true   effects Danger = false
DangerGrenadePlanner      requires DangerGrenade     = true   effects Danger = false
DangerBySoundPlanner      requires DangerBySound     = true   effects Danger = false
```

**Invariants** — the four preconditions are mutually exclusive by construction of the
evaluators, so exactly one branch is ever available. That is why none of them needs a
negative precondition against the others, and why the order they are added in does not
matter.

All four claim the same effect. The parent asked for `Danger = false`; each sub-planner
reaches it its own way, and the planner picks whichever is currently *possible* rather than
whichever is cheapest — there is only ever one.

## `initialize`

**Contract** — on entering the danger branch: silence everything but the low-level humming
channel, and **release the creature's claim on its cover point** with the squad
coordinator.

**Invariants** — dropping the cover claim is the load-bearing half. Cover points are a
squad-wide scarce resource; a creature entering a danger reaction is about to pick a new
one appropriate to the new threat, and holding the old claim would keep a point reserved
that nobody is standing at. The release happens at branch entry rather than at cover
selection because the gap between the two is where an ally could otherwise not take it.

## `update`

**Contract** — run one planning-and-execution cycle, then run two reactions that belong to
the branch as a whole rather than to any one sub-planner: reacting to grenades in flight,
and reacting to a squad member's death.

```text
FUNCTION update()
  base.update()
  creature.react_on_grenades()
  creature.react_on_member_death()
```

**Invariants** — both run *after* the cycle and unconditionally, on every cycle, for every
danger class. They are the paths by which a new danger enters the danger memory while the
creature is already reacting to an old one — a grenade landing while you are investigating
a ricochet must be noticed — so they must not be gated on which sub-planner is running.

**Notes** — putting these here rather than in the creature's own per-frame update means a
stalker only watches for thrown grenades and only notices its comrades dying **while it is
already alarmed**. That is a deliberate perception economy and it is visible in play: a
stalker at ease walks past a grenade. A rebuild that hoists them into the general update
makes creatures noticeably more alert than the original.

## `finalize`

**Contract** — on leaving the danger branch, if the creature is alive and has an enemy,
**stamp the danger memory's time line to now**.

**Invariants** — the stamp invalidates every recorded danger older than this moment. The
condition is the point: it applies only when an enemy is selected, i.e. when the creature
is leaving danger reaction *for combat*. A creature entering a firefight must not spend the
firefight re-reacting to the ricochets and deaths that led up to it, so the whole danger
history is retired at the boundary. Leaving the branch for any other reason keeps the
history, because it may still be the best thing the creature knows.
