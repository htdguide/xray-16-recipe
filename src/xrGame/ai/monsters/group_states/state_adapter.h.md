# src/xrGame/ai/monsters/group_states/state_adapter.h

> Lets a creature state be written as a plain object with no knowledge of the state tree, by wrapping it in a node the tree understands.

**Needs** — [`state.h`](../state.h.md) · [`basemonster/base_monster.h`](../basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure indirection between two object shapes; nothing here touches memory layout, a device or a format

## Purpose

Chapter 24's states are nodes in a tree: each one may own child states, is selected by its
parent's selector, and answers `initialize` / `execute` / `check_completion`. That machinery
is generic over the creature type, so writing a state means writing against a generic node.

This file offers the other way round. A state *implementation* is written as an ordinary
object that knows only its creature and its own data, and an adapter node forwards the tree's
three questions to it. The gain is that the implementation is not generic over the creature
type and so can be compiled once and written in a single file, while the node it sits inside
still is. The cost is one indirection per state tick and one extra allocation per state.

The two halves are deliberately split: `MonsterStateInterface` is the contract an
implementation satisfies, `MonsterStateAdapter` is the tree node that owns one.

## State

```text
RECORD MonsterStateImplementation        # what an author of a group state writes
  creature           : BaseMonster        # the creature this state drives
  time_state_started : int                # global clock at the last entry into this state
```

The adapter node owns its implementation exclusively — the node's teardown destroys it — so
an implementation may never be shared between two nodes or outlive the node holding it.

## `MonsterStateInterface`

**Contract** — the surface a group-state implementation must provide. Four questions, all
optional except the first:

- `payload` — hands back the implementation's own data block, which the adapter passes to the
  tree node at construction so that the node's generic data slot points at the
  implementation's storage rather than at a second copy. An implementation with no data
  answers nothing.
- `enter` — called once when the parent selects this state. The default records the global
  clock in `time_state_started`; an override that wants the timestamp must keep it.
- `run` — called each tick while this state is selected. Default does nothing.
- `finished` — answered each tick by the parent's selector, which is free to leave the state
  selected while the answer is no. **Default is yes**, so a state that means to persist must
  override it; the default makes a one-shot state the cheapest thing to write.

**Invariants** — `enter` precedes every `run`; `time_state_started` is meaningless before the
first `enter`.

## `MonsterStateAdapter`

**Contract** — a state-tree node, generic over the creature type, that delegates entry,
execution and the completion question to one implementation it owns. Construction takes the
implementation and the creature, and threads the implementation's payload through to the base
node so both halves see one data block. Teardown destroys the implementation.

```text
FUNCTION make_adapter(impl, creature) -> StateNode
  node = new StateNode(creature, impl.payload())   # one data block, two owners' view of it
  node.impl = impl
  RETURN node

FUNCTION adapter.enter()            -> impl.enter()
FUNCTION adapter.run()              -> impl.run()
FUNCTION adapter.finished() -> bool -> RETURN impl.finished()
```

**Notes**

The name and the file's own comment say *encircle state* — this was built for the pack
manoeuvre where several creatures surround one target, which is the only group behaviour that
needs a state written against the group rather than against one creature. Nothing in this file
is specific to encircling; it is the generic half that survived.

`remove_links`, which the tree uses to drop a destroyed entity from every state's stored
references, is *not* forwarded to the implementation — only the base node's own references are
cleared. An implementation that stores a reference to another entity therefore has no way to
learn that entity has gone. That is a real constraint on what a group-state implementation may
remember, not an oversight the rebuilder should copy blindly.
