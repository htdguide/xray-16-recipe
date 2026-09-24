# src/xrUICore/Options — the settings-control protocol

> Four operations every settings widget implements against one console variable, and a registry
> that drives them across a whole page at once.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The mixin that makes a widget a settings control, and the registry that groups those controls
by page name.

A settings control binds to **one console variable by name**, reads and writes it through typed
accessors, and implements four operations: load the current value, back it up, commit the
current value, restore the backup. It also records what kind of restart committing its change
will cost.

The registry holds controls keyed by page name, runs each of the four operations across a whole
page in one pass, and after an accept discharges the accumulated restart requirements as console
commands.

## The load-bearing ideas

**Backup-and-commit is the whole model.** A settings page is not edited in place: opening it
loads and backs up every control, editing changes only the control, accepting commits, and
cancelling restores from the backup. Because the backup lives on each control rather than in one
snapshot, a rebuild can add a control without touching the page machinery.

**The variable is named, and the typing is at the accessor.** A control does not hold a typed
reference; it holds a name and calls a typed accessor, which is what lets the XML vocabulary
declare geometry and binding in the same element. It also means a misspelled name is a runtime
failure, not a load-time one.

**Restart requirements are accumulated, not asked.** Each control declares what its change costs
— nothing, a renderer restart, a full restart — and the registry unions those across the page
and discharges them once. Asking the player per control would ask several times for one restart.

**A page is a name, not an object.** Controls register themselves under a page name, so a screen
does not have to own a list of its own controls, and a script-authored screen can join an
existing page.

## The twins

| Twin | Role |
|---|---|
| [`UIOptionsItem.cpp`](UIOptionsItem.cpp.md) | Binding to one console variable by name, the typed accessors, and the recorded restart cost |
| [`UIOptionsItem.h`](UIOptionsItem.h.md) | The four-operation protocol a widget implements to be a settings control |
| [`UIOptionsManager.cpp`](UIOptionsManager.cpp.md) | The name-to-controls map, the four group-wide passes, and the pending restart flags discharged as console commands |
| [`UIOptionsManager.h`](UIOptionsManager.h.md) | The registry's declaration |
