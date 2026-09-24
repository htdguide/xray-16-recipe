# src/xrGame/ai/monsters/controlled_entity.h

> The interface a creature implements to be *takeable*: the contract between a mind-controlling creature and its thralls.

**Needs** — [`controller/controller.h`](controller/controller.h.md) · [`controlled_entity_inline.h`](controlled_entity_inline.h.md)
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`bloodsucker.h`](bloodsucker/bloodsucker.h.md) · [`boar.h`](boar/boar.h.md) · [`controlled_entity_inline.h`](controlled_entity_inline.h.md) · [`controller.cpp`](controller/controller.cpp.md) · [`dog.cpp`](dog/dog.cpp.md) · [`dog.h`](dog/dog.h.md) · [`flesh.cpp`](flesh/flesh.cpp.md) · [`flesh.h`](flesh/flesh.h.md) · [`pseudo_gigant.h`](pseudogigant/pseudo_gigant.h.md) · [`tushkano.h`](tushkano/tushkano.h.md) · [`zombie.h`](zombie/zombie.h.md)
**Tier floor** — T2: an interface plus a generic implementation mixed into entity classes

## Purpose

An abstract interface, and therefore substantive: what it *demands of an implementor* is the
contract a rebuild must satisfy. A creature that can be taken over by a mind-controller
implements it; the controller discovers the capability by asking an entity whether it
implements the interface at all, which is how the game distinguishes "can be enthralled"
from "cannot" without a per-class table.

The generic implementation that every taker uses is in
[`controlled_entity_inline.h`](controlled_entity_inline.h.md); this file declares both the
interface and the shape of that implementation.

## State

```text
ENUM Task
  follow      # stay with the controller
  attack      # attack what the controller is attacking
  none

RECORD ControlledInfo
  task     : Task
  object   : optional<Entity>   # who to follow or attack
  position : vector             # declared, unused by this layer
  node     : int                # declared, unused by this layer
  radius   : real               # declared, unused by this layer

RECORD ControlledEntity          # the generic implementation's own state
  data       : ControlledInfo
  saved_ids  : { team, squad, group : int }   # the thrall's allegiance before it was taken
  controller : optional<Controller>
```

**Invariants** — the saved allegiance triple is written exactly once, when the thrall is
taken, and restored exactly once, when it is freed. Every path that ends the hold restores
it; there is no path that clears the controller without restoring, which is the invariant
that keeps a former thrall from staying permanently hostile to its own kind.

The position, node and radius of the task record are set nowhere in this slice. A task is in
practice (kind, object).

## The interface

**Contract** — nine operations an implementor must provide.

| Operation | What it must do |
|---|---|
| `is_under_control` | is a controller currently holding this entity |
| `set_data` / `get_data` | replace or reach the current task record |
| `set_task_follow` / `set_task_attack` | set the task kind and its object |
| `set_under_control` | begin the hold: save the allegiance, adopt the controller's |
| `free_from_control` | end the hold: restore the allegiance |
| `on_reinit` | clear the task and the controller at spawn |
| `on_die` | the thrall died while held — tell the controller |
| `on_destroy` | the thrall is being destroyed while held — restore and tell the controller |

**Notes** — the *allegiance swap* is the mechanism, and it is worth stating plainly because
it is not obvious from the names: taking a creature over does not install an AI override. It
changes the creature's team, squad and group to the controller's, so that every existing
relation rule — who is an enemy, who is a friend, who shares sighting information — treats
the thrall as one of the controller's own. The thrall's own brain then does the rest. That
is why the interface is this small.

`on_die` and `on_destroy` differ by one step: destruction restores the allegiance, death
does not. A dead entity's team no longer matters; a destroyed one may be a server record
that outlives the client object.

## The generic implementation

`CControlledEntity` is a mixin parameterized over the entity type it is mixed into, so one
implementation serves every takeable creature class. It holds the task record, the saved
allegiance and the controller handle, and is bound to its host by an explicit initialization
call rather than by construction — the host constructs first and connects afterwards.

`get_controller` reaches the holding controller, which is how a thrall's own state layer
finds out who to follow.
