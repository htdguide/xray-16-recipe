# src/xrGame/script_action_condition_inline.h

> Construction and the start-of-action stamp.

**Needs** — [`script_action_condition.h`](script_action_condition.h.md) · [`xrEngine/device.h`](../xrEngine/device.h.md)
**Used by** — [`script_action_condition.h`](script_action_condition.h.md)
**Tier floor** — T3: two assignments and a clock read

## Purpose

The two operations the condition type performs.

## Construction

**Contract** — takes the set of parts that must complete and, optionally, a life time in
milliseconds. A negative life time — the default — becomes the all-ones sentinel meaning
unlimited, by the same wrap that produces it elsewhere. The start time is left unset; a
condition is not running until it is initialized.

**Notes** — the life time arrives as a floating-point value and is stored as an integer count
of milliseconds, so a script passing a fractional millisecond loses it. Scripts pass whole
milliseconds; nothing documents the unit at the script boundary, which is the kind of thing a
rebuild should fix by naming the parameter.

## `initialize`

**Contract** — stamps the start time from the global frame clock. Called when the action the
condition belongs to begins, and it is the only write to that field — a re-initialized action
restarts its clock, which is what makes an action repeated in a loop behave the same each
time.

**Invariants** — the clock is the frame clock, not the in-world alife clock, so a script
action's timeout is measured in real time and is unaffected by the game's time acceleration.
A rebuild must pick the same clock or scripted sequences will run at the wrong speed
whenever the world clock is sped up.
