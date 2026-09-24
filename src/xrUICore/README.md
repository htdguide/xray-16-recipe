# src/xrUICore — chapter 15

> The retained-mode widget toolkit. A tree of rectangles in a fixed virtual canvas, built
> from XML by a closed vocabulary of element types, drawn as clipped textured quads, and
> driven by an event model that resolves a pointer position to exactly one widget.

## What this module is responsible for

Everything the player sees that is not the world: the main menu, the inventory, the map, the
trade screen, the dialogue box, the loading screen, the heads-up elements that are widgets
rather than geometry. This chapter is the *toolkit* — the window, the button, the list, the
scroll bar, the text engine, the font set, the cursor. The screens themselves are
[chapter 25](../xrGame/ui/README.md), and most of them are Lua scripts calling into this
chapter's exported surface.

It is emphatically not the debug overlay. That is an immediate-mode toolkit shopped for at
[Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) and it renders
through this chapter's own inspector. The two coexist and share nothing.

## Where it sits

Chapter 15, after the engine. It rests on the core layer for the virtual filesystem, the
configuration format, interned strings and the XML reader; on the engine for the device, the
input pump, the console variables, the font object and the localization string table; on the
renderer interfaces for the primitive stream it emits quads into; and on the script engine
for its exported surface.

It is one half of a declared cycle: the engine owns the frame loop and this module links back
against it to be driven. A rebuild inverts that into abstract ports with adapters.

## The load-bearing ideas

Named once here, so that the two dozen subdirectory pages can be terse.

### The canvas is 1024×768 and nothing else knows the resolution

Every coordinate in every layout document, and every coordinate inside every widget, is in a
fixed virtual canvas. The real back-buffer size enters at exactly one place — the scale
factors in the module's runtime object — and is applied when a quad is converted to screen
pixels. That is why the shipped layouts work at any resolution, and a rebuild that
"helpfully" switches to physical pixels breaks every shipped screen.

The scale is **non-uniform**: a wide display stretches the canvas horizontally and the engine
accepts the distortion, correcting for it in exactly two places — an aspect factor applied to
rotations, and separate widescreen variants of the layout documents, silently substituted
when one exists. Letterboxing instead would look different from the original.

### Position is relative; clipping is not inherited

A window's rectangle is derived from its position, its size and an *alignment mode* that
decides how position is read: as a top-left corner, as a centre, or as an offset from a
screen edge with one coordinate discarded. The absolute rectangle is the sum of the
ancestors' origins with the window's own size re-imposed — so a child is positioned by its
parent but is **neither clipped nor resized by it**. Clipping is an explicit act performed by
the few containers that want it, by pushing a scissor rectangle; the quad emitter intersects
against that rectangle in software, so a partially visible atlas sprite gets correct texture
coordinates rather than a stretched edge.

### Z-order is list position, and hit-test order is its exact reverse

Children are drawn in list order — last attached is on top — and a pointer event is offered
to children in reverse list order, so the topmost-drawn widget gets first refusal. There is
no depth value and no sorting anywhere in the chapter. Everything about overlapping
behaviour follows from those two sentences.

### The event model: position, capture, focus, notification

Four distinct channels, and conflating any two of them is the classic rebuild mistake.

**Pointer events** carry a position, are rebased into each window's own coordinates as they
descend, and stop at the first widget that consumes one. A **capture** overrides position
entirely: a child asks its parent to route everything to it, the request walks up so that
every ancestor holds a link down to the capturer, and the root can then reach it in one hop
per level. Capture is how a drag survives the pointer leaving the widget. When a capture
displaces a previous capturer, the displaced widget is told, so it can abandon its gesture.

**Hover** is recomputed each frame from the cursor position against each window's whole
absolute rectangle, with no regard to overlap — two overlapping widgets can both consider
themselves hovered. Pointer *routing* resolves overlap; hover does not. The shipped layouts
are authored not to overlap, which is why this survives. Hover's timestamp is what delayed
tooltips are measured from.

**Keyboard and text are two channels, not one.** Scancodes arrive on one, composed Unicode
characters on the other (see
[Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)); an edit
box takes navigation from the first and content from the second. Merging them breaks non-Latin
input, which the shipped localizations need.

**Navigation focus** — the gamepad and arrow-key focus — is a separate system again, and it
is *geometric*, not an authored tab order. A registry holds every widget that could take
focus, partitions it each frame into eligible and ineligible against the currently active
tree root, and answers "which widget is next in this direction" by a nearest-candidate search
in one of nine directions. A widget is eligible only if every ancestor up to the root is both
shown and enabled. A modal popup takes over navigation by installing itself as a **locker** —
which disables nothing, it merely makes everything outside its subtree ineligible.

**Notifications** are orthogonal to all of the above. A widget sends a
`(sender, message-id, payload)` triple to its *message target*, which defaults to its parent
but can be redirected to any window. The default handler rebroadcasts down to every enabled
child. The message vocabulary is one flat enumeration, and because the same numbers are
exported to Lua it is frozen. A screen binds handlers by pairing a widget (or a widget's
name) with a message id; the first matching binding wins.

**Double clicks are synthesised in the toolkit, not in the input layer**, with a per-frame
guard so that one physical press cannot become a double click in several windows at once. A
rebuild that synthesises them at the input seam gets that guard for free.

### Construction is data, and the element vocabulary is closed

Every screen is an XML document of nested elements carrying geometry, textures, fonts,
colours, and the names of handlers. §5 of the system requirements marks that format **frozen**.
The reader is one function per control type: it takes a document, an element path, an index,
and *a widget the caller already constructed*, and configures it.

That last point is the reason the vocabulary is frozen in practice and not only on paper:
**XML cannot introduce a widget type, only configure one the engine already knows.** The set
of element names is exactly the set of reader functions, and the set of reader functions is
this chapter's public control list. Readers compose by inheritance — a control's reader calls
its base's first — so every element understands the window attributes, and attribute names
are inherited the way behaviour is.

Two closed tables sit beside the readers: a named-colour table and a font-name table, both
loaded from data and both fatal on an unknown name. A layout may not name a font the manager
did not build.

Layered on top is the **style** mechanism: a style is a subdirectory of the configuration
tree, selected at runtime, and switching one re-points the path prefixes every document
lookup goes through. Style overriding is per entry, not per file — a style ships only what it
changes, and later definitions win.

Text is never a literal. An element's text content is an **identifier into the localization
string table**, resolved per language at load; §4 notes the shipped tables are in single-byte
codepages, not UTF-8.

### The drawing model

**Everything becomes one kind of primitive.** A picture, a button face, a frame corner, a
list row's background — all of it is a rectangle plus a sub-rectangle of an atlas page plus a
colour, clipped against the current scissor frustum and emitted as a triangle fan. Texture
coordinates are normalised at draw time from the page's *actual* reported resolution, which is
why one atlas description works against pages shipped at different sizes. A half-pixel offset
corrects texel centres, and is suppressed for UI drawn in world space (a computer screen
inside the level).

**Batching is per item, deliberately.** Each quad opens and flushes its own batch, because
reordering to batch across widgets would change which widget draws on top. The text layer
takes the opposite trade and batches every glyph in the frame into one flush.

**An icon is a logical name, not a file.** A process-wide registry maps a name onto (atlas
page, sub-rectangle), and caches one material per (page, pass), so a hundred widgets sharing
an atlas share one material.

**Stretchable frames come in two shapes.** A *frame line* is three segments — first, centre,
second — along one axis, with the centre either stretched or tiled; it dresses scroll bars,
edit boxes and list rows. A *frame window* is the nine-slice: four corners drawn at fixed
size, four edges tiled or stretched along their axis, and a centre fill. Between them they
are how every panel in the game scales to an arbitrary size from a small authored texture.

**Text layout has to satisfy three frozen constraints at once.** The shipped strings carry an
inline colour markup, they carry a two-character newline escape that must be recognised as an
escape and not as the letter that follows the backslash, and they are drawn in fonts that are
single-byte for the Latin and Cyrillic localizations and multi-byte for the Asian ones — and
those two font kinds wrap by entirely different rules. Text is decomposed into lines, lines
into coloured runs, and the result placed by horizontal and vertical alignment within the
box. Glyphs are queued and flushed once per frame.

**The font set is closed and fixed.** Eleven named fonts, each choosing its glyph atlas from
three resolution tiers at startup. Rebuilding a font on a device reset is done *in place*, so
every widget that captured a font reference keeps working.

### The settings-screen protocol is part of the toolkit

Several controls are simultaneously widgets and settings controls, implementing a four-step
protocol against one named console variable: read the current value, back it up, commit, undo.
A registry groups them by page name and drives open/accept/cancel across a whole page at once,
accumulating what kind of restart each committed change will cost and discharging those as
console commands after an accept. A rebuild that separates "widget" from "settings binding"
will find that separation fought by the XML vocabulary, which binds them in one element.

### The script surface is frozen

The class hierarchy, the method names, the message enumeration, the font names and the texture
registry are all exported to Lua, and every shipped game screen is a script calling them.
Acceptance criterion 10 is decided for this whole chapter by that one export file: a rebuild
may reorganise everything behind it and must reproduce its naming exactly.

## Twins at the top level

| Twin | Role |
|---|---|
| [`ui_base.cpp`](ui_base.cpp.md) · [`ui_base.h`](ui_base.h.md) | The module's single runtime object: canvas-to-screen scale, the scissor stack, the clipping frustums, and the four module-wide services |
| [`ui_defs.h`](ui_defs.h.md) | The virtual canvas constants and the 2D clipping frustum that trims widget geometry before submission |
| [`UIMessages.h`](UIMessages.h.md) | The flat notification vocabulary every widget sends — frozen, because the numbers are exported to Lua |
| [`uiabstract.h`](uiabstract.h.md) | The three shared vocabularies: the texture-owner interface, the alignment enumerations, and selectability |
| [`ui_focus.cpp`](ui_focus.cpp.md) · [`ui_focus.h`](ui_focus.h.md) | Directional navigation: eligibility, the nine-direction geometric search, and the modal locker |
| [`ui_styles.cpp`](ui_styles.cpp.md) · [`ui_styles.h`](ui_styles.h.md) | Discovering installed styles and switching by re-pointing the document path prefixes |
| [`ui_debug.cpp`](ui_debug.cpp.md) · [`ui_debug.h`](ui_debug.h.md) | The development inspector: the interface every widget implements, and the immediate-mode panel that hosts them |
| [`ui_export_script.cpp`](ui_export_script.cpp.md) | The frozen script surface for the whole toolkit |
| [`pch.cpp`](pch.cpp.md) · [`pch.hpp`](pch.hpp.md) | Build-time header aggregation; no decisions |

Not given twins: the build files (`CMakeLists.txt`, `packages.config`, the two project files).
They record one fact worth keeping — this module links against the core, the engine, the math
layer, the global-environment module and the script engine, and nothing else.

## Subdirectories

| Directory | Contents |
|---|---|
| [`Windows/`](Windows/README.md) | The tree node itself, and the two stretchable frames every panel is dressed in |
| [`Static/`](Static/README.md) | The picture-and-text widget nearly everything derives from, and the quad emitter under it |
| [`Lines/`](Lines/README.md) | The text engine: markup parsing, wrapping, alignment, coloured runs |
| [`FontManager/`](FontManager/README.md) | The closed set of named fonts and the per-frame glyph flush |
| [`XML/`](XML/README.md) | The layout reader, the closed element vocabulary, and the icon registry |
| [`Callbacks/`](Callbacks/README.md) | Binding (widget, message) pairs to handlers on a screen |
| [`Cursor/`](Cursor/README.md) | The single pointer position in canvas units, drawn last |
| [`Buttons/`](Buttons/README.md) | The press state machine, the four-state button, the check box, the radio button, the hover hint |
| [`EditBox/`](EditBox/README.md) | Text entry: the line editor, the caret window, and the two framed variants |
| [`ComboBox/`](ComboBox/README.md) | The drop-down bound to a console variable's token set |
| [`SpinBox/`](SpinBox/README.md) | Steppers: integer, real and token-list |
| [`TrackBar/`](TrackBar/README.md) | The slider |
| [`ProgressBar/`](ProgressBar/README.md) | Linear, comparison and circular progress indicators |
| [`arrow/`](arrow/README.md) | The needle widget that chases a value at a bounded angular speed |
| [`ScrollBar/`](ScrollBar/README.md) | The scroll bar, its fixed-thumb variant, and the draggable box |
| [`ScrollView/`](ScrollView/README.md) | The variable-height-item scrolling container |
| [`ListBox/`](ListBox/README.md) | The scrolling list of uniform selectable rows |
| [`ListWnd/`](ListWnd/README.md) | The row-paged list: whole rows only, focused and selected tracked separately |
| [`TabControl/`](TabControl/README.md) | Mutually exclusive tab buttons, and the settings binding on top |
| [`MessageBox/`](MessageBox/README.md) | The modal dialog whose child set is chosen by a style named in data |
| [`PropertiesBox/`](PropertiesBox/README.md) | The pop-up context menu |
| [`Hint/`](Hint/README.md) | The per-screen tooltip and the dwell rule |
| [`InteractiveBackground/`](InteractiveBackground/README.md) | Four alternative backgrounds of which exactly one is drawn |
| [`Options/`](Options/README.md) | The settings-control protocol and the registry that drives a whole page |
