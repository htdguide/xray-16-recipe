# src/editors/xrSdkControls/Controls/Interfaces

## What this module is responsible for

The five contracts that let the control library stay ignorant of what it is editing. Every one of them is two methods or fewer, and together they are the entire vocabulary between the widgets and whoever supplies their data.

## Where it sits and what it rests on

They rest on nothing but the host toolkit's type for a row description. Everything else in this library and the whole of the weather editor's property layer rests on them.

## The load-bearing ideas

**A property is a binding, not a value.** [`IProperty`](IProperty.cs.md) is read-and-write and nothing else; the grid holds no copies and there is no apply step. This is the same decision stated on the engine's side in [`property_holder_base.hpp`](../../../../Include/editor/property_holder_base.hpp.md), and it is what makes editor/engine divergence structurally impossible.

**Capabilities are asked for, not assumed.** [`IIncrementable`](IIncrementable.cs.md) and [`IMouseListener`](IMouseListener.cs.md) are optional: the grid tests a property for them and skips the gesture if the property does not claim it. Adding a gesture therefore does not change any existing property.

**Gestures are expressed in pixels, quantities in the property's own units.** The grid knows the gesture; only the property knows what a pixel of drag is worth in fog density. Splitting the responsibility that way is what stops a 0-to-1 field and a 0-to-100000 field feeling identical under the same drag.

**Contents are pulled, not pushed.** [`ITreeViewSource`](ITreeViewSource.cs.md) refills a tree that has already emptied itself; there is no incremental update path, so a tree cannot drift from its data.

## The twins

| File | Role |
|---|---|
| [`IProperty.cs`](IProperty.cs.md) | The binding: read the value, write the value |
| [`IPropertyContainer.cs`](IPropertyContainer.cs.md) | The join from a row the grid is drawing to the binding behind it |
| [`IIncrementable.cs`](IIncrementable.cs.md) | Optional: this property may be scrubbed by dragging |
| [`IMouseListener.cs`](IMouseListener.cs.md) | Optional: a double-click on this property opens something |
| [`ITreeViewSource.cs`](ITreeViewSource.cs.md) | Where a tree's contents come from, and how it is asked to refill |
