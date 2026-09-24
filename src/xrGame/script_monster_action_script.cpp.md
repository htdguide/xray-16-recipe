# src/xrGame/script_monster_action_script.cpp

> Exports the monster behaviour channel to the script layer as `act`.

**Needs** — [`script_monster_action.h`](script_monster_action.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of the monster behaviour channel. The name `act` and the
behaviour spellings are frozen by shipped scripts.

## `script_register`

**Contract** — registers a class named `act` with three constructors and one nested
constant table:

```text
act()                      # imposes nothing
act(type)                  # a behaviour with no target
act(type, game_object)     # a behaviour aimed at an object

act.type = { rest, eat, attack, panic }
```

**Notes**

`set_object` is *not* exported, although the channel has it: a script builds the channel
whole and hands it to an action, and retargeting a channel already queued would race the
scheduler. The method exists for engine callers only.

`none` is likewise absent from the exported table. A script that wants "impose nothing"
constructs the channel with no arguments; there is no spelling for the zero behaviour.
