# src/Include/editor

> The two-way contract between the weather editor's engine host and its separately built editor library — two modules, two implementation languages, four headers.

## What this module is responsible for

The weather editor is a separate application built on the same engine core, and it is split across a module boundary that is also a language boundary: an unmanaged *engine host* that owns the level, the renderer and the weather data, and a managed *editor library* that owns the windows, the menus, the property grid and the timeline. The host loads the library at run time.

These four headers are everything the two halves may say to each other. Nothing else crosses.

They sit in chapter 4 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order) with the rest of the shared interfaces, but their consumers are chapter 29 — the editors — and only those. A rebuild that does not want the weather editor can delete this directory without touching anything else.

## Where it sits and what it rests on

It rests on nothing but the core types. It names the engine's weather system and its interned string type by forward declaration only, which is what lets a module written in a different language include these headers at all.

## The load-bearing ideas

**Control is inverted.** The editor library owns the window and the message loop; the engine host owns a child surface and a single "advance one frame" call. The editor's idle handler pumps the engine in a tight loop until a window message arrives. A rebuild is free — and encouraged — to invert this back, making the editor a panel drawn by the engine's own debug overlay; what survives either arrangement is the *data* the two halves exchange, which is what these twins document.

**Two window handles, not one.** The graphics device is created against a child window, so the editor's docked panels and toolbars are ordinary widgets outside the rendered surface.

**Properties are bindings, not values.** The engine describes an editable object to the grid as a list of typed, categorized fields each bound to a live getter and setter — or directly to the field. The grid reads on paint and writes on edit. There is no copy, no synchronization, and no apply step.

**The editor reads the weather structure on demand and never mirrors it.** The timeline pulls the list of cycles and keyframes every time it paints. Adding or renaming a keyframe therefore needs no notification protocol, and editor/engine divergence is structurally impossible.

**Undo is a reload.** There is no command history anywhere in this design; reverting means re-reading the configuration from disk at one of four granularities. A rebuild should put a history on the host side, where the mutations happen.

**Records that cross the boundary declare their layout.** The two small numeric records in the property contract exist, rather than the engine's own vector types being reused, because the two modules are separately compiled in different languages and neither may assume the other's layout rules. A rebuild that merges the two halves deletes them.

## The twins

| File | Role |
|---|---|
| [`interfaces.hpp`](interfaces.hpp.md) | The two exported symbols of the editor library: construct the editor, destroy it |
| [`engine.hpp`](engine.hpp.md) | What the editor may ask of the engine: pump a frame, take a window message, read and write the weather being edited |
| [`ide.hpp`](ide.hpp.md) | What the engine may ask of the editor: two windows, the application loop, property holders, and the timeline's pull-based data sources |
| [`property_holder_base.hpp`](property_holder_base.hpp.md) | The property-grid schema: typed, categorized, live-bound fields, and editable collections of them |
