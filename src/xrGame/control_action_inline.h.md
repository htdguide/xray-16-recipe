# src/xrGame/control_action_inline.h

> The do-nothing defaults of the behaviour-step base.

**Needs** — [`control_action.h`](control_action.h.md)
**Used by** — [`control_action.h`](control_action.h.md)
**Tier floor** — T3: empty bodies

## Purpose

Supplies the bodies for [`control_action.h`](control_action.h.md), separated so that they
inline everywhere — which is the point, since the base's whole value is that an unoverridden
hook costs nothing. A rebuild has no second file.

## State

`Stateless.`

## the defaults

**Contract** — a step is applicable and already complete; setup, execution and teardown do
nothing; link removal does nothing. Binding the subject requires a real subject and reading it
requires one to have been bound. See [`control_action.h`](control_action.h.md) for why the
defaults are these and not their opposites.
