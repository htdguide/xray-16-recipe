# src/editors/xrWeatherEditor/property_container_converter.cpp

> Makes a nested object expand into a sub-grid, and forces its rows back into the order the engine declared them.

**Needs** — [`property_container_converter.hpp`](property_container_converter.hpp.md) · [`property_container.hpp`](property_container.hpp.md)
**Used by** — [`property_container_converter.hpp`](property_container_converter.hpp.md)
**Tier floor** — T3: it reorders a row set and returns a constant string.

## Purpose

A grid row whose value is another editable object should expand in place into that object's rows. This makes that happen, and does one more thing that matters as much: it restores the declaration order the grid's default alphabetical sort would have destroyed.

## State

`Stateless.`

## `properties_of`

**Contract** — asks the grid's own machinery for the rows of the nested object, then re-sorts them into the container's declaration order.

```text
FUNCTION properties_of(context, value) -> list<Row>
  rows = default_rows_of(value)
  container = the container being expanded
  RETURN rows sorted into the order of container.ordered_properties
```

**Invariants** — sorting by an explicit name list is how the grid accepts an order at all, which is why [`property_container`](property_container.cpp.md) keeps `ordered_properties` and why row names are made unique there. The two mechanisms are one design: **names must be unique because the order is expressed as a list of names**.

## Multiple selection

**Contract** — when several objects are selected at once, the grid hands a *set* of containers rather than one. The converter takes the first and uses its row order for all of them.

**Notes** — the source says outright that this should be the *intersection* of the selected objects' rows and is not. The consequence is real: selecting two keyframes of different kinds shows the first one's rows, and editing a row that the second does not have writes into a binding that does not exist for it.

In practice the editor's multiple selection is only offered over homogeneous lists, so it does not fire. A rebuild that keeps multi-select should compute the intersection; one that does not should refuse the selection.

## Text rendering

**Contract** — a container renders as a fixed ellipsis when collapsed. Same decision, and same reasoning, as [`property_collection_converter`](property_collection_converter.cpp.md): the summary line of a composite value is not worth computing on every repaint.
