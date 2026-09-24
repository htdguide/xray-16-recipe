# src/xrGame/shootingObject_dump_impl.cpp

> Writes a firing object's currently effective ballistics back out as a configuration section, for the balance-tuning tools.

**Needs** — [`ShootingObject.h`](ShootingObject.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field-by-field write into the configuration writer

## Purpose

The multiplayer balance tooling round-trips weapon parameters: it reads a configuration
section, the game applies it, and this writes back what the object is *actually* using.
It is a separate file from the rest of the firing behaviour so that the tools can link it
without the shooting implementation, which is the only reason the split exists.

## State

`Stateless.`

## `dump_active_params`

**Contract** — given a section name and a configuration writer, writes ten keys into that
section describing the object's live firing parameters. Does not read anything back and
does not validate the section name; an existing section is extended, not replaced.

The keys written, and what each is:

| key | meaning |
|---|---|
| `hit_power` | damage, as a four-component vector — one entry per game difficulty |
| `hit_impulse` | the physical impulse a hit transfers |
| `bullet_speed` | muzzle velocity |
| `max_distance` | the range past which the projectile is dropped |
| `disp_base` | the base cone of fire before any per-shot growth |
| `sil_hit_power`, `sil_hit_impulse`, `sil_bullet_speed`, `sil_disp_base` | the four multipliers a fitted silencer applies to the corresponding value above |

**Invariants** — the key names are exactly the names the configuration *reader* uses, so
the output is loadable as input. That round-trip is the file's whole contract; renaming a
key here without renaming it in the reader breaks the tools silently.

**Notes** — damage being a four-vector rather than a scalar is the load-bearing detail
here: difficulty is applied by selecting a component, not by scaling, so a rebuild cannot
collapse it to one number and a difficulty multiplier.

The shot-time counter is written in the source but commented out — it is per-shot state,
not a tuning parameter, and dumping it would have made the output unloadable as a section.
