# src/xrGame/script_entity_action_script.cpp

> Exports the scripted-action type to the script layer under the name `entity_action`.

**Needs** — [`script_entity_action.h`](script_entity_action.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of a scripted action. The names are frozen by shipped
scripts.

## `script_register`

**Contract** — registers a class named `entity_action`, default-constructible and
copy-constructible from another action, with:

```text
set_action(channel)   # eight overloads, resolved by the argument's type:
                      # movement, watch, animation, sound, particle, object,
                      # condition, monster behaviour
move      -> is the movement channel finished
look      -> is the watch channel finished
anim      -> is the animation channel finished
sound     -> is the sound channel finished
particle  -> is the particle channel finished
object    -> is the object channel finished
time      -> has the time limit passed
all       -> the joint completion rule
completed -> the joint completion rule        # same function, second name
```

**Notes**

The channel-completion names collide with the channel *types'* own script names — a script
writes `move(...)` to build a movement channel and `action:move()` to ask whether one
finished. Both spellings ship, so both are frozen.

`set_action` is a single script name dispatching on argument type. A rebuild whose binding
layer cannot overload by argument type must either emulate it or accept that no shipped
script will load, since every one of them uses the single name.
