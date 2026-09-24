# src/xrGame/script_monster_action_inline.h

> The two behaviour-naming constructors of the monster action channel.

**Needs** — [`script_monster_action.h`](script_monster_action.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

Bodies for two of the declarations in
[`script_monster_action.h`](script_monster_action.h.md). The contracts are documented
there.

## Exported units

- construct from a behaviour — store it and clear the channel's completion flag.
- construct from a behaviour and a target — the same, then set the target.

**Notes**

Both constructors clear the completion flag explicitly rather than relying on the base's
default, because a channel built with a behaviour is by definition not yet finished; the
default-constructed channel is left alone so that an action with no monster channel does
not appear perpetually pending.
