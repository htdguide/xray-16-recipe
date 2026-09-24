# src/xrGame/agent_explosive_manager_inline.h

> Construction and accessors for the squad's explosive manager.

**Needs** — [`agent_explosive_manager.h`](agent_explosive_manager.h.md)
**Used by** — [`agent_explosive_manager.h`](agent_explosive_manager.h.md)
**Tier floor** — T3: three accessors

## Purpose

The trivial half of the explosive manager. Everything that decides anything is in
[`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md).

## State

Adds nothing.

## Construction

**Contract** — binds the squad brain this manager belongs to, permanently; requires it to
be present.

## `object` / `explosives`

**Contract** — the squad brain (binding required) and the pending list. Both are
protected: unlike the corpse manager, this manager does not publish its queue, and nothing
outside it reads the list.
