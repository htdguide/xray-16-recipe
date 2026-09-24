# src/editors — chapter 29

## What this chapter is responsible for

One tool: the **weather editor**. It is a separate application built on the same engine core, and its entire purpose is authoring the time-of-day and weather keyframe set that the engine interpolates at run time.

It is not a level editor and not a general-purpose tool platform. It loads a level, runs it, shows it in a panel, and lets an author scrub through a day while editing the values the sky is made of — seeing every change in the rendered view on the next frame.

Two things make the chapter worth reading even for a rebuild that will never build this tool.

**The editor interface seam.** The engine exposes a small abstract surface so a tool can drive it: what a tool may do to a running engine, and what the engine demands back. That surface is four headers, documented in [`src/Include/editor`](../Include/editor/README.md), and it is the only place in the repository where the engine is treated as something other than an application.

**The weather data model.** The file this tool writes is frozen — the shipped game reads it. So the document model, the editing operations, and the on-disk layout are load-bearing regardless of what the tool that edits them looks like. That content lives in [`xrWeatherEngine`](xrWeatherEngine/README.md), and specifically in [`editor_environment_weathers_time.cpp`](xrWeatherEngine/editor_environment_weathers_time.cpp.md), which is the file that defines what a weather keyframe *is*.

## Where it sits and what it rests on

Chapter 29 is the last chapter and nothing rests on it. It rests on [`src/Include/editor`](../Include/editor/README.md) (chapter 4) for the contract, on [`xrCore`](../xrCore/README.md) and [`xrEngine`](../xrEngine/README.md) for the engine it embeds, and on two third-party pieces of its own — a property-bag library that supplies the grid's row descriptions, and a window-docking library.

A rebuild that does not want the weather editor deletes this directory and `src/Include/editor` together, and touches nothing else.

## The shape of the thing

The tool is **one process split into two separately built modules, in two implementation languages**:

- an **engine host** ([`xrWeatherEngine`](xrWeatherEngine/README.md)) that owns the level, the renderer, the running clock and the weather document — which is not a copy of the engine's weather system but *is* the engine's weather system, with every authored record replaced by an editable one;
- an **editor library** ([`xrWeatherEditor`](xrWeatherEditor/README.md)) that owns the window, the message loop, the timeline and the property grid, built on a widget library of its own ([`xrSdkControls`](xrSdkControls/README.md)).

The host loads the library at run time and finds exactly two symbols in it: construct the editor, destroy it.

Per the recipe's rule, the *languages* are incidental. What is not incidental is the **boundary** they create, and the four problems it forces — which any rebuild that keeps a split like this will meet, whatever languages it picks.

## The load-bearing ideas

Read these once; every twin in the chapter assumes them.

### 1. Control is inverted, and it costs

The editor owns the message loop. The engine owns one "advance a frame" call and one "would you like this input event" call, and no loop of its own. The editor's idle handler pumps the engine in a tight loop until a window message arrives.

That is the only way to run a fixed-rate engine inside an event-driven application, and it has one consequence that shows up everywhere: **a modal dialog starves the engine**, because the dialog runs its own loop and the idle notification never arrives. The rendered view freezes whenever a picker or a list editor is open — which, in a tool whose entire value is watching the sky change as you edit, is fatal. It is why [the colour picker is disabled](xrWeatherEditor/property_color_base.cpp.md) despite a complete one existing in the widget library, and why [the list editor repaints the view every time it is dragged](xrWeatherEditor/property_collection_editor.cpp.md).

**A rebuild should invert this back**: the engine owns the loop, and the editor's panels are drawn by the engine's own [debug overlay](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui). Doing so deletes [`xrSdkControls`](xrSdkControls/README.md) entirely, deletes the message-interception path, and removes every symptom above. What survives the inversion is the *data* the two halves exchange, which is what these twins are really about.

### 2. The document is the running world

There is no import, no in-memory model, no commit step. The editor's weather manager *is* the engine's, subclassed; a cycle's keyframe list and the engine's cycle table are the same objects. An edit is visible on the next frame because there is nothing between the edited field and the renderer.

Everything else follows from this: no synchronization pass, no dirty flags, no reconciliation, and no way for the editor and the engine to disagree about the world.

### 3. A property is a binding, not a value

The engine describes an editable object to the grid as a list of typed, categorized fields, each bound to a live getter and setter — or directly to the field. The grid reads on paint and writes on edit. Nothing is copied.

The same decision appears on the widget side as [`IProperty`](xrSdkControls/Controls/Interfaces/IProperty.cs.md), two methods wide, and on the engine side as [`property_holder_base.hpp`](../Include/editor/property_holder_base.hpp.md). It is the reason the property layer is an enormous overload set rather than a data structure — and the recipe's recommendation, stated in that twin, is that a rebuild should describe a property as **data**: identifier, category, description, type, default, constraints, presentation hint, accessor pair. Written that way it is a fraction of the size, and it is serializable.

### 4. The boundary forces four questions

They are incidental in their spelling and permanent in their substance. A rebuild that merges the two halves deletes all four; one that keeps a split must answer them the same way.

**Who frees what.** The module that allocated a thing is the module that frees it — which is why the two entry points take the editor by reference rather than returning it, and why a text string converted across the boundary obliges its caller to free it in one direction and not the other.

**What may cross by value.** Not strings — a display name is written into a caller-supplied fixed buffer. Not the engine's interned text — the intern table lives on one side and the other must ask. Not the engine's vector types — their alignment is chosen for wide-float math, so [two tiny records are declared with an explicit layout on both sides](../Include/editor/property_holder_base.hpp.md) instead.

**When things are released.** One side's objects are collected at an unpredictable moment, possibly after the other side has been torn down. [`property_container`](xrWeatherEditor/property_container.cpp.md) guards against telling a destroyed object that it went away. A rebuild should make release deterministic and delete the class of bug.

**What a shared structure looks like.** A keyframe crosses the boundary for copy and paste as **its own on-disk text form**, because that was the only common currency available. It is a good decision for a bad reason: an author can paste a keyframe into a text file and back, or between two running editors.

### 5. Pull, never push

The timeline asks the engine for the weather cycle list every time it repaints. The grid reads its values on every paint. Trees are cleared and refilled by their source. Nothing in the whole chapter holds a mirror of anything, and there is no notification protocol anywhere — so adding or renaming a keyframe requires no wiring, and editor/engine divergence is structurally impossible.

The cost is losing expansion and selection state on a refresh, and a handful of calls per paint. Both are accepted deliberately.

### 6. Undo is a reload

There is no command history on either side. Reverting means re-reading the configuration from disk at one of four granularities: this keyframe, the target keyframe, this whole cycle, or every cycle. **A rebuild should put a history on the host side**, where the data lives and the mutations happen; the absence of one is the design's most significant limitation and it is structural, not an oversight.

### 7. The frozen file is the real deliverable

Everything else in this chapter can be replaced, redesigned or deleted. The keyframe section layout cannot: the shipped game reads it. Angles are degrees on disk and radians in memory; texture references are extensionless logical paths; a keyframe's section name is the time of day it takes effect, and is also its sort key. See [`editor_environment_weathers_time.cpp`](xrWeatherEngine/editor_environment_weathers_time.cpp.md) for the record, field by field, and [§5 of the requirements](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) for where it sits among the game's other formats.

## The directories

| Directory | Role |
|---|---|
| [`xrWeatherEngine`](xrWeatherEngine/README.md) | The engine half: a running engine whose weather system is editable record by record. Owns the document and writes the files the game reads |
| [`xrWeatherEditor`](xrWeatherEditor/README.md) | The application half: windows, timeline, menus, and the layer that turns the engine's property descriptions into grid rows |
| [`xrSdkControls`](xrSdkControls/README.md) | The widget library: a property grid with drag gestures, a path-addressed tree with filter and search, a colour picker, four numeric inputs, and the five interfaces that keep all of it ignorant of weather |

The contract between the first two is [`src/Include/editor`](../Include/editor/README.md), in chapter 4.

## Where to start

Read [`src/Include/editor`](../Include/editor/README.md) first — four short files that state the whole contract. Then [`editor_environment_weathers_time.cpp`](xrWeatherEngine/editor_environment_weathers_time.cpp.md) for the data model, and [`editor_environment_manager.cpp`](xrWeatherEngine/editor_environment_manager.cpp.md) for how the document is assembled. [`entry_point.cpp`](xrWeatherEditor/entry_point.cpp.md) is the whole inversion in one page. Everything else is the property layer, and [`xrWeatherEditor/README.md`](xrWeatherEditor/README.md) explains why there is so much of it.
