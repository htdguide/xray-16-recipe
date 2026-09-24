# src/xrGame/action_management_config.h

> One switch: whether the planner's actions log their lifecycle.

**Needs** — _(none)_
**Used by** — [`action_base.h`](action_base.h.md) · [`property_evaluator.h`](property_evaluator.h.md)
**Tier floor** — T4: a build-time flag

## Purpose

The goal-and-plan layer's only tracing facility is a per-action line printed at each
lifecycle stage. It is expensive enough — a string per action per creature per update —
that it is compiled out of release builds, and this file is the single place that decision
is made, so that every file in the action family agrees on it.

## State

`Stateless.`

## Action logging

**Contract** — action lifecycle logging is enabled exactly when the build is a debug
build.

**Notes** — the flag changes the *layout* of the action type, not only its behaviour: the
per-instance logging flags are members. A rebuild should make the tracing a run-time
setting instead, since the AI is precisely the layer where a shipping build's misbehaviour
most needs explaining.
