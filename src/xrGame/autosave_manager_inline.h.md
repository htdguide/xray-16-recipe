# src/xrGame/autosave_manager_inline.h

> The counter and timestamp operations of the autosave manager.

**Needs** — [`autosave_manager.h`](autosave_manager.h.md)
**Used by** — [`autosave_manager.h`](autosave_manager.h.md)
**Tier floor** — T3: field arithmetic

## Purpose

Splits the one-line operations of [`autosave_manager.h`](autosave_manager.h.md) out of the
declaration so they inline at their call sites — and the readiness counter's call sites are
scattered across the whole game layer, so that matters more here than usual. Purely a
compilation arrangement; a rebuild has no second file.

## State

`Stateless.` Operates on the manager's fields, described in
[`autosave_manager.cpp`](autosave_manager.cpp.md).

## the operations

**Contract** — `autosave_interval` and `last_autosave_time` read; `update_autosave_time`
stamps now; `delay_autosave` pushes the stamp forward by the delay interval rather than
setting a separate retry time; `inc_not_ready` and `dec_not_ready` move the veto counter,
with the decrement asserting the counter is non-zero; `ready_for_autosave` is the counter
being zero.

**Notes** — the decrement's assertion is the file's only real content. An unmatched decrement
lifts somebody else's veto, and the resulting save is taken in the middle of a transaction
some unrelated subsystem was performing — a failure that surfaces much later as a corrupt
save with no trace back to here.
