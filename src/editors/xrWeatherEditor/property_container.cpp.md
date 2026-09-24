# src/editors/xrWeatherEditor/property_container.cpp

> One editable object as the grid sees it: an ordered set of described rows, each bound to a live accessor pair — plus the two disambiguation rules that let an object carry duplicate names and categories.

**Needs** — [`property_container.hpp`](property_container.hpp.md) · [`property_holder.hpp`](property_holder.hpp.md) · [`xrSdkControls/Controls/Interfaces/IPropertyContainer.cs`](../xrSdkControls/Controls/Interfaces/IPropertyContainer.cs.md) · [`xrSdkControls/Controls/Interfaces/IProperty.cs`](../xrSdkControls/Controls/Interfaces/IProperty.cs.md) · [`property_container_converter.hpp`](property_container_converter.hpp.md)
**Used by** — [`property_container.hpp`](property_container.hpp.md)
**Tier floor** — T1: its finalizer must decide whether the unmanaged half of the editor still exists before touching it.

## Purpose

The managed face of [`property_holder`](property_holder.cpp.md). The engine describes an object by adding properties to a holder; this is what the grid actually binds to, and it is where a description becomes a set of rows.

Two of its five jobs are mechanical — keep the rows, route reads and writes to the right binding. The other three are the interesting ones: preserve the order the engine described the rows in, and two rules for what happens when two rows want the same name or the same category.

## State

```text
RECORD PropertyContainer
  descriptions_by_spec : map<PropertySpec, Property>   # row description -> live binding
  ordered              : list<PropertySpec>            # in the order the engine added them
  holder               : optional<Holder>              # the unmanaged object behind this
  container_holder     : optional<ContainerHolder>     # the object that owns this container
```

**Invariants** — `ordered` and `descriptions_by_spec` hold the same descriptions; the list carries the order, the map carries the binding. A description is added once — adding it twice is a programming error, asserted. The order is the *engine's declaration order*, and preserving it is the whole reason the list exists alongside the map.

## `add_property`

**Contract** — takes a row description and its live binding, runs the two disambiguation rules below, records both, and appends the description to the grid's own property set and to the ordered list.

```text
FUNCTION add_property(description, binding)
  FAIL WITH "already present" IF description is already recorded
  description.category = disambiguate_category(description.category)
  disambiguate_names(description.name)
  descriptions_by_spec[description] = binding
  grid_properties.add(description)
  ordered.append(description)
```

## The category rule

**Contract** — grid categories are identified by their text. Two genuinely different categories that happen to share a name would merge. The rule: when a new category's name matches an existing one *after stripping leading tab characters*, the new row joins that existing category; when it does not match anything, **every existing category gains one more leading tab**, and the new one is used bare.

```text
FUNCTION disambiguate_category(wanted) -> text
  FOR EACH existing IN ordered
    IF same_category(wanted, existing.category) THEN RETURN existing.category
  FOR EACH existing IN ordered
    existing.category = TAB + existing.category      # push every older category down
  RETURN wanted

FUNCTION same_category(fresh, existing) -> bool
  # `fresh` never carries tabs; `existing` may carry any number
  RETURN fresh == existing with its leading tabs removed
```

**Notes** — this is the file's one genuinely load-bearing trick and it is worth unpacking, because the mechanism is ugly and the decision underneath is sound.

The grid sorts categories alphabetically and has no other ordering control. The engine, however, describes an object's fields in a meaningful order — general first, then colour, then sound, then effects — and an alphabetical sort destroys that. Prefixing with a character that sorts before every printable one turns "how many tabs" into an ordering key: the category with the most tabs sorts first.

So the rule *means*: **categories appear in the order the engine first mentioned them, and a name collision after stripping the ordering prefix is the same category.** A rebuild whose grid accepts an explicit category order writes that order and deletes all of this.

The invariant the code asserts — a freshly supplied category never begins with a tab — is what makes the stripped comparison unambiguous.

## The name rule

**Contract** — when a newly added row's name equals an existing row's name, *every* row with that name (including the older ones) gains a leading tab, repeatedly, so that no two rows in the container end up with the same displayed name.

**Notes** — same mechanism, different purpose: here it is uniqueness, not ordering, because the grid keys rows by name within a category and two identical names collapse into one row. The same object really can declare two fields with the same name — a nested colour's three components, described twice under different parents — and the tab keeps them distinct while remaining nearly invisible on screen.

A rebuild whose rows are keyed by identity rather than by displayed name deletes this rule outright, and should.

## Reads and writes

**Contract** — the grid asks for a row's value on paint and offers a new one on edit; both look the row's binding up in the map and forward. `property_for` answers [`IPropertyContainer`](../xrSdkControls/Controls/Interfaces/IPropertyContainer.cs.md)'s one question and is what the extended grid's gestures route through.

**Invariants** — a lookup that misses is a programming error, asserted, because the description came from this container.

## `clear`

**Contract** — empties the bindings, the ordered list and the grid's own property set, leaving the container reusable.

**Notes** — one of the three collections it clears — a category list — is never populated anywhere, and clearing it on an object that was never filled is the only place it is touched. It is dead state left from an earlier version of the category rule; the live version reads categories off the ordered descriptions. A rebuild deletes it. (In the shipped code it is never constructed, so a `clear` on a fresh container would fault — nothing calls it that way.)

## Disposal

**Contract** — when the container is released, it tells its holder so, **unless the editor library as a whole is already gone**.

**Notes** — the guard is the interesting part. Containers are managed objects and are collected at an unpredictable moment, possibly after the unmanaged editor root has been destroyed at shutdown. Telling a destroyed holder that its container went away would touch freed memory. The guard is a check that the library's single root still exists.

That is the incidental shape of a decision every rebuild with two memory regimes must make: **objects whose release is not deterministic must not call into objects whose release is.** A rebuild in one language never sees it. A rebuild that keeps the split must keep a shutdown flag or an equivalent, and the recipe recommends instead making container release deterministic, which removes the class of bug entirely.
