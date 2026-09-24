# src/xrGame/stalker_decision_space.h

> The whole vocabulary of a stalker's brain: every proposition its planners may reason about, every operator that may change one, and the two ways it may aim its eyes.

**Needs** — _(none: this is a vocabulary declaration)_
**Used by** — [`ai_stalker_script.cpp`](ai/stalker/ai_stalker_script.cpp.md) · [`member_order.h`](member_order.h.md) · [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md) · [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) · [`smart_cover_loophole_planner_actions.h`](smart_cover_loophole_planner_actions.h.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`smart_cover_planner_target_provider.cpp`](smart_cover_planner_target_provider.cpp.md) · [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md) · [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md) · [`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md) · [`stalker_anomaly_planner.cpp`](stalker_anomaly_planner.cpp.md) · _and 22 more_
**Tier floor** — T3: three flat namespaces of names. Nothing here touches a machine.

## Purpose

Planning is a search from the current world state to a goal state, and every planner in
the stalker searches the *same* space. This file is that space: one identifier set for
propositions, one for operators, one for sight modes. It exists as a separate file for a
reason that matters to a rebuild — the sub-planners (combat, danger, anomaly, death, smart
cover) are written in different files by different concerns, and they must agree on what
"in cover" means without any of them owning the definition. A shared vocabulary file is
the cheapest way to make that agreement structural instead of conventional.

Two properties of the identifiers are load-bearing:

- **They are opaque and dense.** Every planner indexes its evaluator and operator tables
  by these values, so they must be small integers with no gaps that matter. They are never
  written to a save file or a network message, so the *values* are free — only their
  uniqueness is required.
- **The set is open at the top.** One proposition and one operator identifier are reserved
  for the script layer, and the script-extensible planner allocates further identifiers
  above them at runtime. A rebuild must therefore treat these as the *engine-reserved
  prefix* of a larger space, not as the whole space.

## `WorldProperty`

**Contract** — the name of one boolean question about the world. An evaluator answers
exactly one of these; an operator's preconditions and effects are expressed as a set of
(property, expected answer) pairs over them. No property carries a value beyond the
boolean; anything richer (which enemy, which cover point) is held on the creature and read
by whichever action needs it, which is why the planner stays cheap.

The set below is grouped the way the brain is grouped, and the grouping is the useful part
of the page: reading one group tells a rebuilder what that sub-planner is allowed to think
about.

```text
ENUM WorldProperty

  # life and the top-level branch
  Alive            # the creature is alive
  Dead             # the creature is dead
  AlreadyDead      # the death sequence has already been played out
  ALife            # the off-screen-life branch is satisfied
  PuzzleSolved     # the never-true root goal; see stalker_planner.cpp
  SmartTerrainTask # a smart terrain has handed this creature a job
  Items            # there is something nearby worth picking up
  Enemy            # there is an enemy, or there was one recently
  Danger           # a danger is recorded

  # weapon readiness — the chain a stalker walks before it can shoot at all
  ItemToKill        # a weapon is in inventory
  FoundItemToKill   # a weapon is known to exist nearby
  ItemCanKill       # the weapon has ammunition
  FoundAmmo         # ammunition is known to exist nearby
  ReadyToKill       # weapon drawn, loaded, aimed-capable
  ReadyToDetour     # weapon in the posture used while flanking

  # the visibility duel
  SeeEnemy          # the enemy is visible to me now
  EnemySeeMe        # I believe the enemy can see me
  PureEnemy         # the enemy is a real combatant, not a wounded or scripted one
  UseSuddenness     # the enemy has not noticed me, so the ambush opening applies
  Panic             # the creature's morale has broken

  # cover and position
  InCover              # standing at the chosen cover point
  LookedOut            # has leaned out of that cover at least once
  PositionHolded       # has held this position for its authored duration
  EnemyDetoured        # the flanking manoeuvre has completed
  UseCrouchToLookOut   # the chosen cover is low, so looking out means crouching
  CoverActual          # the chosen cover point is still the right one
  CoverReached         # the creature is standing at it
  LookedAround         # the searching sweep has completed
  LowCover             # the creature is behind cover that only protects crouched
  EnemyLocationReached # the enemy's last known position has been walked to
  AmbushLocationReached

  # wounded enemies — a separate courtesy/execution sequence
  EnemyWounded
  WoundedEnemyReached
  WoundedEnemyPrepared
  WoundedEnemyAimed
  KilledWounded
  PausedAfterKill

  # constraints on firing
  PlayerOnThePath          # the player stands between me and my target
  CriticallyWounded        # I am in the critical-hit state
  EnemyCriticallyWounded
  TooFarToKillEnemy
  StartedToThrowGrenade
  ShouldThrowGrenade
  GrenadeExploded

  # the four danger kinds, one sub-planner each
  DangerUnknown            # something happened, source unknown
  DangerInDirection        # a direction is known
  DangerGrenade            # a live grenade is nearby
  DangerBySound            # a sound was heard

  # anomalies
  Anomaly          # an undetected anomaly is nearby
  InsideAnomaly    # the creature is standing in one

  # smart cover: the loophole state machine
  InSmartCover
  SmartCoverEntered
  SmartCoverActual
  ExitSmartCover
  LoopholeIdle  LoopholeActual  LoopholeFire  LoopholeFireNoLookout
  ReadyToIdle   ReadyToLookout  ReadyToFire   ReadyToFireNoLookout
  LoopholeCanStayIdle  LoopholeCanLookout  LoopholeCanFire  LoopholeCanFireNoLookout
  LoopholeCanFireAtEnemy  LoopholeCanExitWithAnimation  LoopholeExitable
  LoopholeUseDefaultBehaviour
  LoopholeLastHitWasLongAgo   # nobody has shot at this loophole recently
  LoopholeTooMuchTimeFiring   # firing from one loophole too long is how you get flanked
  PlannerHasTarget
  StayIdle

  # extension point
  Script           # first identifier the script layer may use; it allocates upward
  Dummy            # "no property": the largest representable value
```

**Notes** — the `Dummy` value is deliberately the maximum of the identifier's range rather
than a distinguished small value, so that "no property" sorts after every real one and an
uninitialized slot is loud rather than silently meaning `Alive`. A rebuild with an
`optional<WorldProperty>` gets the same safety for free and should use it.

`Dead` and `AlreadyDead` are not redundant: the first is a fact about the body, the second
a fact about the *animation and bookkeeping* having run. The death planner needs both so
that it can play a death exactly once.

## `WorldOperator`

**Contract** — the name of one action or sub-planner that may appear in a plan. An operator
identifier is a key into a planner's operator table; the object behind it supplies the
preconditions, the effects and a weight (its cost to the search).

The grouping again carries the information:

```text
ENUM WorldOperator

  # death
  Dead  Dying

  # off-screen life and smart-terrain jobs
  GatherItems  ALifeEmulation  SmartTerrainTask
  SolveZonePuzzle  ReachTaskLocation  AccomplishTask
  ReachCustomerLocation  CommunicateWithCustomer

  # anomaly
  GetOutOfAnomaly  DetectAnomaly

  # combat: arming
  GetItemToKill  FindItemToKill  MakeItemKilling  FindAmmo

  # combat: fighting
  AimEnemy  GetReadyToKill  GetReadyToDetour  KillEnemy
  RetreatFromEnemy  TakeCover  LookOut  HoldPosition  GetDistance
  DetourEnemy  SearchEnemy  HideFromGrenade  SuddenAttack
  KillEnemyIfNotVisible  KillEnemyIfPlayerOnThePath
  KillEnemyIfCriticallyWounded  CriticallyWounded
  ReachWoundedEnemy  AimWoundedEnemy  PrepareWoundedEnemy  KillWoundedEnemy
  PauseAfterKill  PostCombatWait  ThrowGrenade
  RunToCover  WaitInCover  ReachEnemyLocation
  ReachAmbushLocation  HoldAmbushLocation  LowCover

  # smart cover
  InSmartCover  SmartCoverEnter  ChangeLoophole  GoToLoophole  ExitSmartCover
  Animation  SmartCoverIdle  SmartCoverLookout  SmartCoverFire
  SmartCoverReload  SmartCoverFireNoLookout  SmartCoverExit
  SmartCoverIdle2Lookout  SmartCoverLookout2Idle
  SmartCoverIdle2Fire     SmartCoverFire2Idle
  SmartCoverIdle2FireNoLookout  SmartCoverFireNoLookout2Idle
  LoopholeTargetIdle  LoopholeTargetLookout  LoopholeTargetFire
  LoopholeTargetFireNoLookout  LoopholeTargetDefaultBehaviour
  WaitAfterSmartCoverExit

  # danger: four sub-planners plus their leaf actions
  DangerUnknownPlanner  DangerInDirectionPlanner
  DangerGrenadePlanner  DangerBySoundPlanner
  DangerUnknownTakeCover  DangerUnknownLookAround  DangerUnknownSearchEnemy
  DangerInDirectionTakeCover  DangerInDirectionLookOut
  DangerInDirectionHoldPosition  DangerInDirectionDetourEnemy
  DangerInDirectionSearchEnemy
  DangerGrenadeTakeCover  DangerGrenadeWaitForExplosion
  DangerGrenadeTakeCoverAfterExplosion  DangerGrenadeLookAround
  DangerGrenadeSearch

  # the root planner's six branches
  DeathPlanner  ALifePlanner  CombatPlanner  AnomalyPlanner  DangerPlanner

  # extension point
  Script  Dummy
```

**Notes** — the explicit pairwise transition operators in the smart-cover block
(`Idle2Lookout`, `Fire2Idle`, and so on) are not an unrolled loop. Each pair names a
*distinct authored animation*, and the planner picks the transition by planning over the
loophole's current and wanted state, so the enumeration of pairs is the data the search
walks. A rebuild that collapses them into a single parametrized "transition" operator must
still be able to plan a route between two loophole states through the transitions the
level data actually authored, because a loophole may permit some transitions and not
others.

The five `*Planner` operators are the only ones whose implementations are themselves
planners; everything else is a leaf action. The identifier set does not distinguish them,
and does not need to: a planner used as an operator presents the same precondition/effect
contract as an action.

## `SightActionType`

**Contract** — which of two aiming intents an action wants while it runs. The sight system
is otherwise driven by its own parameters; this narrows the choice to the two that a
combat action cares about.

```text
ENUM SightActionType
  WatchItem     # keep the eyes on the object being manipulated
  WatchEnemy    # keep the eyes on the current enemy
  Dummy
```
