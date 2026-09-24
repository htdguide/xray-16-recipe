# src/editors/xrWeatherEditor/ide_impl.hpp

> Declares the editor's implementation of the engine-facing interface — the class the host talks to.

**Needs** — [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`ide_impl.cpp`](ide_impl.cpp.md) · [`window_ide.h`](window_ide.h.md) · [`xrCore/fastdelegate.h`](../../xrCore/fastdelegate.h.md)
**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`ide_impl.cpp`](ide_impl.cpp.md) · [`property_collection_editor.cpp`](property_collection_editor.cpp.md) · [`window_view.cpp`](window_view.cpp.md) · [`window_weather_editor.h`](window_weather_editor.h.md)
**Tier floor** — T1: it declares a type that straddles the managed/unmanaged boundary, holding a garbage-collected window from an unmanaged object.

## Purpose

The surface declaration for the class implemented in [`ide_impl.cpp`](ide_impl.cpp.md). It is the editor library's answer to [`ide_base`](../../Include/editor/ide.hpp.md).

## The exported units

- **`ide_impl`** — the editor root. Holds the host's engine facade, the main window, a paused flag and an in-idle flag.
- **`window`** (set and get) — the main window, assigned just after construction because the window's own constructor needs the already-constructed root.
- **`begin_idle` / `advance_one_frame` / `end_idle` / `in_idle`** — the idle bracket driven from [`entry_point.cpp`](entry_point.cpp.md).
- **`main_handle` / `view_handle`** — the two native window handles the engine binds to.
- **`environment`** — forwards the host's weather system.
- **`run` / `on_load_finished` / `pause`** — the application loop and its two state changes.
- **`create_property_holder` / `destroy`** — property-holder lifecycle.
- **`environment_levels` / `environment_weathers`** — bind a prepopulated holder to a named browser panel.
- **`weather_editor_setup`** — install the timeline's four pull callables.

## Notes

The class holds its window through a handle that keeps a garbage-collected object alive from unmanaged code. That is the incidental face of a real decision: **the editor root is an unmanaged object (the host must be able to hold a pointer to it) that owns a managed one**. A rebuild in one language deletes the distinction; a rebuild that keeps a two-language split meets the same problem and must answer it the same way — one side's collector must be told that the other side holds a reference.
