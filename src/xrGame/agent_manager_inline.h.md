# src/xrGame/agent_manager_inline.h

> The squad brain's subordinate accessors, each asserting the part exists before handing it out.

**Needs** — [`agent_manager.h`](agent_manager.h.md)
**Used by** — [`agent_manager.h`](agent_manager.h.md)
**Tier floor** — T3: field access.

## Purpose

Seven one-line accessors and the manager's constant name. They exist as a separate file
only because the original language wants inline definitions after the class body; the
split is arbitrary and a rebuild should fold them into the type.

## Accessors

**Contract** — `corpse`, `enemy`, `explosive`, `location`, `member`, `memory` and `brain`
each return the corresponding subordinate. Each asserts the subordinate was built; a
missing one is a construction bug, not a runtime condition, so the check exists only in
checked builds.

**Contract** — `cName` returns the constant text `agent_manager`.
