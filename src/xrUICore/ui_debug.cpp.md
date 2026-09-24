# src/xrUICore/ui_debug.cpp

> The development-only inspector for the widget tree — it hosts an immediate-mode panel showing every live window hierarchy, draws their rectangles over the running game colour-coded by focus role, and lets a developer mutate a widget's geometry and flags in place.

**Needs** — [`ui_debug.h`](ui_debug.h.md) · [`ui_base.h`](ui_base.h.md) · [`ui_styles.h`](ui_styles.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`ui_debug.h`](ui_debug.h.md)
**Tier floor** — T3: a registry and a per-frame panel. It is on no budget; it is compiled out
of the shipping build entirely.

## Purpose

A retained widget tree with no authored z-order, positions relative to parents, and
visibility that depends on every ancestor is genuinely hard to reason about from the layout
files. This file is the answer: a live tree view of every registered root, rectangles drawn
over the game, and an editable property sheet for whatever is selected.

It is entirely optional. A rebuild may drop it — but the two questions it answers ("which
widget is actually under the cursor" and "why can the gamepad not reach this button") have no
other answer in this design, so dropping it means answering them by reading XML.

## State

```text
RECORD Debugger
  roots    : list<Debuggable>    # registered top-level trees; borrowed, not owned
  state    : DebugState

RECORD DebugState
  selected     : optional<Debuggable>   # whose properties are shown
  new_selected : optional<Debuggable>   # the click this frame, applied after the tree walk
  examined     : optional<Debuggable>   # hovered inside the tree view this frame
  settings     : DebuggerSettings

RECORD DebuggerSettings
  colors       : { normal, normal_hovered, examined, focused,
                   focusable_valuable, focusable_valuable_hovered,
                   focusable_non_valuable, focusable_non_valuable_hovered,
                   direction_arrow, direction_text }
  draw_rects   : bool
  colored_rects: bool     # replace role colouring with a per-widget identity hash
```

**Invariants**

- The registry borrows. A debuggable removes itself on destruction, and the removal also
  clears any of the three state pointers that name it — otherwise the next frame would
  dereference a dead widget.
- Selection changes are recorded into `new_selected` during the tree walk and applied after,
  because changing the selection mid-walk would change the highlighting of nodes already
  drawn this frame.
- `examined` is reset by whoever borrows it. The focus system's sub-tree deliberately saves
  and restores it so that the direction overlay is only drawn when hovering inside the *focus*
  tree, not the main one.

## `CUIDebuggable`

**Contract** — the interface a type implements to appear in the inspector. Three demands:
report a human-readable type name, emit a tree node (returning whether it was expanded), and
emit a property sheet. Registration is explicit and symmetric; destruction always
unregisters, so a type that registers need not remember to.

**Notes** — the type-name method is implemented by every widget as a literal string, giving
the tree view the concrete type of a node when the static type is only "window". In the
original this substitutes for runtime type information that the shipping build disables; a
rebuild with reflection gets it for nothing.

## `CUIDebugger::Register` / `Unregister` / `SetSelected`

**Contract** — append to or remove from the root list, clearing any stale state pointers.
Both are no-ops in the shipping build. `SetSelected` sets both selection fields together,
which is how an external caller (a console command, say) selects a widget without going
through a click.

## `CUIDebugger` construction

**Contract** — routes the overlay toolkit's allocations through the engine's allocator,
binds it to the engine's overlay context, and installs the default settings.

**Notes** — the allocator handoff is the one thing here that is not optional if the overlay
toolkit is kept: the engine expects every allocation in the process to be attributable, and
a third-party toolkit with its own allocator defeats the memory accounting. See
[Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator).

## `on_tool_frame`

**Contract** — draws the inspector panel for one frame, if it is open. Structure:

```text
FUNCTION on_tool_frame(dbg)
  IF panel is closed THEN RETURN
  menu bar:
     toggle "draw rects"
     options submenu: reset settings, toggle identity colouring, edit every role colour
     styles submenu:  pick a UI style by name (applies immediately), force a UI reload
  two-column table:
     left  — for each registered root, walk its tree; after each root, apply a pending
             selection change
     right — if something is selected, offer a "break here" button (inert without a
             debugger attached) and then that object's property sheet
```

**Notes** — the style submenu lives here rather than in a settings screen because changing
style during development means reloading every screen, and this panel is where a developer
already is. It calls straight into
[`ui_styles.cpp`](ui_styles.cpp.md).

## `reset_settings`

**Contract** — installs the default colour scheme and flags. The scheme is a legend, and
each colour is chosen to be read at a glance against the game:

```text
normal                  blue,   slightly transparent   # any window
normal hovered          blue,   opaque                 # ... under the cursor
examined                cyan                           # hovered in the tree view
focused                 gold                           # holds navigation focus
focusable, eligible     green                          # reachable by the gamepad now
focusable, not eligible red                            # registered but unreachable
direction arrow         white   / text black           # the navigation overlay
```

Rectangles are drawn by default; identity colouring is off by default.

**Notes** — transparency distinguishes "hovered" from "not" within each role, so hover and
role are two independent readings of the same rectangle.

## `apply_setting` / `save_settings` / `estimate_settings_size`

**Contract** — the panel's settings persist across runs through the overlay toolkit's own
settings file, as one `Key=Value` line per field: the boolean as a decimal, each colour as a
hexadecimal word. Reading parses each known key and silently ignores anything else, so an
older or newer settings file loads without failing. Writing emits every field. The size
estimate reserves the buffer up front.

**Notes** — the size estimate is written out field by field with the literal key strings, so
that adding a field forces the author to update it. That is a maintenance hazard rather than
a decision: a rebuild serialises to a structured format and the estimate disappears. Nothing
depends on the format — it is development-only state.
