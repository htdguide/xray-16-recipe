# src/xrGame/ai/monsters/controller/controller_state_panic.h

> A declared-but-unbuildable controller panic state: three substate names and nothing behind them.

**Needs** — [`../state.h`](../state.h.md)
**Used by** — [`controller_state_panic_inline.h`](../../../controller_state_panic_inline.h.md)
**Tier floor** — T3: a declaration with no body

## Purpose

This file declares a controller-specific panic composite with three intended substates — run,
steal away, and look around — and then includes an implementation file **that does not exist in
the tree**. Nothing includes this header, which is the only reason the project compiles. The
build description lists the missing implementation file at the project root, where there is no
such file either.

It is recorded here because the mirror must be complete and because the declaration states an
intent worth knowing: the controller was meant to panic differently from the generic creature —
sprinting, then *stealing* away rather than simply sprinting further, then stopping to look —
and it ships using the generic panic state instead (see
[`controller_state_manager.cpp`](controller_state_manager.cpp.md)).

A rebuild should either implement that three-phase panic deliberately or omit the file. Nothing
is lost by omitting it.

## `CStateControllerPanic`

**Contract** — declared, never defined. A composite state with a substate re-selection hook and
three substate identifiers (run, steal, look around). No body exists, so there is no contract to
state.
