# src/xrGame/alife_registry_container_space.h

> The four macros that let the composition file be written as a flat list of registries.

**Needs** — _(none)_
**Used by** — [`alife_registry_container.h`](alife_registry_container.h.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md)
**Tier floor** — T4: text substitution with no runtime existence

## Purpose

Nothing in this file survives a rebuild. It defines the vocabulary that
[`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) is
written in: a way to accumulate a list of types one line at a time, by redefining a name
to mean "the previous list with one more entry on the front".

The problem it solves is real, though, and a rebuild must solve it somehow: **the list of
persistent registries must be declared once and consumed three times** — to give the
container a member per registry, to give the save and load their iteration order, and to
give each registry a name the rest of the game can select it by. The original's answer is
a compile-time list built by macro accumulation. A rebuild's answer is a list literal, an
enumeration, a table, or a registration call per registry; any of them is better, and none
of them changes the save format.

One consequence of the accumulate-on-the-front construction is worth knowing before
reading the composition file: the list is built in reverse, and the save walk then
reverses it again, so **the order in the stream is the order the registries are declared
in the composition file, top to bottom.**

## The macros

**Contract** —

- an empty-list marker, the starting value of the accumulating name;
- *append*, which produces a new list from a type and the current list;
- *name a type as a value*, which is how a registry's type is passed to the container's
  selector (as a null pointer of that type — see
  [`alife_registry_container_inline.h`](alife_registry_container_inline.h.md));
- *fetch the accumulated list*, used to reassign the accumulating name.

All four exist only during compilation.
