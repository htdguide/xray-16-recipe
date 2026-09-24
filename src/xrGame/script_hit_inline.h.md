# src/xrGame/script_hit_inline.h

> The script hit's defaults, its copy, and its bone setter.

**Needs** — [`script_hit.h`](script_hit.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

Bodies for the declarations in [`script_hit.h`](script_hit.h.md). Only the default values
are load-bearing; the rest is field assignment.

## Default hit

**Contract** — a freshly constructed hit is already a usable one: full-strength wound
damage pointing along the world's first axis, blamed on nobody.

```text
power     = 100        # the engine's reference damage unit; an authored weapon's damage
                       # is expressed against the same scale
direction = (1, 0, 0)  # arbitrary but non-zero: a zero direction would divide by zero
                       # in the recipient's impulse maths
bone_name = ""         # "no particular bone"
draftsman = none
impulse   = 100
type      = wound
```

**Notes**

The defaults matter because shipped scripts construct a hit and set only the two or three
fields they care about. `wound` as the default kind means an unset `type` behaves like a
bullet, which is what a script author expects; a rebuild choosing a different default
silently changes how every partially-configured hit resolves against armour.

## Copy construction

**Contract** — field-for-field copy, including the blamed object reference, which is
shared rather than duplicated.

## `set_bone_name`

**Contract** — stores the name; no validation, because the skeleton is not known yet.
