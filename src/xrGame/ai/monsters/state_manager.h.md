# src/xrGame/ai/monsters/state_manager.h

> The interface a creature's body sees its brain through — the only part of the state machinery that is not a template, and therefore the only part a creature can hold without knowing what kind of brain it has.

**Needs** — [`state_defs.h`](state_defs.h.md) · [`control_com_defs.h`](control_com_defs.h.md)
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_debug.cpp`](basemonster/base_monster_debug.cpp.md) · [`base_monster_script.cpp`](basemonster/base_monster_script.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`base_monster_think.cpp`](basemonster/base_monster_think.cpp.md) · [`monster_state_manager.h`](monster_state_manager.h.md)
**Tier floor** — T3: an interface declaration

## Purpose

Every creature in chapter 24 owns exactly one brain, and the base creature type must be able
to drive it, tear it down and query it without naming the creature's own type. Since the state
node is a template parameterised by the creature, a creature holding its brain as a state node
would be holding a type that names itself. This interface breaks that: the brain is stored as
this non-generic surface, and the concrete brain — which *is* parameterised by the creature —
implements it alongside the node machinery.

It is a pure interface. Everything it declares is a demand on the implementor, so the whole
page is contract.

## The demands

**`update`** — advance the brain by one tick. Called from the creature's think step, after
perception has been refreshed and before motion is committed. The implementor is expected to
guard its own preconditions (alive, not frozen); the caller guards none of them.

**`reinit`** — the creature was respawned, or a save was loaded. Must reach every node in the
tree, not only the running branch, and must release anything any node reserved in a previous
life.

**`critical_finalize`** — the creature is being torn down or forcibly stopped mid-behaviour.
Distinguished from a behaviour completing: reservations are released, but nothing is treated
as having succeeded.

**`force_script_state`** and **`execute_script_state`** — the script layer's override. A script
names a state identifier, and from then until the override is cleared the brain runs that state
instead of the one its selector would have chosen. The two are separate calls because forcing
happens when the script says so and executing happens on the creature's own tick; a rebuild
that merges them loses the ability to set a state outside the tick. This is the chapter's only
inbound path from [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer),
and the identifiers a script may name are frozen by that.

**`remove_links`** — an entity is being destroyed; drop every reference to it held anywhere in
the tree. Incidental to a language with raw references; see [`state.h`](state.h.md).

**`get_state_type`** — name the deepest running node, for the debug overlay and for creature
code that branches on what its own brain is doing (the base creature's think step, for
instance, treats "moving to a restrictor" specially).

**`check_control_start_conditions`** — veto point. A motion-control component asks before
seizing the creature — for a jump, a threaten animation, an anti-aim step — and the running
behaviour may refuse. Answering conservatively (always permitting) makes creatures interrupt
their own attacks; answering too strictly makes special moves never fire.

**Notes** — the destructor is declared pure and then given a body, which is the C++ way of
saying "this type is abstract but still has cleanup to run". A rebuild with explicit interface
declarations expresses it directly and loses nothing.
