# src/Layers/xrRender/r_constants.cpp

> Building one shader's constant table out of several stages' reflections: lookup by name, and the merge that folds a stage into a table without losing another stage's bindings.

**Needs** — [`r_constants.h`](r_constants.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r_constants.h`](r_constants.h.md)
**Tier floor** — T1: the table is searched by name on every pass bind, so the search's shape — a binary search over a sorted array, or a pointer comparison over interned strings — is the file's whole content.

## Purpose

A drawable pass is several shader stages compiled separately, each declaring its own constants, many of them the *same* constant. The table has to end up with one entry per name carrying every stage's binding. This file is that consolidation, plus the two lookups everything else uses.

## `get` — by raw name

**Contract** — Find a constant by its name, optionally restricted to a kind. Returns nothing when absent. Does not allocate.

```text
FUNCTION get(name : text, kind : optional) -> optional<Constant>
  IF kind is not given
    # The table is sorted by name, so this is a binary search.
    candidate = lower_bound(entries, name)
  ELSE
    # A kind-restricted search cannot binary-search: two entries may share a
    # name and differ only in kind (the legacy case), and they are adjacent in
    # an unspecified order. Linear scan.
    candidate = first entry whose name and kind both match

  IF candidate is past the end OR its name differs THEN RETURN none
  RETURN candidate
```

**Notes** — That the kind-restricted form degrades to a linear scan is acceptable only because it is used at *load* time, when a material is being recorded, and never per draw.

## `get` — by interned name

**Contract** — The same lookup, given an already-interned string. Compares by identity rather than by content, so no character comparison happens at all — but the table is sorted by *content*, so identity comparison cannot binary-search and this form is a linear scan.

**Invariants** — Both forms must return the same constant for the same name. The interned form is what the per-draw paths use, because interning happens once at load and the scan is over a table of a few dozen entries with a one-instruction comparison — which measures faster than a binary search with string comparisons.

This is the sort of trade that looks wrong and is right: a rebuild should keep the two forms and should measure before replacing either with a hash map, because the tables are small enough that a hash is not obviously better.

## `merge`

**Contract** — Fold another table — typically one stage's reflection — into this one. Every constant the other table has either extends an existing entry with its stage binding or becomes a new entry. Re-sorts when anything was added. Allocates.

```text
FUNCTION merge(other)
  IF other is none THEN RETURN
  IF other is legacy-compatible THEN this becomes legacy-compatible too

  additions = empty
  FOR EACH source IN other.entries
    # In legacy mode a name is ambiguous, so the match must also agree on kind.
    existing = get(source.name, legacy ? source.kind : any)

    IF existing is none OR (legacy AND existing.kind differs)
      # New constant: copy it whole, bindings and setter included.
      additions.append(copy of source)
    ELSE
      # Existing constant, new stage. Widen the destination word and copy in
      # ONLY the binding for the stage the source declares.
      existing.destination = existing.destination OR source.destination
      REQUIRE existing.kind = source.kind
      existing.binding_for(source.stage) = source.binding_for(source.stage)

  IF additions is not empty
    append them and re-sort the table by name

  # Constant buffers are concatenated per render context without any
  # de-duplication — noted in the original as unfinished.
  FOR EACH context
    append other's buffers for that context to this table's
```

**Invariants**

- Only the binding for the stage the source declares is copied. Copying every binding would overwrite the bindings this table already holds for other stages, which is the exact bug the whole function exists to avoid.
- The kind must agree when an entry is extended. A name used as a float in one stage and an integer in another is a shader authoring error, and the original asserts rather than guessing.
- The re-sort happens once after all additions, not per addition.

**Notes** — The constant-buffer concatenation is flagged in the original as needing a validity check, and it does: two stages declaring the same buffer will each contribute an entry. Nothing downstream de-duplicates, so a buffer may be bound twice. It is harmless and wasteful, and a rebuild should key buffers by their usage tag.

## `clear`

**Contract** — Drop every constant reference and every buffer, per render context. The constants are shared resources; clearing releases this table's claim on them.

## `equal`

**Contract** — Whether two tables describe the same interface: same length, and every entry equal in order. Order matters, which is safe because both are sorted by name. This is what lets the renderer recognise two compiled passes as interchangeable and skip the state change between them.

## Lifetime

Destroying a table unregisters it from the resource manager's table registry, so a later compile of the same shader source does not hand out a dangling table. That registry is what makes identical shader interfaces *share* one table object — which in turn is what makes the equality test above usually a pointer comparison in practice.
