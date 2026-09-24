# src/editors/xrSdkControls

## What this module is responsible for

The editor's widget library: everything the weather editor's windows are built from that the host toolkit does not already supply. It is a separately built module with no dependency on the engine, on the game, or on any of the repository's other code.

The whole library is a bet that is worth naming, because it shapes chapter 29: **the tool's user interface is built from generic widgets that know nothing about weather, joined to the engine only through five two-method interfaces.** That is why the engine-side property description in [`property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) is so elaborate — it is the only channel through which the engine can tell these widgets anything.

## Where it sits and what it rests on

It rests on a host widget toolkit and one third-party property-bag library, and on nothing in this repository. It sits at the very end of chapter 29's build order: the weather editor's windows reference it, and nothing references them but the host.

A rebuild that puts the editor's panels inside the engine's own [debug overlay](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) — which the recipe recommends — deletes this entire directory and replaces it with immediate-mode widgets, keeping only [the five interfaces](Controls/Interfaces/README.md) as the shape of the engine/tool contract.

## The load-bearing ideas

They are stated once in [`Controls/README.md`](Controls/README.md) and the twins are terse because of it: bindings rather than values, optional capabilities that are asked for rather than assumed, gestures measured in pixels and converted by the property, pull-never-push refreshes, echo-breaking in every paired widget, and forgiving parsing with strict storing.

## The twins

| File | Role |
|---|---|
| [`Controls/`](Controls/README.md) | Every widget: the grid, the tree and its two panels, the colour picker, four numeric inputs, and the five interfaces |
| [`Properties/`](Properties/README.md) | Module identity and the one embedded image |

The directory also carries a build description, and the checkerboard image itself. Neither holds a decision and neither has a twin.

## What the twins record

Two complete features in this library are not reachable from the shipped editor: [the colour picker](Controls/ColorPicker/README.md), which the weather editor's colour rows do not open, and [the tree filter](Controls/TreeViewFilterPanel/README.md), whose text box is never bound to its own filtering code. Both are documented in full, and both are flagged as inert, so a rebuilder comparing behaviour against the original does not go looking for them.
