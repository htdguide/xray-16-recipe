# src/xrGame/ai/monsters/states/monster_state_panic_run.h

> Declares the bolt leaf of the panic behaviour, implemented in
> [`monster_state_panic_run_inline.h`](monster_state_panic_run_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_panic_run_inline.h`](monster_state_panic_run_inline.h.md)
**Used by** — [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md) · [`monster_state_panic_run_inline.h`](monster_state_panic_run_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf in which a panicking creature runs directly away from its enemy. Carries the two
numbers that decide when it has escaped: fifteen units of separation and fifteen seconds without
being seen.

## `CStateMonsterPanicRun`

- **enter** — prepare the path builder
- **execute** — retreat from the enemy's position at a full aggressive-profile run
- **is_finished** — far enough away *and* unseen for long enough

Stateless. Contracts are in the implementation twin.
