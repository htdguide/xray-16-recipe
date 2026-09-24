# src/xrGame/FryupZone.cpp

> An anomaly class that exists only as a name in the class-identifier table: it inherits everything and overrides nothing.

**Needs** — [`FryupZone.h`](FryupZone.h.md) · [`script_object.h`](script_object.h.md)
**Used by** — reached through its declarations in [`FryupZone.h`](FryupZone.h.md); callers name that, not this file.
**Tier floor** — T3: a named leaf of the entity class hierarchy with no behaviour of its own

## Purpose

The class-identifier table in the shipped game data names more anomaly kinds than the
engine implements distinctly. `CFryupZone` is one of those names: it is a script-driven
object with no engine-side rules, so that a spawn record carrying its class identifier
instantiates *something* and the level loads. All of its behaviour comes from the Lua
side of its base, the scriptable object.

A rebuild that drives its class-identifier table from data will get this file for free
and need not write it at all.

## State

`Stateless.`

## `CFryupZone`

**Contract** — constructs and destroys with no work. Its only nominally overridden
behaviour is the debug-build render hook, which draws nothing.

**Notes** — the empty debug render override is a placeholder where a developer would
drop visualization while tuning; it carries no decision.
