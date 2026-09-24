# src/editors/xrWeatherEngine/editor_environment_thunderbolts_collection.cpp

> One named set of thunderbolts — stored as a section whose keys are the members and whose values are nothing.

**Needs** — [`editor_environment_thunderbolts_collection.hpp`](editor_environment_thunderbolts_collection.hpp.md) · [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) · [`editor_environment_thunderbolts_thunderbolt_id.hpp`](editor_environment_thunderbolts_thunderbolt_id.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_thunderbolts_collection.hpp`](editor_environment_thunderbolts_collection.hpp.md)
**Tier floor** — T2: it is the engine's thunderbolt collection, extended.

## Purpose

A keyframe names a set, not a strike, so that a storm can vary. This file defines what a
set is on disk and in the grid.

## State

See [`editor_environment_thunderbolts_collection.hpp`](editor_environment_thunderbolts_collection.hpp.md).

## The collection record on disk

```text
RECORD CollectionSection      # one configuration section per set
  section name : text         # the set's name, as keyframes reference it
  <member>     : (empty)      # one key per member thunderbolt, value unused
  ...
```

**Notes** — **A set is a section whose keys are its members.** That is a different encoding
from the comma-separated list the ambients and sound channels use for the same job, in the
same format, in the same model — and both are frozen. The key-per-member form has the
advantage that a name may contain a comma and the disadvantage that it cannot hold
duplicates or an order; the engine picks at random, so neither matters.

A rebuild must emit both encodings, one per file, and should not try to unify them.

## `load` and `save`

```text
FUNCTION load(config)
  FOR EACH key IN config.section(id)
    append new ThunderboltId(manager, key) TO entries, registered with collection
    append manager.description(config, key) TO palette

FUNCTION save(config)
  FOR EACH entry IN entries
    config.write_text(id, entry.name, "")     # the key is the datum; the value is empty
```

**Contract** — reading builds the editable names and the resolved palette side by side.
Writing emits one empty-valued key per entry.

**Invariants** — resolving a member fails hard on an unknown name, which is why
thunderbolts must load before collections.

**Notes** — Two parallel lists again, built once and never reconciled — the same pattern,
and the same consequence, as the ambients' two lists.

## `fill`

```text
FUNCTION fill(collection : PropertyCollection)
  property_holder = editor.create_property_holder(id)      # NOTE: collection not passed
  add "id"           (text, filtered through the manager's collection naming rule)
  add "thunderbolts" (the entries collection)
```

**Contract** — two rows: the set's name, and its member list.

**Notes** — **The parameter is ignored.** Every other `fill` in this module passes the
collection and itself to the holder, which is what tells the grid that this object is an
element of an editable list and gives it its add, remove and reorder commands. This one
does not — so a thunderbolt set appears in the grid without them, and cannot be removed or
reordered from its own page.

The manager still holds it in an editable collection, so the list-level commands exist one
level up; the effect is an inconsistency in the interface rather than a lost capability.
It reads as an oversight, and a rebuild should pass both.

Teardown clears the palette before releasing the entries, so no resolved record outlives
the set that referenced it.
