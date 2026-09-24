# src/xrGame/script_property_evaluator_wrapper.cpp

> Routes a planner evaluator's two overridable points into a script class, and decides what a misbehaving script evaluator answers.

**Needs** — [`script_property_evaluator_wrapper.h`](script_property_evaluator_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_property_evaluator_wrapper.h`](script_property_evaluator_wrapper.h.md)
**Tier floor** — T2: a call convention across the script boundary

## Purpose

The same two-function adapter pattern as
[`script_effector_wrapper.cpp`](script_effector_wrapper.cpp.md), applied to an evaluator's
two overridable methods — plus one decision that belongs to this file alone: what happens
when a script's evaluator returns the wrong kind of thing.

## State

`Stateless.`

## `setup(object, storage)`

**Contract** — invokes the script object's `setup` method with the subject and the shared
property storage. The script is expected to call the base version from inside it; nothing
enforces that.

## `setup_base(evaluator, object, storage)`

**Contract** — invokes the inherited `setup` directly, bypassing the script override. This
is what a script's own `setup` calls to get the standard binding; without the bypass it
would re-enter itself.

## `evaluate -> bool`

**Contract** — invokes the script object's `evaluate` and returns its answer. If the script
raises, or answers with something that is not a yes-or-no value, the failure is reported as
a script error naming the evaluator and the answer is taken as **false**.

```text
FUNCTION evaluate -> bool
  TRY
    RETURN script_object.evaluate()
  ON any failure
    log script error "evaluator [<name>] returns value with not a bool type"
    RETURN false
```

**Invariants**

- A broken evaluator answers *false*, never fails. The planner runs this call inside its
  search, several times per creature per cycle; a failure here would take down the game on
  a modder's typo. Answering false makes the operators guarded by that question
  unavailable, so the creature falls back to a simpler plan and keeps moving.
- The report is emitted **every time**, not once. On a hot evaluator that floods the log,
  which is deliberate: the flood is what makes the mistake impossible to ignore.

**Notes**

The engine distinguishes a type mismatch from any other failure only in debug builds, where
it can name the type actually returned. In a shipping build both paths produce the same
message, which is why that message claims a type error even when the script simply raised.
A rebuild should report the two cases distinctly.

## `evaluate_base(evaluator) -> bool`

**Contract** — the inherited answer, bypassing the script override, for a script's own
`evaluate` to build on.
