# src/editors/xrWeatherEditor/window_ide_serialize.cpp

> Remembers the author's window layout between sessions, and puts it back.

**Needs** — [`window_ide.h`](window_ide.h.md) · [`window_view.h`](window_view.h.md) · [`window_levels.h`](window_levels.h.md) · [`window_weather.h`](window_weather.h.md) · [`window_weather_editor.h`](window_weather_editor.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`window_ide.cpp`](window_ide.cpp.md) · [`window_ide.h`](window_ide.h.md)
**Tier floor** — T2: it reads and writes a per-user settings store.

## Purpose

Session persistence, and one decision worth arguing with: **the author's layout is stored
in the operating system's per-user settings store, under the retail game's company and
product names**, not in a file beside the editor.

## State

```text
# the stored tree
<user settings>/Software/<company>/<product>/windows
    editor          : bytes        # the dock layout, as a serialised document
    ide/position    : left, top, width, height
    ide/window_state : int         # 1 = maximised, 2 = normal
    weather_editor/property_grid_current : ...   # per-grid state
    weather_editor/property_grid_blend   : ...
    weather_editor/property_grid_target  : ...
```

**Invariants** — the window state is an integer with exactly two defined values; anything
else is treated as unreachable on restore.

**Notes** — Keying the store by the *game's* company and product names, rather than the
editor's own, means the editor's session lives beside the game's settings and a reinstall
of the game can clear it. That is a real coupling and a rebuild should key on the tool.

Storing a serialised layout document as opaque bytes in a settings value is the other
questionable half: the store is meant for small scalars, and this puts a document in one.
A rebuild writes a layout file next to the tool's configuration and keeps only scalars
here — or keeps nothing here at all.

## `base_registry_key`

```text
FUNCTION base_registry_key() -> SettingsNode
  software = open the user's software node for writing
  company  = software.open_or_create(company_name)
  product  = company.open_or_create(product_name)
  close software, close company
  RETURN product
```

**Contract** — returns the tool's settings node, creating the path if absent. The caller
closes it.

**Notes** — Each level is closed as soon as the next is open, which is a discipline the
store requires and a rebuild will express with scoped ownership instead. What survives is
that **the path is created on first use**, so a first run has nothing to restore and must
not fail.

## `save_on_exit`

```text
FUNCTION save_on_exit()
  windows = product.create("windows")
  stream  = serialise the dock layout AS a document
  windows["editor"] = stream bytes
  position = windows.create("ide/position")
  write left, top, width, height FROM window_rectangle
  windows["ide"].window_state = (maximized ? 1 : 2)
  weather_editor.save(INTO windows)         # the three grids' own state
  close everything
```

**Contract** — writes the dock layout, the restored geometry, the window state and the
weather panel's grid state. Called from the window's close handler, before the engine is
asked to quit.

**Notes** — Note what is *not* saved: which weather cycle and keyframe the author was
editing. Reopening the tool starts at the first cycle by name — see
[`editor_environment_manager.cpp`](../xrWeatherEngine/editor_environment_manager.cpp.md).
That is a gap a rebuild should close; the two names are the cheapest thing here to store
and the most useful to restore.

The window state collapses three possible states into two: minimised is saved as normal,
so a tool closed while minimised reopens visible. Deliberate and right.

## `load_on_create`

```text
FUNCTION load_on_create()
  size = 800 by 600
  window_rectangle = (location, size)
  windows = product.open("windows")
  IF windows EXISTS
    IF "ide" EXISTS
      restore left, top, width, height, each falling back to the current value
      window_rectangle = (location, size)
      window_state = (stored state == 1) ? maximized : normal
    IF "editor" EXISTS
      restore the dock layout FROM its bytes, resolving each panel by name
      RETURN                                    # restored; done

  # first run, or no stored layout
  show view           AS the document
  show levels         DOCKED RIGHT
  show weather        DOCKED RIGHT
  show weather_editor DOCKED RIGHT
  window_state = maximized
```

**Contract** — restores geometry and layout, or falls back to a default arrangement and a
maximised window. Each stored value falls back independently to what is already set, so a
partially-written store still restores what it has.

**Invariants** — the geometry is restored **before** the layout, so panel sizes are
computed against the final frame size rather than the default one.

**Notes** — The default arrangement is the specification of what this tool looks like when
you first open it: **the three-dimensional view as the document, the three panels stacked
down the right, maximised.** A rebuild should reproduce it.

## `reload_content`

```text
FUNCTION reload_content(name : text) -> optional<Panel>
  MATCH name
    "editor.window_view"           -> view
    "editor.window_levels"         -> levels
    "editor.window_weather"        -> weather
    "editor.window_weather_editor" -> weather_editor
    OTHERWISE                      -> none
```

**Contract** — the layout document stores each panel by a stable name; this maps a name
back to the already-built panel. An unknown name is dropped, so a layout saved by a
different version restores what it can.

**Notes** — **Panels are resolved, never created.** That is why the panels must exist
before the restore runs, and it is the right arrangement: a layout file cannot conjure a
panel the application does not have, so a stale layout degrades instead of failing.

The stored names are the panel type names, which makes them a compatibility surface: rename
a panel type and every saved layout loses that panel. A rebuild should use explicit,
stable identifiers.
