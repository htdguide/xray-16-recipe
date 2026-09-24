# src/xrGame/setup_manager.h

> Declares the generic weighted-random action selector that the sight manager and its kin are built from.

**Needs** — [`Common/object_broker.h`](../Common/object_broker.h.md) · [`setup_manager_inline.h`](setup_manager_inline.h.md)
**Used by** — [`setup_manager_inline.h`](setup_manager_inline.h.md) · [`sight_manager.h`](sight_manager.h.md)
**Tier floor** — T2: a small owning registry with a per-update selection step

## Purpose

Declares a *parameterized* manager: given an action kind, the kind of object those actions
drive, and a kind of action identifier, it owns a set of identified actions, picks one,
and keeps it running until it completes. The bodies are in
[`setup_manager_inline.h`](setup_manager_inline.h.md) and carry the substance; a rebuild
without templates writes one of these per action kind, or one over an interface.

## Exported units

- **The class**, over three parameters: the action type, the driven object type, and the
  action identifier type.
- **`add_action(id, action)`** — register an action under an identifier; ownership
  transfers.
- **`action(id)`, `current_action()`, `current_action_id()`** — lookup and the running
  selection.
- **`select_action()`** — the weighted choice; the file's one real algorithm.
- **`update()`** — the per-tick step: select, initialize on change, execute.
- **`reinit()` / `clear()`** — drop every action and forget the selection.
- **`object()`, `actions()`** — the driven object and the registry.

## Notes

What this demands of an action type is the real interface and a rebuild must satisfy it:
an action answers whether it is *applicable* now, carries a *weight*, answers whether it
has *completed*, and has *initialize*, *execute* and *finalize* steps plus a setter for
the object it drives. Nothing else about an action is assumed.
