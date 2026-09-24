# src/xrGame/stalker_anomaly_actions.cpp

> Leaving an anomaly and probing for one — the two behaviours that keep a stalker alive in a world where the ground kills you.

**Needs** — [`stalker_anomaly_actions.h`](stalker_anomaly_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`RadioactiveZone.h`](RadioactiveZone.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`Inventory.h`](Inventory.h.md) · [`sound_player.h`](sound_player.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — [`stalker_anomaly_actions.h`](stalker_anomaly_actions.h.md)
**Tier floor** — T2: touches the creature's restrictor set and its touch-sense list every cycle.

## Purpose

Anomalies outrank enemies in a stalker's priorities (see
[`stalker_planner.cpp`](stalker_planner.cpp.md)) because an anomaly kills faster than a
rifle. These are the two actions that branch implements. They are interesting for one
reason that generalizes: **escaping an anomaly is expressed as a change to the creature's
restrictor set, not as a path**. The creature does not walk to a computed safe point; it
declares the zones it is in as forbidden and asks the movement layer for the nearest place
it is still allowed to be. The pathfinder then does the geometry.

## State

Neither action holds meaningful state. `GetOutOfAnomaly` keeps two reusable lists of entity
identifiers so the per-cycle restrictor update can pass an add-set and a remove-set without
allocating; both are cleared at the top of every cycle.

## `GetOutOfAnomaly` — entry

**Contract** — configure the creature for a careful, alert walk out, and assert that the
anomaly proposition is true so the planner stays in this branch while the action runs.

```text
FUNCTION initialize()
  base.initialize()
  silence all sounds except the low-level humming channel
  movement.desired_direction := none      # direction comes from the pathfinder
  movement.path_type         := level path      # route through the navigation mesh
  movement.detail_path_type  := smooth          # no sharp corners
  movement.body_state        := standing
  movement.gait              := walk
  movement.mental_state      := danger
  sight.mode                 := look where I am going
  IF there is an enemy AND the item in hand is already the best weapon THEN
    weapon_goal(IDLE, best_weapon)        # keep it out, do not use it
  ELSE
    weapon_goal(IDLE)                     # whatever is in hand, idle
  assert property Anomaly = true
```

**Invariants** — the gait is *walk*, never run, and this is the decision the whole action
turns on. Running through an anomaly field is how a creature dies in it; the escape is
deliberately slow. A rebuild that "improves" this by running will kill every stalker that
steps in a gravity anomaly.

**Notes** — the weapon branch looks redundant and is not. If the creature is already
holding its best weapon *and* has an enemy, the weapon is kept in hand and merely idled, so
that the creature emerges from the anomaly still armed and can resume the fight without a
draw animation. Otherwise the weapon goal is issued without naming an item, which leaves
whatever is in hand alone. The point is that escaping an anomaly must never cost the
creature a weapon state it will need in two seconds.

Asserting the anomaly proposition on entry is what stops the planner oscillating: the
evaluator that produced it may go false the moment the creature leaves the zone boundary,
and without the assertion the plan would be rebuilt mid-step.

## `GetOutOfAnomaly` — per cycle

**Contract** — re-assert the movement configuration, recompute which zones the creature is
standing in, forbid them, and ask the movement layer to head for the nearest position that
is still permitted. Allocates nothing; reads the creature's touch-sense list and its
authoritative record's restrictor list.

```text
FUNCTION execute()
  base.execute()
  re-assert path type, detail path type, posture, gait and mental state

  new_forbidden := empty
  record := alife.record_for(self)                  # the authoritative server object
  IF record is missing OR record is not a human-shaped entity THEN RETURN
  already_permitted := record.dynamic_permitted_restrictors

  FOR EACH touched IN self.feel_touch               # what is physically overlapping me
    IF touched is not a zone THEN CONTINUE
    IF the zone declares no restrictor volume THEN CONTINUE
    IF the zone is a radiation field THEN CONTINUE          # see Notes
    IF touched.id IN already_permitted THEN CONTINUE        # see Notes
    new_forbidden.add(touched.id)

  movement.restrictions.add_restrictions(permitted = empty, forbidden = new_forbidden)
  movement.head_for_nearest_accessible_position()
```

**Invariants**

- The movement configuration is re-asserted every cycle rather than set once on entry.
  Anything else in the creature — a script, a hit reaction, the smart-cover layer — may
  have overwritten it between cycles, and an anomaly escape that silently became a run is
  fatal. Re-asserting is cheap; being wrong is not.
- Restrictors are only ever *added* here. The removal happens when the action ends and the
  restrictor set is rebuilt, not incrementally, because a zone the creature has left is
  still a zone it must not path back through while escaping.

**Notes** — two exclusions, two different reasons.

A **radiation field** is skipped because radiation is survivable and ubiquitous: treating
every radioactive patch as forbidden ground would forbid most of several levels and leave
the creature with nowhere accessible at all. Radiation is handled by the health system, not
by pathing.

A zone already in the creature's **permitted** restrictor list is skipped because
something — a script, a smart terrain, a quest — has explicitly declared that this creature
is allowed inside this zone. Overriding that here would make a scripted setpiece impossible.
The permitted list is read from the creature's authoritative record rather than from the
live instance, because the record is what survives the creature going offline and coming
back.

The early return when the record is missing or is not a human-shaped entity is not a null
check dressed up: it is the statement that this action only applies to entities whose
authoritative record carries a restrictor list, and a rebuild whose records all carry one
can drop it.

## `GetOutOfAnomaly` — exit

**Contract** — restore the creature's sound mask so ordinary chatter resumes, unless the
creature died inside the anomaly, in which case its sound state is left as it was.

## `DetectAnomaly` — entry

**Contract** — start probing. Silences chatter, arms a randomized completion delay, and
aims the throw ten units straight ahead of where the creature is looking.

```text
FUNCTION initialize()
  base.initialize()
  silence all sounds except the humming channel
  inertia_time := 15 seconds + uniform random up to 5 more
  target := point 10 units forward along the eye direction
  aim_throw_at(target)
```

**Notes** — the inertia time is how long the action refuses to report completion, so the
creature commits to probing for a believable interval instead of throwing one bolt and
moving on. It is randomized per invocation so that a group of stalkers arriving at the same
anomaly do not probe in lockstep — the same trick used throughout the creature layer to
break synchrony without any coordination.

Ten units ahead is the throw distance, in world units, and is a plain tuning choice: far
enough that the bolt lands outside the creature's own stopping distance, close enough that
the creature can see where it landed.

## `DetectAnomaly` — per cycle

**Contract** — throw a bolt, unless the probe has run long enough or an enemy has appeared,
in which case the anomaly proposition is cleared and the planner leaves this branch.

```text
FUNCTION execute()
  base.execute()
  IF inertia has elapsed OR an enemy is selected THEN
    set property Anomaly = false           # stop probing; the planner will re-branch
    RETURN
  weapon_goal(THROW, inventory.item_in_bolt_slot)
```

**Invariants** — an appearing enemy cancels the probe immediately. Anomalies outrank
enemies for *escape*, but not for *investigation*: being curious about a patch of ground is
not worth being shot over.

**Notes** — the bolt is addressed by its inventory slot rather than by class. Bolts are the
series' probing tool: an infinite supply of one throwable held in a dedicated slot, thrown
to make anomalies discharge visibly. If the slot is empty the throw goal simply has nothing
to act on and the creature stands there until the inertia expires, which is the intended
degradation.

Clearing the proposition rather than reporting the action complete is how a planner action
ends here: the effect it claimed has been achieved by declaration, and the planner replans
on the next cycle.

## `DetectAnomaly` — exit

**Contract** — idle the weapon and unmask sounds, unless the creature is dead. Idling the
weapon matters because the action leaves the creature mid-throw-sequence otherwise, holding
a primed bolt forever.
