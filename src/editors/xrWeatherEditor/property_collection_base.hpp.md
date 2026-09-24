# src/editors/xrWeatherEditor/property_collection_base.hpp

> Declares the editable-list binding and attaches the two presentation pieces every collection row uses.

**Needs** — [`property_collection_base.cpp`](property_collection_base.cpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`property_collection_editor.hpp`](property_collection_editor.hpp.md) · [`property_collection_converter.hpp`](property_collection_converter.hpp.md)
**Used by** — [`property_collection.hpp`](property_collection.hpp.md) · [`property_collection_base.cpp`](property_collection_base.cpp.md) · [`property_collection_converter.cpp`](property_collection_converter.cpp.md) · [`property_collection_getter.hpp`](property_collection_getter.hpp.md)
**Tier floor** — T1: declares a managed type over an unmanaged collection interface.

## Purpose

The surface implemented in [`property_collection_base.cpp`](property_collection_base.cpp.md), plus one thing the implementation file does not carry: the declaration that **every collection row is rendered by [`property_collection_converter`](property_collection_converter.hpp.md) and edited by [`property_collection_editor`](property_collection_editor.hpp.md)**.

That attachment is a real decision and it lives only here. It is repeated verbatim on both concrete subclasses, because the attachment is read off the value's own type rather than inherited — which is why the same two lines appear three times in this directory.

## The exported units

- **`property_collection_base`** — abstract. Implements the grid's value interface and the standard indexed-collection interface over an engine-side collection. Its one abstract operation is *which* collection, answered by [`property_collection`](property_collection.hpp.md) and [`property_collection_getter`](property_collection_getter.hpp.md).
- **`create`** — make a new element and return its container, without inserting it.
