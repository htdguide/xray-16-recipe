# src/editors/xrWeatherEditor/property_collection_converter.cpp

> What a collection row shows when it is not open: an ellipsis, because a list has no one-line value.

**Needs** — [`property_collection_converter.hpp`](property_collection_converter.hpp.md) · [`property_collection_base.hpp`](property_collection_base.hpp.md)
**Used by** — [`property_collection_converter.hpp`](property_collection_converter.hpp.md)
**Tier floor** — T3: one string.

## Purpose

Every grid row must render its value as text, even rows whose value is a list of objects. This says what that text is: a fixed ellipsis, `< ... >`.

## State

`Stateless.`

## Contract

**Contract** — the value converts to text and to nothing else; the text is a constant, independent of the collection's length or contents.

## Notes

The decision is to show *nothing informative*. The plausible alternative — the element count, or the first few display names — was not taken. Given that the display name of a collection element is only obtainable through a buffer-filling call on the engine side (see [`property_collection_editor`](property_collection_editor.cpp.md)), rendering contents on every repaint of every collapsed row would be measurably expensive, so the constant is defensible. A rebuild that can cheaply count should show the count; it reads better and costs nothing.
