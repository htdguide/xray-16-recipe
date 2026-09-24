# src/xrGame/patrol_path_manager_inline.h

> Construction and the setters, where changing a policy invalidates the route and changing it to its current value does not.

**Needs** — [`patrol_path_manager.h`](patrol_path_manager.h.md)
**Used by** — [`patrol_path_manager.cpp`](patrol_path_manager.cpp.md) · [`patrol_path_manager.h`](patrol_path_manager.h.md)
**Tier floor** — T3: field access with an invalidation rule

## Purpose

Split from the class body by the original language's rules. The accessors are trivial; the
three setters are not, and they follow the same conditional-invalidation pattern as the
movement manager's, for the same reason.

## State

Operates on the record declared in [`patrol_path_manager.h`](patrol_path_manager.h.md).

## Construction

**Contract** — binds the restrictor set and the game object, leaves the route unbound, sets
both policies to their sentinel, marks the manager **actual and completed**, clears the
randomness flag, sets all three point indices to the all-ones sentinel, and sets the
destination to the infinite sentinel.

**Invariants** — a freshly constructed manager is *completed*, not merely idle. Completed is
the state that means "do not ask me for a next point", and a manager with no route must be in
it or the movement pipeline will call into an unbound route. Setting a route clears it.

The destination is the all-coordinates-infinite sentinel, and the accessor asserts the
destination is a valid number before handing it out — so reading it before a point has been
selected is caught rather than silently producing a path to infinity.

## `set_path`

**Contract** — adopts a route. Returns immediately if it is the route already held. Otherwise
records it and its name, marks the manager stale and not completed, and resets the three point
indices and both policies to their sentinels.

```text
FUNCTION set_path(route, name)
  IF route is the one already held THEN RETURN     # re-issuing the same order is free
  route := route; name := name
  actual := false; completed := false
  reset point indices and both policies
```

**Invariants** — the early return on an unchanged route is what makes it safe for a brain to
re-issue its patrol order every update, which they do.

Adopting a route resets the **policies** as well as the position. A rebuild must therefore set
the policies *after* the route, and the three-argument convenience form does exactly that. A
rebuild that sets them first will have them silently erased.

**Notes** — two convenience forms exist: one looks the route up by name in the world's route
storage, the other does that and then applies both policies and the randomness flag. The
lookup-by-name form is what scripts use.

## `set_start_type` / `set_route_type`

**Contract** — each conjoins the freshness flag with "the new value equals the old" before
storing, and then conjoins the completed flag with the resulting freshness.

```text
FUNCTION set_start_type(new_policy)
  actual := actual AND (start_policy == new_policy)
  completed := completed AND actual
  start_policy := new_policy
```

**Invariants** — two rules in one line each. First, setting a policy to the value it already
holds must not invalidate — brains re-issue policies every update. Second, **invalidating a
route un-completes it**: a creature that had finished its route and is then given a different
policy must start walking again. Without the second line the creature would stay completed
forever and never move, which is the failure a rebuild is most likely to reproduce by
omission.

The randomness setter has neither rule: it changes only *which* branch is chosen at a fork,
never whether the current choice is still valid.

## `make_inactual`

**Contract** — clears both the freshness flag and the completed flag unconditionally.

**Notes** — the external forcing function, used when something outside the route changed — a
restrictor moved, a script intervened — and the current choice can no longer be trusted. It is
the two effects of the policy setters without the comparison.

## Accessors

**Contract** — `actual`, `failed`, `completed`, `get_path`, `random`, `get_current_point_index`
and `extrapolate_callback` read fields. `object` hands out the restrictor set, asserting it
exists. `destination_position` hands out the chosen world point, asserting it is a valid
number first.
