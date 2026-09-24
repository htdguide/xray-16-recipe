# src/xrUICore/SpinBox/UISpinNum.cpp

> The two numeric spin boxes — one over integers, one over reals — each binding a bounded stepped number to a named setting.

**Needs** — [`UISpinNum.h`](UISpinNum.h.md) · [`UICustomSpin.h`](UICustomSpin.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [Data: User settings](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UISpinNum.h`](UISpinNum.h.md)
**Tier floor** — T3: bounded stepping and number formatting.

## Purpose

Fills the spin box's four-operation contract with a number, twice: once for whole values such
as a texture-detail level, once for fractional ones such as a mouse sensitivity. The pair is
in one file because the two are the same nine lines with a different arithmetic type, and
nothing but the type differs — which is itself the observation that matters, since the two do
*not* behave identically at their bounds.

## State

```text
RECORD SpinNum EXTENDS CustomSpin
  min, max, step, value : int        # 0, 100, 1, 0
  backup                : int

RECORD SpinFlt EXTENDS CustomSpin
  min, max, step, value : real       # 0, 100, 0.1, 0
  backup                : real
```

**Invariants** — the displayed text is always the current value rendered by the type's own
rule, so every mutation ends by rewriting it. Real values are displayed to one decimal place
regardless of the step, so a step finer than a tenth moves the value invisibly.

## `IncVal` / `DecVal` / `CanPressUp` / `CanPressDown`

**Contract** — the contract the base demands. The two types answer it differently, and the
difference is behaviour, not style.

```text
# integer
FUNCTION can_press_up()   -> bool  RETURN value + step <= max
FUNCTION can_press_down() -> bool  RETURN value - step >= min
FUNCTION inc_val()   IF can_press_up()   THEN value <- value + step ; redisplay
FUNCTION dec_val()   IF can_press_down() THEN value <- value - step ; redisplay

# real
FUNCTION can_press_up()   -> bool  RETURN value + step <= max
FUNCTION can_press_down() -> bool  RETURN value - step > min OR value - step ~= min
FUNCTION inc_val()   value <- clamp(value + step, min, max) ; redisplay
FUNCTION dec_val()   value <- clamp(value - step, min, max) ; redisplay
```

**Notes** — the integer form *refuses* a step that would overshoot, so a range whose span is
not a whole number of steps can never reach its own maximum: with a minimum of 0, a maximum
of 10 and a step of 3, the value stops at 9. The real form instead *clamps*, so it always
reaches both bounds exactly, at the cost of a final step shorter than the others.

Whether that divergence is intentional is not recoverable. It is observable in the shipped
options screen and a rebuild that unifies the two will change which values a slider-like
integer setting can take.

The real form's lower predicate uses an approximate equality against the minimum, because
repeated subtraction of a tenth accumulates error and an exact comparison would leave the down
arrow greyed out a step early. The upper predicate has no such tolerance — an asymmetry that
makes the top of a real range slightly harder to reach than the bottom.

## `OnBtnUpClick` / `OnBtnDownClick`

**Contract** — move the value one step, then delegate to the base so the owner is notified of
the click. The arrow press and the auto-repeat therefore take different paths: a press notifies,
a repeat does not.

## The options-item protocol

**Contract** — the five operations the settings screen drives, in both types.

- **Read from the setting** — pulls the value *and its bounds* from the named setting, so the
  range is data, not code, and then redisplays.
- **Back up** — copies the current value aside.
- **Save** — writes the value back to the setting and marks the item saved.
- **Undo** — restores the backup, redisplays, and marks the item reverted.
- **Changed?** — compares against the backup; the real type compares approximately, since a
  value that made a round trip through text and back must still count as unchanged.

**Notes** — reading the bounds from the setting is why the constructor's defaults of 0..100
almost never apply: the shipped settings declare their own ranges alongside their values.

## `InitSpin` / `SetValue` / `SetMin` / `SetMax` / `Value`

**Contract** — initialization runs the base's build and then paints the current value, so a
freshly built box is never blank. `SetValue` is the only writer of the display: it formats the
number — plain base-ten digits for the integer type, one decimal place for the real one — and
hands it to the text block. The bound setters are plain writes with no re-clamping of the
current value.
