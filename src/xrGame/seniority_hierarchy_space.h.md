# src/xrGame/seniority_hierarchy_space.h

> Shared helpers for the four-level command hierarchy: a number-to-text formatter for assertion messages, and a fixed-vector filler.

**Needs** — [`xrCore/xrstring.h`](../xrCore/xrstring.h.md)
**Used by** — [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md) · [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`seniority_hierarchy_holder_inline.h`](seniority_hierarchy_holder_inline.h.md) · [`squad_hierarchy_holder.cpp`](squad_hierarchy_holder.cpp.md) · [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`squad_hierarchy_holder_inline.h`](squad_hierarchy_holder_inline.h.md) · [`team_hierarchy_holder.cpp`](team_hierarchy_holder.cpp.md) · [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md)
**Tier floor** — T3: two trivial helpers; nothing here constrains the tier

## Purpose

The command hierarchy is four nested registries — seniority holds teams, a team holds
squads, a squad holds groups, a group holds members — each written as the same shape with
a different capacity. This header is the vocabulary those four files share. It exists only
so the four do not each define the same two helpers.

A rebuild has no reason to keep it: both helpers are one-liners in most languages.

## State

`Stateless.`

## Exported units

**`to_string(number)`** — renders an unsigned number in base ten as an interned string.
Used *only* to build the text of the assertion that fires when a caller asks for a team,
squad or group index beyond the registry's capacity. It exists because the assertion
macro takes a message string, not a format.

**`assign_svector(container, count, value)`** — resizes a fixed-capacity vector to a
given length and writes the same value into every slot. The four registries call it at
construction to fill themselves with "no holder yet", which is what makes the
allocate-on-first-use pattern in the holders safe.

**Notes** — the file also defines a compilation switch declaring that the squad level
tracks a leader. It is defined unconditionally here and never undefined, so the leader is
always present; a rebuild should treat the leader as an unconditional part of a squad and
drop the switch.
