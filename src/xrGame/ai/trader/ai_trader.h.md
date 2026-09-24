# src/xrGame/ai/trader/ai_trader.h

> Declares the trader: a human who talks, holds stock and never thinks.

**Needs** — [`CustomMonster.h`](../../CustomMonster.h.md) · [`InventoryOwner.h`](../../InventoryOwner.h.md) · [`script_entity.h`](../../script_entity.h.md) · [`sound_player.h`](../../sound_player.h.md) · [`AI_PhraseDialogManager.h`](../../AI_PhraseDialogManager.h.md) · [`trader_animation.h`](trader_animation.h.md) · [`ai_trader.cpp`](ai_trader.cpp.md)
**Used by** — [`actor_communication.cpp`](../../actor_communication.cpp.md) · [`ai_trader.cpp`](ai_trader.cpp.md) · [`ai_trader_script.cpp`](ai_trader_script.cpp.md) · [`trader_animation.cpp`](trader_animation.cpp.md) · [`script_game_object_trader.cpp`](../../script_game_object_trader.cpp.md) · [`trade.cpp`](../../trade.cpp.md) · [`trade2.cpp`](../../trade2.cpp.md)
**Tier floor** — T3: a composition of four existing mixins with one head-turn rule of its own

## Purpose

Declares the surface implemented in [`ai_trader.cpp`](ai_trader.cpp.md).

The trader is the counterexample to everything else in this chapter. It is a living, named,
inventory-holding, conversation-capable human being, and it has **no brain at all**: no
planner, no behaviour tree, no perception beyond touch, no navigation. Its per-tick think
function is empty. Everything a trader appears to decide is decided by Lua, through the
script-entity mixin and through dialogue.

What it *is*, is the sum of four mixins:

| Mixin | What it brings |
|---|---|
| living entity | health, damage, death, animation, network identity |
| inventory owner | stock, money, trade rules, the character record and reputation |
| script entity | the action queue a Lua script drives it with |
| phrase dialogue manager | conversations, and the reputation effects of conversations |

Plus two things of its own: a head that turns to face a nearby player, and an animation
component driven by dialogue rather than by movement.

## State

```text
RECORD Trader
  busy_now      : bool             # set while a trade screen is open
  sound_player  : SoundPlayer      # owned
  animation     : TraderAnimation  # owned; see trader_animation.h
```

## Exported units

- **the cast battery** — the mixin down-casts every game object exposes. A trader answers
  yes to attachment owner, inventory owner, living entity, entity, game object, physics
  shell holder, particles player and script entity.
- **lifecycle** — construct, load, spawn, destroy, re-initialise, reload, save and restore.
- **the per-tick update** — run the Lua action queue if a script has control, otherwise
  think; see [`ai_trader.cpp`](ai_trader.cpp.md) for what thinking amounts to.
- **the frame update** — advance sounds and, when no script animation is running, the
  dialogue-driven animation.
- **the bone callback and the head-turn rule** — the trader's one piece of autonomous
  behaviour.
- **event handling** — take, drop, buy and sell.
- **trade lifecycle hooks** — trade started and trade stopped, each of which fires a script
  callback.
- **relation lookup** — reputation-based, falling back to the base entity rule.
- **fixed perception parameters** — a 150-degree field of view and a 30-metre range, both
  hard-coded and neither used by any perception the trader actually runs.
- **inventory policy** — the trader may not attach anything, does not use bolts, and the
  only slot it will fill is the personal-data-assistant slot.
- **artefact pricing** — a price query and a purchase, both stubs.
- **dialogue sound** — start and stop a spoken phrase.

**Notes** — most of these are one-line delegations to a mixin, present so that the trader
can be used wherever a game object is expected. The three that carry decisions are the
head-turn rule, the event handling and the inventory policy.
