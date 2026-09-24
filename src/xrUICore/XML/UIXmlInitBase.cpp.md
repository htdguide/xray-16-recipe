# src/xrUICore/XML/UIXmlInitBase.cpp

> The entire XML-to-widget vocabulary — one function per control type that reads a node and configures an already-constructed widget from it, plus the named-colour table and the font name table that every layout file draws on.

**Needs** — [`UIXmlInitBase.h`](UIXmlInitBase.h.md) · [`xrUIXmlParser.h`](xrUIXmlParser.h.md) · [`UITextureMaster.h`](UITextureMaster.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Static/UIAnimatedStatic.h`](../Static/UIAnimatedStatic.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`Buttons/UICheckButton.h`](../Buttons/UICheckButton.h.md) · [`Buttons/UIRadioButton.h`](../Buttons/UIRadioButton.h.md) · [`SpinBox/UICustomSpin.h`](../SpinBox/UICustomSpin.h.md) · [`ProgressBar/UIProgressBar.h`](../ProgressBar/UIProgressBar.h.md) · [`ProgressBar/UIProgressShape.h`](../ProgressBar/UIProgressShape.h.md) · [`TabControl/UITabControl.h`](../TabControl/UITabControl.h.md) · [`ScrollView/UIScrollView.h`](../ScrollView/UIScrollView.h.md) · [`ListWnd/UIListWnd.h`](../ListWnd/UIListWnd.h.md) · [`ListBox/UIListBox.h`](../ListBox/UIListBox.h.md) · [`ComboBox/UIComboBox.h`](../ComboBox/UIComboBox.h.md) · [`TrackBar/UITrackBar.h`](../TrackBar/UITrackBar.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`xrEngine/StringTable/StringTable.h`](../../xrEngine/StringTable/StringTable.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md)
**Used by** — [`UIXmlInitBase.h`](UIXmlInitBase.h.md)
**Tier floor** — T3: attribute reads and property assignment. Runs at screen-load time, not
per frame.

## Purpose

The shipped interface is data. Every screen is an XML document of nested elements with
positions, textures, fonts, colours and handler names, and §5 of the system requirements
marks that format **frozen**. This file is the reader: for each control type, a function
that takes a document, an element path, an index within that path, and an
already-constructed widget, and configures the widget from the element.

Two structural decisions run through the whole file and a rebuild must copy both.

**The vocabulary is closed and the widget is not constructed here.** Every entry point
receives a widget the caller already made. XML therefore cannot introduce a widget type —
it can only configure one the engine already knows. That is why the element vocabulary is
frozen in practice as well as on paper.

**Initialisation composes by inheritance.** A control's reader calls its base's reader
first, so `<tab_control>` gets everything `<window>` has, a button gets everything a static
has, and an edit box gets everything a custom edit has. Attribute names are therefore
inherited too: every element understands `x`, `y`, `width`, `height`, `alignment`.

The unit of failure is uniform: each reader takes a *fatal* flag. When the element is missing
and fatal is set, loading aborts with the document name and the path; when it is not, the
reader returns failure and leaves the widget untouched. Optional sub-elements are how a
layout stays compatible across the three games.

## State

```text
RECORD ColorDefs                       # process-wide, reloaded on every UI reset
  by_name : map<text, colour>          # from "color_defs.xml"
```

Plus a fixed table of font names, mapping the identifiers a layout may write
(`graffiti19`, `graffiti22`, `graffiti32`, `graffiti50`, `arial_14`, `medium`, `small`,
`letterica16`, `letterica18`, `letterica25`, `di`) onto the font manager's instances.

**Invariants**

- The colour table must exist before any element is read; every reader that resolves a named
  colour asserts the name is present rather than defaulting.
- A font name not in the table aborts. The table is the closed set of fonts a layout may
  reference; adding one means adding a font to the manager first.

## `InitWindow`

**Contract** — the base of every other reader. Reads position and size, applies the
alignment attributes, optionally sets the window's name from a `window_name` child, then
scans the element's children for the auto-static group. Returns false (or aborts) when the
element is absent.

```text
FUNCTION init_window(doc, path, index, w, fatal) -> bool
  IF element at (path, index) is absent
    IF fatal THEN abort naming path and document
    RETURN false
  position = (attr "x", attr "y")
  apply_alignment(doc, path, index, position, w)     # may set w's alignment mode
  size = (attr "width", attr "height")
  w.position = position; w.size = size
  IF child "window_name" exists THEN w.name = its text
  init_auto_static_group(doc, path, index, w)
  RETURN true
```

## `InitAutoStaticGroup`

**Contract** — the one place XML *creates* widgets. Scans the element's immediate children
for elements named `auto_static` and `auto_frameline`, constructs a static (or frame line)
for each, configures it from that element, names it `auto_static_<n>` / `auto_frameline_<n>`
by ordinal, marks it auto-delete, and attaches it to the parent window.

```text
FUNCTION init_auto_static_group(doc, path, index, parent)
  save the document's current local root; descend into (path, index)
  static_n = 0; frameline_n = 0
  FOR EACH immediate child element
    IF its name is "auto_static"
      s = new static; init_static(doc, "auto_static", static_n, s)
      s.name = "auto_static_" + static_n          # re-applied AFTER init on purpose
      s.auto_delete = true; parent attach s
      static_n = static_n + 1
    ELSE IF its name is "auto_frameline"
      ... the same with a frame line ...
  restore the saved local root
```

**Invariants** — the ordinal name is assigned *after* the element's own reader has run,
because that reader may itself have set a name from a `window_name` child and some game code
looks the widget up by its ordinal name. The reassignment is deliberate, not redundant.

**Notes** — `auto_text` children are counted and otherwise ignored; the counter is dead. The
separate `InitAutoFrameLineGroup` entry point does the frame-line half alone and is not
called from the window reader — the combined scan superseded it. A rebuild keeps one.

## `InitStatic`

**Contract** — the workhorse: the base of buttons, progress bars, check buttons, edit boxes
and most game widgets. On top of a window it reads a `text` child (through `InitText`), a
texture (through `InitTexture`), a texture offset, a mirroring mode, a rotation, two light
animations, a complex-text flag and a hint string.

```text
FUNCTION init_static(doc, path, index, w, fatal, text_only) -> bool
  IF NOT init_window(...) THEN RETURN false
  init_text(doc, path + ":text", index, w.text_control)
  init_texture(doc, path, index, w)
  init_texture_offset(doc, path, index, w)

  mirror  = attr "mirror"            # "h" | "v" | "b" -> horizontal / vertical / both
  heading = attr "heading"           # non-zero: this widget may be rotated
  angle   = attr "heading_angle" in degrees
  IF angle is non-zero
    enable heading, mark it constant, set it to `angle` in radians

  # colour animation, addressed by name into the shared light-animation library
  animation      = attr "light_anim"
  cyclic         = attr "la_cyclic"   (default 1)
  affects_text   = attr "la_text"     (default 1)
  affects_texture= attr "la_texture"  (default 1)
  alpha_only     = attr "la_alpha"    (default 0)
  set colour animation with those flags
  #   `text_only` callers force the text-colour flag on regardless

  # transform animation: the same library driving rotation and scale instead of colour
  set transform animation from attr "xform_anim", cyclic from "xform_anim_cyclic"

  IF attr "complex_mode" THEN enable multi-line/markup text layout
  w.hint = attr "hint"

  IF text_only                          # this element is declared as a pure text widget
    assert it declares no texture child
    assert it has no children
  RETURN true
```

**Invariants** — the `text_only` mode exists so that a layout can declare a text widget and
be *told* at load time if it accidentally gave it a texture or children, rather than getting
silently wrong layout. The assertion message points the author at the right alternative.

**Notes** — the two light animations are independent and come from the same authored
library: one drives colour, the other drives rotation and scale. Both are named, not
inlined — see [`UIStatic.cpp`](../Static/UIStatic.cpp.md) for how the animation's colour
channels are reinterpreted as an angle and a scale.

## `InitText`

**Contract** — configures a text block from a `text` element: font and colour, horizontal and
vertical alignment, complex mode, an offset, and the content itself — **translated through
the localization table**. Returns false if the element is absent.

```text
FUNCTION init_text(doc, path, index, lines) -> bool
  IF element absent THEN RETURN false
  (colour, font) = init_font(doc, path, index)
  lines.colour = colour
  IF font EXISTS THEN lines.font = font ELSE warn: this node has no font
  lines.align  = attr "align"       # "l" | "c" | "r"
  lines.valign = attr "vert_align"  # "t" | "c" | "b"
  lines.complex_mode = attr "complex_mode"
  # the offset may already have been set by the parent element's reader (a button's
  # text is offset by the button); use the existing value as the default
  lines.offset = (attr "x" default lines.offset.x, attr "y" default lines.offset.y)
  IF the element has text content
    lines.text = localize(content)
  RETURN true
```

**Invariants** — the element's text is an *identifier into the string table*, not a literal:
the shipped documents contain keys, and the string table resolves them per language (§4,
locale and encoding). A missing key resolves to itself, which is why a missing translation
shows as a key on screen rather than blank.

**Notes** — using the widget's existing offset as the attribute default is what lets a
composite control (a three-state button) position its own text and then let the layout
override just one axis.

## `InitFont`

**Contract** — resolves the `font` attribute against the closed name table and reads the
colour from the same element. Returns whether a font was named at all; an unnamed font is not
an error (the caller warns), an *unknown* font name aborts.

## `GetColor` (two forms) and the colour table

**Contract** — a colour may be given either as a named reference (`color="ui_gray_1"`) or as
loose `r`/`g`/`b`/`a` components, each defaulting from a caller-supplied fallback. The named
form is resolved against the table and asserts the name exists.

`InitColorDefs` loads that table from `color_defs.xml`, found through the style-aware path
lookup, as a list of `name, r, g, b, a` entries with alpha defaulting to opaque.
`AssignColor` adds or replaces one entry programmatically, which is how the game layer
registers colours the documents can then use.

**Notes** — named colours are the layout data's only abstraction mechanism. They are
reloaded on every UI reset, which is what makes a style able to restate the palette.

## `InitAlignment`

**Contract** — reads two different attributes with confusingly similar names, doing two
different things.

`alignment` sets the *window's* alignment mode — how its position is to be interpreted — from
one of `l`, `r`, `t`, `b`, `c`. This is the one that matters; see
[`UIWindow.cpp`](../Windows/UIWindow.cpp.md).

`align` is meant to adjust the *coordinates themselves* for right/bottom/centre anchoring.
It reaches `ApplyAlignX` / `ApplyAlignY`, **which return the coordinate unchanged**. The
attribute is therefore inert: it is parsed, dispatched, and has no effect.

**Notes** — this is an unrecovered decision. The coordinate-adjusting path is fully wired
and deliberately neutered; the alignment-mode path supersedes it. A rebuild should implement
only `alignment` and treat `align` as a document attribute it accepts and ignores — which is
what the original does, whether or not that was the intent.

## `InitTexture`

**Contract** — the shared texture reader used by every textured widget through the
texture-owner interface. Reads a `texture` child: its content is the registry key, an
optional `shader` attribute names a material pass, and optional `x`/`y`/`width`/`height`
attributes override the sub-rectangle. Also reads a `stretch` flag and a modulation colour.
A zero-sized rectangle is not applied, leaving whatever the registry supplied.

**Notes** — the element is looked up by path only (not by index) for its *presence*, then by
index for its content. That inconsistency means a repeated element's presence is decided by
the first occurrence. It is survivable because texture children are not repeated in practice.

## `InitMultiTexture`

**Contract** — the three-state button's texture reader. A single `texture` child means one
texture for every state. Otherwise it looks for four suffixed children — `texture_e`,
`texture_t`, `texture_d`, `texture_h` — and assigns them to the enabled, touched, disabled
and highlighted states, routing each to whichever backing the button has (a plain image or a
stretchable frame line, in which case the orientation is also set from the button's own
vertical flag). Turns the texture on only if at least one state was supplied.

## `InitFrameLine`

**Contract** — reads a stretchable three-part line: orientation from a `vertical` flag, then
window and texture, then **re-reads the stretch flag with a game-dependent default** — off
for the two older games, on for the newest — because the shipped art differs in whether the
centre segment is meant to tile or stretch. Optionally reads a `title` child as a static.

**Invariants** — the stretch re-read must happen after the texture reader, which already set
it from the generic default. This is the one attribute in the whole vocabulary whose default
depends on which game's data is mounted.

## `InitFrameWindow`

**Contract** — window plus nine-slice texture, plus an optional `title` child read as a
static. The title exists for the oldest game's dialogs, which draw a caption inside the
frame.

## `InitButton` / `Init3tButton`

**Contract** — a plain button is a static plus accelerators plus a hint. A three-state
button additionally reads a frame-mode flag and a vertical flag *before* the window (because
they change how the button builds itself), then its four per-state text colours, its
multi-texture set, its texture offset and its two sounds.

Accelerators are read from `accel` and `accel_ext` and resolved in two steps: first as a
*key name* against the scancode table, then — if that fails — as a *bound action name*
against the key-binding table. A name that is neither logs a warning naming the document,
the path and the index, and is ignored.

**Invariants** — resolving to an action rather than a key is what lets a button say "I am the
Use key" and follow the player's rebinding. Storing the scancode directly is what lets a
button claim a fixed key regardless of bindings. The two are distinguished by a flag stored
alongside; see [`UIButton`](../Buttons/UIButton.cpp.md).

**Notes** — the hint text is localized at load; the accelerator names are not, because they
index binding tables.

## `InitCheck` / `InitSpin` / `InitCustomEdit` / `InitEditBox`

**Contract** — each layers a few attributes on a base reader and then calls the widget's own
initialiser with the position and size the base reader already established.

- *Check button*: a static, then a texture name defaulting to the standard checkbox art, then
  four optional per-state text colours, then the options-item binding.
- *Spin box*: a window, the options-item binding, the widget's own build, then enabled and
  disabled text colours.
- *Custom edit*: a static, the widget's build, an enabled text colour, then the four
  behaviour flags — maximum length, digits only, read only, filename mode. **If any of them
  is set and no maximum length was given, the length defaults to 32.** A password flag
  switches the display to masked.
- *Edit box*: a custom edit plus a texture and the options-item binding.

**Notes** — the 32-character default is arbitrary and undocumented; it only applies when the
author asked for constrained input without saying how long.

## `InitOptionsItem`

**Contract** — binds a widget to a settings entry. Reads an `options_item` child: the `entry`
names the console variable or settings key, the `group` names the settings page it belongs
to, and an optional `depend` says what a change costs — nothing, a video restart, a sound
restart, a UI reload, a full restart, or applies immediately. An unrecognised dependency
logs and is treated as nothing.

**Invariants** — the group is what lets the settings screen save, restore or undo a whole
page at once without knowing which widgets are on it. The dependency is what produces the
"restart required" notice.

## `InitProgressBar` / `InitProgressShape`

**Contract** — the bar reads position, size and a fill direction; the direction may be given
either as a legacy `horz` flag or as a `mode` string with six values: horizontal, vertical,
backwards, downwards, from the centre horizontally, from the centre vertically. Then range,
initial position, an inertia factor, a mandatory `progress` child and an optional
`background` child (each read as a static and resized to the bar), and an optional
three-colour gradient — minimum, optional middle, maximum — which when present makes the bar
tint itself by fill fraction.

The shape variant reads a static, an optional text flag, optional `back` and `front` children
(each constructed as a child static), a sector count defaulting to eight, a clockwise flag, a
blend flag, and a start and end angle defaulting to a full turn. It fails if neither a front
child nor a texture on the shape itself was supplied, since there would be nothing to draw.

**Notes** — the legacy flag and the mode string coexist because the older games' documents use
the flag. Both must be honoured, flag first.

## `InitTabControl`

**Contract** — reads a window, an options-item binding, an accelerator mode flag, and a
`radio` flag that decides whether the tabs are constructed as tab buttons or as radio
buttons. Then descends into the element and reads each `button` child as a three-state
button, taking its `id` attribute as the tab's identity.

**Invariants** — a tab **must** have an identity, because the active tab is stored and
restored by identity, not index. Missing identities abort unless the caller explicitly allows
defaults, in which case the ordinal is substituted, the substitution is logged, and the tab
is flagged as having a generated identity so later code can tell.

## `InitScrollView` / `InitListWnd` / `InitListBox` / `InitComboBox`

**Contract** — the four container readers.

*Scroll view*: four indents, a vertical interval between rows, an inverse-direction flag, a
scroll-bar profile name, then the widget's own build, then a vertical-flip flag, an
always-show-scrollbar flag (default on) and a selectable-items flag. Finally, any `text`
children are constructed as statics sized to the view's usable width, height-fitted to their
text, and added as rows — which is how a purely textual scrolling panel is authored with no
code at all.

*List window*: position, size, row height, an active-background flag, a font, a scroll-bar
profile, then the widget's build, then the two scroll-visibility flags (always show, always
hide) and a vertical-flip flag.

*List box*: a scroll view plus a font and a row height defaulting to 20 canvas units.

*Combo box*: a visible list length defaulting to four, a window, the widget's build, the
options-item binding, an always-show-scrollbar flag, the list font and colour, and the
enabled and disabled text colours.

**Notes** — the scroll-bar *profile* is a name into a separate document describing the scroll
bar's art and metrics; see [`UIScrollBar.cpp`](../ScrollBar/UIScrollBar.cpp.md). Defaulting it
to `"default"` everywhere is what lets a layout omit it.

## `InitTrackBar`

**Contract** — a window, the widget's build, an integer-or-real type flag, the options-item
binding, an invert flag, a step defaulting to a tenth, and a minimum and maximum in the
matching type. **Bounds are applied only if they differ**; equal bounds mean "take the range
from the settings entry instead", and the widget is told whether its bounds were set here.
Optionally an `output_wnd` child becomes a static showing the value, with a `format` attribute
giving the numeric formatting.

**Invariants** — the reader passes index 0 rather than the caller's index for the window and
the options item while using the caller's index for every attribute. That inconsistency means
a repeated track-bar element takes its geometry from the first occurrence. It is a defect, not
a decision, and a rebuild should use the index throughout.

## `InitAnimatedStatic`

**Contract** — a static plus a sprite-sheet animation: a source offset into the page, a frame
count, a total duration in milliseconds, a column count, a frame size, a cyclic flag and an
autoplay flag. Sets the animation to its first frame and starts it if autoplay was asked for.

## `GetFRect`

**Contract** — reads a rectangle from `x`, `y`, `width`, `height` on any element, aborting if
the element is absent. Used where a raw rectangle rather than a widget is wanted.

## `InitTextureOffset`

**Contract** — reads a `texture_offset` child's `x` and `y` and applies them to a static.
When the caller's path is empty, the child is addressed at the document root instead — which
is how a texture offset is read from a document whose root *is* the element.

## `InitSound`

**Contract** — reads a three-state button's two sound names, `sound_h` (highlight) and
`sound_t` (touch), and installs whichever is non-empty. Both are optional.
