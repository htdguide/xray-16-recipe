# src/xrGame/stalker_combat_planner.cpp

> Twenty operators over twenty-five propositions: the whole of how a stalker fights, expressed as preconditions.

**Needs** — [`stalker_combat_planner.h`](stalker_combat_planner.h.md) · [`stalker_combat_actions.h`](stalker_combat_actions.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_kill_wounded_planner.h`](stalker_kill_wounded_planner.h.md) · [`stalker_get_distance_planner.h`](stalker_get_distance_planner.h.md) · [`stalker_low_cover_planner.h`](stalker_low_cover_planner.h.md) · [`stalker_search_planner.h`](stalker_search_planner.h.md) · [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) · [`stalker_planner.h`](stalker_planner.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_point.h`](cover_point.h.md) · [`Inventory.h`](Inventory.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`sound_player.h`](sound_player.h.md)
**Used by** — [`stalker_combat_planner.h`](stalker_combat_planner.h.md)
**Tier floor** — T2: a twenty-operator search per creature per cycle, plus save and load.

## Purpose

The largest planner in the engine and the reason the planner architecture was chosen at all.
Twenty operators — eleven leaf actions, four sub-planners, and three operators sharing one
action class — over a world state of about twenty-five propositions. Nothing sequences them.
The order a stalker does things in a firefight is entirely a consequence of which
preconditions hold.

Reading it as a state machine will not work. Read it instead as: *what must be true for this
to be worth doing*, one operator at a time. The priority structure is then visible in the
preconditions every operator shares.

## State

```text
RECORD CombatPlanner
  last_enemy_id    : optional<EntityId>   # written, never read
  last_level_time  : int                  # written, never read
  last_wounded     : bool                 # handed by reference to the enemy evaluator
```

**Invariants** — only the wounded flag is live, and it is live by *reference*: the enemy
evaluator holds a pointer to it so that it can distinguish "the enemy is gone" from "the
enemy went down wounded", which changes how long the post-combat delay runs. The other two
fields are dead; a rebuild drops them. The code that would have maintained the wounded flag
each cycle is commented out, so the flag is only ever false — which is a live defect, not a
design: the evaluator's wounded branch is unreachable.

## The shared preconditions

Almost every operator carries the same three or four negative preconditions, and they are
the priority order of the whole branch:

```text
CriticallyWounded = false   on 19 of 20 operators
DangerGrenade     = false   on 17
UseSuddenness     = false   on 14
EnemyWounded      = false   on 13
Panic             = false   on 8
InSmartCover      = false   on 9
LowCover          = false   on 7
```

Read downward, that is: being critically wounded overrides everything; a live grenade
overrides everything else; an unnoticed approach overrides ordinary combat; a downed enemy
overrides ordinary combat; panic disables the deliberate behaviours; and being inside a
smart cover or at low cover routes to the sub-planner for that instead of the general loop.

A rebuild does not need to reproduce the repetition — this is exactly what an operator
*group* with shared preconditions would express — but it must reproduce the ordering, because
this is where it lives. There is no priority number anywhere in the combat branch.

## `setup`

**Contract** — bind, seed eight propositions, rebuild the tables, subscribe to the creature's
cover-change notification, and hand the movement layer the *parent's* property storage.

```text
FUNCTION setup(creature, parent_storage)
  base.setup(creature, parent_storage)
  inner.InCover = false;  inner.LookedOut = false
  inner.PositionHolded = false;  inner.EnemyDetoured = false
  inner.UseSuddenness = true;    inner.UseCrouchToLookOut = true
  inner.KilledWounded = false;   inner.StartedToThrowGrenade = false
  the brain's own storage: CriticallyWounded = false
  clear evaluators and operators
  add_evaluators();  add_actions()
  subscribe on_best_cover_changed
  movement.property_storage := parent_storage
```

**Invariants** — `UseSuddenness` is seeded **true**. A creature that has not yet been seen
begins every fight with the ambush branch available, and it is cleared the moment anything
proves otherwise. Seeding it false would delete the sneak-up behaviour from the game.

Handing the movement layer a property storage is the channel by which movement decisions —
smart-cover entry and exit especially — can read and write planner propositions directly. It
is the parent's storage rather than this planner's, because the movement layer outlives any
one combat engagement.

## `on_best_cover_changed`

**Contract** — invoked when the creature's chosen cover point changes. Clears all four
cover-sequence propositions.

**Invariants** — this is the branch's *invalidation* mechanism, and it is the reason the
sequence does not need to detect staleness itself. Moving the goalposts resets the sequence:
a creature that had taken cover, looked out and held position, and whose cover point is then
taken by an ally or invalidated by the enemy moving, starts the sequence again from
take-cover rather than flanking from a point it no longer holds.

Subscribing to a notification rather than polling is what makes that instantaneous. A
rebuild that polls will have creatures completing a sequence step against a point they have
already lost.

## `initialize`

**Contract** — branch entry. Seeds the sequence propositions *unless* this planner was just
restored from a save, releases the cover claim, clears the path, enables the trigger-finger
reflex, decides whether an ambush is possible, registers with the squad as in combat, and
sounds the alarm.

```text
FUNCTION initialize()
  base.initialize()
  IF NOT restored_from_save THEN
    clear InCover, LookedOut, PositionHolded, EnemyDetoured
    UseSuddenness := true;  KilledWounded := false
    the brain's storage: CriticallyWounded = false
  StartedToThrowGrenade := false

  release my cover claim
  movement.clear_path()                     # see Notes
  enable the clutched-trigger reflex        # see stalker_death_actions.cpp

  IF NOT restored_from_save AND an enemy is selected THEN
    UseSuddenness := NOT (the enemy can see me right now)

  restored_from_save := false

  IF the squad already has members in combat THEN UseSuddenness := false

  IF the enemy is visible now AND human AND I may speak
     AND my squad has more than one member AND I am not sneaking THEN
    play the alarm line
  squad.register_in_combat(me)
```

**Invariants** — the restore guard is the whole reason this planner is serialized. A creature
saved mid-firefight must resume with its sequence propositions as they were, or loading a
save would visibly reset every stalker's tactical state.

**Invariants** — the ambush is cancelled if *any* squad member is already fighting. One
member being in contact means the enemy is alerted, so nobody in that squad is sneaking up on
anything.

**Notes** — clearing the path at combat entry is marked in the original as a workaround. The
path the creature was following was built with the relaxed movement speeds; entering combat
changes them, and a half-executed path built under the old speeds produces a visible stutter
while the new one is computed. Clearing is the cheap fix; the honest fix is for the path to
carry the speeds it was built with.

The alarm line's five conditions are the usual anti-cacophony gating, plus one that matters:
a creature that is still sneaking does not shout.

## `update`

**Contract** — run one cycle, then react to grenades in flight and to squad members dying.

**Notes** — these are the same two reactions the danger planner runs, for the same reason:
they are how a *new* danger reaches a creature that is already busy, and they must not be
gated on which action is running. A creature is watchful for grenades exactly while it is
fighting or already alarmed, and not otherwise.

## `finalize`

**Contract** — branch exit. If alive: push the danger memory's time line **three seconds into
the future**, unregister from the squad's combat roster, disable the trigger-finger reflex,
and idle the sidearm if that is what is in hand.

**Invariants** — stamping the time line *ahead* of now, rather than at now, is the unusual
part. It retires every danger recorded up to three seconds after the fight ends, which
suppresses the ricochets, corpses and attack sounds generated by the fight's own last moments.
Without it a creature would leave combat and immediately re-enter the danger branch reacting
to its own victory.

**Notes** — the sidearm is idled explicitly because a creature that executed a downed enemy
ends combat holding its pistol raised. Nothing else lowers it.

## `add_evaluators`

Twenty-five questions. The ones worth naming:

```text
PureEnemy       : is there an enemy right now, with no grace period
Enemy           : ... or was there within the post-combat interval (3 s)
SeeEnemy        : can I see it
EnemySeeMe      : can it see me
ItemToKill      : do I hold a weapon
ItemCanKill     : does that weapon have ammunition
FoundItemToKill : is there a weapon on the ground I know about
FoundAmmo       : is there ammunition on the ground I know about
ReadyToKill     : is the weapon up and aimed-capable
ReadyToDetour   : is it in the flanking carry
Panic           : has my morale broken
DangerGrenade   : is there a grenade still in flight
EnemyWounded    : is the enemy downed but alive
EnemyCriticallyWounded : is it staggering from a critical hit
PlayerOnThePath : is the player between me and my target
TooFarToKillEnemy : is it beyond my weapon's useful range
ShouldThrowGrenade : is a grenade the right answer here
LowCover        : does my cover only protect me crouched
InSmartCover    : am I inside an authored smart cover
InCover / LookedOut / PositionHolded / EnemyDetoured : the sequence, latched
UseSuddenness   : latched — is the ambush still on
CriticallyWounded / KilledWounded : latched in the BRAIN's storage, not mine
```

**Invariants** — the distinction between `PureEnemy` and `Enemy` is the post-combat interval,
and both are answered by the same evaluator with a different delay. `PureEnemy` false and
`Enemy` true is precisely the window the post-combat wait runs in, and that is how the wait
is made reachable without a timer proposition.

`CriticallyWounded` and `KilledWounded` are read from *the root planner's* storage rather
than this one's. That is deliberate: both outlive a single combat engagement — a creature
staggering from a critical hit keeps staggering if combat ends — so they belong one level up.
A rebuild must keep the scope distinction; putting them in the combat planner's storage loses
the stagger whenever the branch is re-entered.

## `add_actions`

Twenty operators. Grouped by what they are for, with only the preconditions that distinguish
them from the shared set listed:

**Arming** — reachable only when the creature is not yet able to fight.

```text
GetItemToKill    requires FoundItemToKill = true, ItemToKill = false
                 effects  ItemToKill = true, ItemCanKill = true
MakeItemKilling  requires FoundAmmo = true, ItemCanKill = false
                 effects  ItemCanKill = true
```

**The cover loop** — the six operators that make up ordinary combat.

```text
GetReadyToKill    requires ItemToKill, ItemCanKill, ReadyToKill = false,
                           PlayerOnThePath = false, ShouldThrowGrenade = false
                  effects  ReadyToKill = true; clears all four sequence propositions

GetReadyToDetour  requires ItemToKill, ItemCanKill, ReadyToDetour = false
                  effects  ReadyToDetour = true

TakeCover         requires ItemToKill, ItemCanKill, ReadyToKill, InCover = false,
                           PlayerOnThePath = false
                  effects  InCover = true; clears LookedOut, PositionHolded, EnemyDetoured

KillEnemy         requires ReadyToKill, SeeEnemy, InCover, TooFarToKillEnemy = false
                  effects  PureEnemy = false; clears the three later sequence propositions

LookOut           requires ReadyToKill, InCover, LookedOut = false, SeeEnemy = false,
                           TooFarToKillEnemy = false
                  effects  LookedOut = true

HoldPosition      requires ReadyToKill, InCover, LookedOut, SeeEnemy = false,
                           PositionHolded = false
                  effects  PositionHolded = true, InCover = false

DetourEnemy       requires ReadyToKill, ReadyToDetour, InCover = false, LookedOut,
                           PositionHolded, EnemyDetoured = false, SeeEnemy = false
                  effects  EnemyDetoured = true

SearchEnemyPlanner requires ReadyToKill, SeeEnemy = false, InCover = false,
                            LookedOut, PositionHolded, EnemyDetoured
                   effects  PureEnemy = false
```

**Invariants** — the loop's whole structure is in the `SeeEnemy` condition. Every step after
taking cover requires *not* seeing the enemy: look out, hold, flank and search are what a
creature does when it has lost its target. The moment it sees the enemy again, `KillEnemy`
becomes available and all of them become unavailable, so the creature snaps back to shooting.
That single proposition is what makes a firefight read as a firefight rather than as a drill.

The sequence runs cover → look out → hold → flank → search, each step's effect the next's
precondition, and `HoldPosition` clearing `InCover` is what opens the flank.

**Alternate kill routes** — the same kill action under different names, each with its own
excuse for shooting:

```text
KillEnemyIfNotVisible        requires ReadyToKill, SeeEnemy, EnemySeeMe = false
KillEnemyIfCriticallyWounded requires ReadyToKill, SeeEnemy, EnemyCriticallyWounded
KillEnemyIfPlayerOnThePath   requires PlayerOnThePath = true, Panic = false
```

**Notes** — the first two exist so that a creature with a clean shot takes it *without*
needing to be in cover, which the ordinary kill requires. Shooting an enemy who cannot see
you, or who is staggering, does not call for cover first.

**The special cases:**

```text
RetreatFromEnemy  requires (nothing)          effects PureEnemy = false     weight 100
HideFromGrenade   requires DangerGrenade      effects Enemy = false; clears the sequence
SuddenAttack      requires UseSuddenness, Enemy   effects Enemy = false
CriticalHit       requires CriticallyWounded, Panic = false
                  effects  CriticallyWounded = false
ThrowGrenade      requires PureEnemy, ShouldThrowGrenade, Panic = false
                  effects  ShouldThrowGrenade = false
PostCombatWait    requires PureEnemy = false, Enemy = true
                  effects  Enemy = false
```

**Invariants** — `RetreatFromEnemy` has no preconditions at all and a weight of one hundred.
It is always available and almost never chosen: the planner takes it only when every cheaper
route to the goal is blocked. That is the correct encoding of "flee when there is nothing
else to do", and it is the only place cost is used instead of conditions.

`PostCombatWait` is reachable exactly in the three-second window where the immediate enemy is
gone but the delayed one is not. Its effect clears the delayed proposition, which ends the
branch.

**The sub-planners:**

```text
KillWoundedPlanner  requires EnemyWounded, Enemy     effects Enemy = false
GetDistancePlanner  requires ReadyToKill, InCover, TooFarToKillEnemy, Panic = false
                    effects  TooFarToKillEnemy = false
LowCoverPlanner     requires ItemToKill, ItemCanKill, InCover, LowCover
                    effects  LowCover = false
SmartCoverAction    requires PureEnemy, ItemToKill, ItemCanKill, InSmartCover, Panic = false
                    effects  InSmartCover = false
```

**Notes** — each of these takes over the creature entirely for a situation the general loop
handles badly: a downed enemy, a target out of range, cover that only works crouched, an
authored strongpoint. The general loop's operators exclude each case by a negative
precondition, so the sub-planner is not merely preferred but is the only available branch.

`GetDistancePlanner` requires `InCover = true` — the creature must be behind something before
it starts bounding forward, which is what makes the advance start from a position rather than
from the open.

## `save` and `load`

**Contract** — pure delegation to the planner base, which writes the plan, the current
operator and the latched propositions. The override exists so the combat planner is a
serialization point at all.

**Invariants** — what is restored is what the entry hook's restore guard then declines to
overwrite. The two are one mechanism; a rebuild that serializes the planner but not the
guard will restore the state and then immediately clear it.
