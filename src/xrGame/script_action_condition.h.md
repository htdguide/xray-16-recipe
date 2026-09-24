# src/xrGame/script_action_condition.h

> Declares the rule for when a script-issued action ends: a set of parts that must all finish, and optionally a time limit.

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`script_action_condition_inline.h`](script_action_condition_inline.h.md)
**Used by** — [`script_action_condition_inline.h`](script_action_condition_inline.h.md) · [`script_action_condition_script.cpp`](script_action_condition_script.cpp.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`script_entity_action_inline.h`](script_entity_action_inline.h.md)
**Tier floor** — T3: a flag set and two timestamps

## Purpose

A script action bundles several parts — a movement, a head-turn, an animation, a sound, a
particle effect, an object interaction — and the script must say which of them decide when the
action is over. Without this, an action that plays a two-second animation while walking ten
metres would have no defined end. This type is that declaration, authored per action by the
script.

## State

```text
RECORD ScriptActionCondition
  flags      : set of part kinds      # which parts must complete for the action to end
  life_time  : int (milliseconds)     # how long the action may run; all-ones means unlimited
  start_time : int (milliseconds)     # when it began; all-ones until initialized
```

```text
ENUM ActionPartFlag                   # one bit each; combined into `flags`
  MOVEMENT    # the creature reached its destination
  WATCH       # the head-turn finished
  ANIMATION   # the animation played out
  SOUND       # the sound finished
  PARTICLE    # the particle effect finished
  OBJECT      # the object interaction finished
  TIME        # the life time elapsed
  ACT         # the action's own script-side work finished
```

**Invariants**

- The flags are **conjunctive**: every named part must report completion. Naming no parts at
  all means the action ends immediately, which scripts use for fire-and-forget actions.
- The time limit is a *part*, selected by its own flag, not a separate mechanism. A condition
  that sets the time flag and no others is a plain timer; one that sets the time flag
  alongside others makes the time an additional requirement rather than a deadline — **it
  does not cut the action short**, it extends it until the time has also elapsed. That is the
  opposite of what "life time" suggests and is the most likely misreading of this type.
- The start time is meaningless until the action is initialized. Comparing against it before
  then compares against the all-ones sentinel.

## Construction and `initialize`

**Contract** — see [`script_action_condition_inline.h`](script_action_condition_inline.h.md).

## Script surface

Registered to the script virtual machine as the condition type with its flag names. See
[`script_action_condition_script.cpp`](script_action_condition_script.cpp.md).

**Notes** — the particle flag has a bit and a name in the code but is **not exported to
script**, so no shipped script can set it. Whether that is an oversight or a deliberate
retirement is not recoverable; the bit is reserved either way.
