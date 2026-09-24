# src/xrGame/ai/trader — the trader

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
See the [chapter opener](../README.md) for the two kinds of mind in this chapter. The trader
is the third case: a human with no mind at all.

A trader is a person who stands somewhere, turns his head to watch the player, plays an
animation, says a line, and holds an inventory you can buy from. Its think is **empty**. It
has no planner, no state machine, no perception beyond a distance test, no navigation and no
combat. Everything it ever does is either a script telling it to, a dialogue driving it, or
the player opening the trade screen.

It is in this chapter because it is a *human* entity — it inherits the same inventory,
dialogue and script-entity plumbing a stalker does — not because it decides anything.

## What actually happens

**A bone callback watches the player.** The head bone carries a callback that, every time the
skeleton is posed, turns the head toward the player — but only within twenty units. This is
the trader's entire perception and its entire behaviour. It runs on the animation path, not
on the think path, which is why a trader keeps watching you while the game is otherwise doing
nothing about him.

**Animation is two independent channels driven by name.** A global clip and a head clip, each
set by *string*, each with a completion callback, plus a sound attached to a head animation so
that a spoken line and a mouth movement start together. Dialogue drives that pair directly: a
phrase supplies both a sound and a head animation name.

**Trading is a latch with two script callbacks.** Opening the trade screen sets a busy flag
and fires a callback; closing it clears the flag and fires another. Scripts use the flag to
avoid disturbing a trader mid-transaction. Nothing in the engine reads it.

**The script-entity path is the only drive.** The scheduled update branches: if a script holds
the trader, run the script's action queue; otherwise run the think — which is empty. So a
trader that no script is driving does literally nothing each tick except update its inventory
and its sound player.

**Relations are asked of the registry, with one exception.** The trader's answer to "how do I
feel about this entity" defers to the global relation registry for anything that owns an
inventory and is not a creature, and to the base otherwise. A trader has no opinion of its
own about anybody.

## Deliberate refusals

Reading what the trader *declines* to do is the fastest way to understand it. It cannot attach
items to itself. It does not use bolts. The only inventory slot it will accept anything into
is the one holding a personal data assistant. Hit signals and hit impulses are empty bodies:
a trader does not flinch, does not stagger, and does not react to being shot in any way other
than by dying.

Its field of view is a hundred and fifty degrees and its range thirty units, both fixed in
code and both used by nothing, since the trader has no perception system to feed.

## What could not be recovered

- **The artefact-order system is a stub.** The price of an artefact is simply its cost, and
  the routine that would buy one and remove it from a standing order list returns a refusal
  unconditionally. The comments describe a system of generated orders that no longer exists.
- **The scheduling window carries a note that its bounds were changed by hand** and that the
  intended relationship to network latency was broken in the process. The original relationship
  is written in a disabled expression beside it.
- The weapon bone names and the head bone name are string literals in code, so a trader model
  must use the standard skeleton naming.
- The twenty-unit head-watching radius is a bare constant.

## Twins

| Twin | Role |
|---|---|
| [`ai_trader.cpp`](ai_trader.cpp.md) | The trader's whole engine-side behaviour: turn your head toward the player, accept what is put in your hands, and let Lua do the rest. |
| [`ai_trader.h`](ai_trader.h.md) | Declares the trader: a human who talks, holds stock and never thinks. |
| [`ai_trader_script.cpp`](ai_trader_script.cpp.md) | Exports the trader's class identity to the script layer. |
| [`trader_animation.cpp`](trader_animation.cpp.md) | Implements the trader's animation: three callbacks into the script layer, fired whenever a motion or a spoken phrase runs out. |
| [`trader_animation.h`](trader_animation.h.md) | Declares the trader's animation component: a body loop and a head loop, each re-requested from Lua whenever it ends. |
