# src/xrUICore/XML/UIXmlInitBase.h

> Declares the XML-to-widget reader implemented in [`UIXmlInitBase.cpp`](UIXmlInitBase.cpp.md) — which is to say, it enumerates the closed set of element types a layout document may contain.

**Needs** — [`UIXmlInitBase.cpp`](UIXmlInitBase.cpp.md) · [`xrUIXmlParser.h`](xrUIXmlParser.h.md)
**Used by** — [`UIHelper.cpp`](../../xrGame/ui/UIHelper.cpp.md) · [`UIXmlInit.h`](../../xrGame/ui/UIXmlInit.h.md) · [`UIBtnHint.cpp`](../Buttons/UIBtnHint.cpp.md) · [`UIHint.cpp`](../Hint/UIHint.cpp.md) · [`UILines.cpp`](../Lines/UILines.cpp.md) · [`UIMessageBox.cpp`](../MessageBox/UIMessageBox.cpp.md) · [`UIDoubleProgressBar.cpp`](../ProgressBar/UIDoubleProgressBar.cpp.md) · [`UIPropertiesBox.cpp`](../PropertiesBox/UIPropertiesBox.cpp.md) · [`UIFixedScrollBar.cpp`](../ScrollBar/UIFixedScrollBar.cpp.md) · [`UIScrollBar.cpp`](../ScrollBar/UIScrollBar.cpp.md) · [`UIXmlInitBase.cpp`](UIXmlInitBase.cpp.md) · [`ui_arrow.cpp`](../arrow/ui_arrow.cpp.md) · [`ui_base.cpp`](../ui_base.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`UIXmlInitBase.cpp`](UIXmlInitBase.cpp.md). Read as a
list, it *is* the frozen element vocabulary: a layout document can contain exactly the
constructs this header names and nothing else, because nothing else can be read.

Two decisions are visible only here.

**Every reader is a free operation, not a widget method.** The widget knows nothing about
XML and the reader knows nothing about drawing. That separation is why the game layer can
add its own readers for its own widgets without touching the toolkit, and why a rebuild can
replace the document format without touching a widget.

**The named-colour table is process-wide and owned here**, not by the document or the screen.
It is reloaded on every UI reset and asserted to exist whenever a colour name is resolved,
which makes "the palette is loaded before any screen is" an ordering requirement on startup.

The type is a class of static operations with a constructor whose only effect is to load the
colour table — a way of saying "someone must have constructed one of these before reading a
document". A rebuild makes that an explicit initialisation step.

## Exported units

Readers, each taking (document, element path, index, widget, fatal flag):

- `InitWindow` — position, size, alignment, name, auto-static children; the base of all others
- `InitStatic` — text, texture, mirroring, rotation, two light animations, hint
- `InitText` — font, colour, alignment, offset, localized content
- `InitFont` — resolve a font name and colour against the closed font table
- `InitTexture` / `InitTextureOffset` / `InitMultiTexture` — the texture-owner readers
- `InitFrameWindow` / `InitFrameLine` — nine-slice and three-part backgrounds
- `InitButton` / `Init3tButton` / `InitCheck` / `InitSound` — buttons and their accelerators
- `InitSpin` / `InitCustomEdit` / `InitEditBox` — value and text entry
- `InitProgressBar` / `InitProgressShape` — linear and radial progress
- `InitTabControl` — tabs, each with a mandatory identity
- `InitScrollView` / `InitListWnd` / `InitListBox` / `InitComboBox` — containers
- `InitTrackBar` — slider, integer or real, with an optional value readout
- `InitAnimatedStatic` — sprite-sheet animation
- `InitOptionsItem` — bind a widget to a settings entry, group and restart dependency
- `InitAlignment` — the window alignment mode (and the inert coordinate-adjust attribute)
- `InitAutoStaticGroup` / `InitAutoFrameLineGroup` — construct decoration children from XML
- `GetFRect` — a bare rectangle from any element

Colour table:

- `GetColor(document, path, index, fallback)` — named reference or loose components
- `GetColor(name, out)` — look up a named colour, reporting absence
- `InitColorDefs` / `DeleteColorDefs` / `AssignColor` / `GetColorDefs`

Inert:

- `ApplyAlignX` / `ApplyAlignY` / `ApplyAlign` — declared and wired, return their input
  unchanged; see the note in [`UIXmlInitBase.cpp`](UIXmlInitBase.cpp.md)
