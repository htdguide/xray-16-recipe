# src/xrGame/script_game_object_script_trader.cpp

> Adds the trader presentation methods to the game object's script registration.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_game_object_script.cpp`](script_game_object_script.cpp.md)
**Tier floor** — T2: registration data only

## Purpose

The game object facade's script surface is too large to register in one translation unit,
so the registration is itself split into a chain of functions, each taking the
half-built class declaration, adding its share of methods, and handing it back. This is the
smallest link in that chain; the chain is assembled in
[`script_game_object_script.cpp`](script_game_object_script.cpp.md).

## `script_register_game_object_trader`

**Contract** — adds five methods and returns the declaration for the next link:

```text
set_trader_global_anim(name)
set_trader_head_anim(name)
set_trader_sound(sound_name, anim_name)
external_sound_start(sound_name)
external_sound_stop()
```

**Notes**

The chain exists purely because building one declaration of this size exceeded what the
compiler would accept in a single unit. It is the clearest case in the recipe of a file
that is an artifact of the implementation language: a rebuild registers the surface in one
place and has no equivalent of any link in the chain.
