# src/xrGame/restricted_object_obstacle.h

> Declares a restricted object that also fences out the navigation vertices currently blocked by other objects.

**Needs** — [`restricted_object.h`](restricted_object.h.md) · [`obstacles_query.h`](obstacles_query.h.md)
**Used by** — [`restricted_object_obstacle.cpp`](restricted_object_obstacle.cpp.md) · [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md) · [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md)
**Tier floor** — T3: a declaration plus two borrowed queries

## Purpose

Restrictors are authored geometry; obstacles are other objects standing in the way right now.
Pathing needs both, and they arrive from different places. This type binds the two: it takes
two obstacle queries — one over objects that do not move and one over objects that do — and
widens every border operation to mask their blocked vertices out of the navigation graph as
well.

The substance is in [`restricted_object_obstacle.cpp`](restricted_object_obstacle.cpp.md).

## State

```text
RECORD RestrictedObjectObstacle extends RestrictedObject
  static_query  : reference to ObstaclesQuery    # borrowed; outlives this object
  dynamic_query : reference to ObstaclesQuery    # borrowed; outlives this object
```

**Invariants**

- Both queries are **borrowed, not owned**. They belong to the creature's obstacle manager
  and must outlive this mixin. That is what lets several consumers share one computed
  blocked set per frame instead of each recomputing it.
- The two accessors assert that **no border is currently installed**. Reading a query while
  a border is up would hand out a set the border was already built from, and a caller acting
  on it would double-apply. The assertion states the rule: obstacle sets are consulted
  between borders, never during one.

Exported units: the constructor taking the creature and the two queries; the three
`add_border` forms and `remove_border`, each overriding the base to add the obstacle masking;
and the two query accessors described above.
