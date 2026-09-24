# src/xrGame/stalker_velocity_collection_inline.h

> The speed lookup, and the legality rules it enforces on the posture it is asked about.

**Needs** — [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md)
**Used by** — [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md)
**Tier floor** — T2: a three-way dispatch with preconditions.

## Purpose

Separated from the header for readability, but it is not trivial: it is where the sparseness
of the table in [`stalker_velocity_collection.cpp`](stalker_velocity_collection.cpp.md)
becomes an enforced contract. The function is on the movement hot path — it is asked every
time a creature's posture is recomputed — which is why it is an indexing operation and not a
search.

## `velocity(mental_state, body_state, movement_type, movement_direction)`

**Contract** — the speed for one complete posture. Every caller must supply a posture the
mental state admits; supplying one it does not is a programming error, not a runtime case.

```text
FUNCTION velocity(mental, body, type, direction) -> real
  REQUIRE type IS NOT standing              # a stationary creature has no speed to look up

  IF mental IS danger
    REQUIRE body IN [stand, crouch]
    REQUIRE type IN [walk, run]
    REQUIRE direction IN [forward, backward, left, right]
    RETURN danger[body][type][direction]

  IF mental IS free
    REQUIRE body IS stand                   # an unworried stalker does not crouch-walk
    REQUIRE direction IS forward            # nor sidestep
    REQUIRE type IN [walk, run]
    RETURN free[type]

  IF mental IS panic
    REQUIRE body IS stand
    REQUIRE type IS run                     # panic has exactly one gait
    REQUIRE direction IS forward
    RETURN panic

  FAIL WITH unknown mental state
```

**Invariants** — the requirements are the whole content of this file. They are not defensive
checks against bad input; they are the statement *these postures do not exist*, enforced at
the one place that would otherwise silently return a neighbouring cell. A rebuild that makes
the table dense and returns a plausible number for, say, panic-crouch-backward has not
reproduced the model: it has invented eighteen speeds nobody tuned, and creatures will use
them.

**Notes** — in the shipped (non-diagnostic) build the checks are compiled out and the
function is three indexed reads. That is the reason for the shape: the cost of the contract
is paid during development and nothing at runtime. A rebuild in a language without that
split should keep the checks — the function is cheap relative to the movement work around
it — and rely on measurement before removing them.

The unreachable tail of the function deliberately computes a division by zero rather than
returning a plausible default, so that a mental state nobody handled announces itself
immediately instead of producing a creature that moves at exactly zero.
