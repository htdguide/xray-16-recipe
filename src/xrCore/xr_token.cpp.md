# src/xrCore/xr_token.cpp

> Lookup both ways through a name/number table — the mechanism behind every enumerated setting the console and the configuration files spell as a word.

**Needs** — [`xr_token.h`](xr_token.h.md) · [`_std_extensions.h`](_std_extensions.h.md)
**Used by** — [`xr_token.h`](xr_token.h.md)
**Tier floor** — T3: a linear search over a table.

## Purpose

Settings that take one of a fixed set of values — renderer name, texture quality, sound preset, difficulty — are written as words in configuration and shown as words in the console, and are numbers everywhere else. A token table is the mapping, declared next to whatever owns the setting rather than centrally.

## State

Stateless. The tables are static, owned by their callers, and **terminated by an entry with no name** — there is no count.

## `name_for`

**Contract** — Linear scan for the first entry whose number matches; returns its name. Returns the **empty string**, not nothing, when the number is not in the table, so a caller can print the result unguarded.

## `id_for`

**Contract** — Linear scan, comparing names case-insensitively; returns the entry's number. Returns **-1** when nothing matches.

**Invariants** — The two misses are spelled differently: an unknown number yields empty text, an unknown name yields -1. Callers must test the second and rarely test the first.

**Notes** — Case-insensitive lookup is required because the shipped configuration files and the console history are inconsistent about case. The comparison is ASCII-only, in line with the platform assumption about locale-independent folding.

Both searches are linear because the tables are short — a handful of entries to a few dozen — and are consulted on configuration changes, not per frame. A rebuild should keep them linear rather than sorting, because table order is meaningful: the console renders the choices in table order.
