# src/xrServerEntities/object_factory_script.cpp

> Publishes the class registry to the script layer, and lets scripts add their own classes to it.

**Needs** — [`object_factory.h`](object_factory.h.md) · [`object_item_script.h`](object_item_script.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: name lookup and table publication across a language boundary.

## Purpose

Two directions across the script boundary. Outward: the whole registry is published as an
enumeration mapping every class's script name to its index in the sorted table, so a script
can write `clsid.wpn_ak74` and compare it against an entity's own class number. Inward: a
script can register a *new* class identifier whose constructors are Lua functions, which is
how mods add entity types without touching the engine.

## `register_script_class` (paired form)

**Contract** — takes the names of two Lua classes (client and server), the eight-character
tag as text, and the script name to publish. Resolves each name in the global script
environment; if either does not resolve to a constructible value, logs an error and adds
nothing — a bad registration is not fatal, because it must not take a whole mod down at
load. On success, converts the tag text to the 64-bit identifier and adds a script-backed
entry.

## `register_script_class` (single form)

**Contract** — the same with one class name, used when one Lua class plays both halves.
Both constructor slots point at it.

## `register_script_classes`

**Contract** — the hook that brings the script side up. On a dedicated server, which runs
with no script engine, it does nothing; otherwise it forces the artificial-intelligence
space to exist, which is what actually constructs the script engine and runs the shipped
registration scripts. A one-line function whose whole content is a condition and a
side-effecting reference — an artefact of the service-locator design, and a rebuild with
explicit wiring deletes it.

## `register_script`

**Contract** — publishes the enumeration. Sorts the table, then emits one enumerated
constant per entry: the entry's script name bound to its position in the sorted table.
Runs after every registration, native and script, has happened.

```text
FUNCTION publish_class_enumeration()
  actualize()                       # sort by class identifier; position becomes the number
  table = empty enumeration named "clsid"
  FOR EACH entry, position IN registry
    table[entry.script_name] = position
  install table into the script global environment
```

**Invariants** — the numbers a script sees and the numbers a server record reports for
itself come from the same sorted table in the same process, so they always agree. They are
*not* stable across builds and must never be persisted. See
[`object_factory_inline.h`](object_factory_inline.h.md) for why that is safe.

**Notes** — the enumeration is attached to a dummy type that exists only to carry it. That
is a quirk of the binding library's declarative form, not a decision; a rebuild publishes a
plain table.

## `script_register` (the factory's own export)

**Contract** — exposes the registry object itself to scripts with its two registration
methods, so a mod's startup script can call `object_factory:register(...)`.
