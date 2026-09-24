# src/xrGame/ui/UIDebugFonts.cpp

> A development screen that renders one line of sample text in every font the font manager built, stacked down the canvas, so a localization change can be eyeballed against the whole closed font set at once.

**Needs** — [`UIDebugFonts.h`](UIDebugFonts.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/FontManager/FontManager.h`](../../xrUICore/FontManager/FontManager.h.md) · [`xrEngine/StringTable/StringTable.h`](../../xrEngine/StringTable/StringTable.h.md)
**Used by** — [`UIDebugFonts.h`](UIDebugFonts.h.md)
**Tier floor** — T3.

## Purpose

Chapter 15 states that the font set is closed and fixed, and that each font picks its glyph
atlas from one of three resolution tiers at startup. This screen is the check on that claim:
it asks the font manager for its whole list and draws the same localized sample string once
per font. Nothing in the game uses it; it is an inspection tool with a screen's shape because
that is the cheapest way to see glyphs at their real size.

Unlike every other screen in this chapter it is **not built from an XML layout** — it builds
its children in code. That is deliberate: a layout document can only name fonts the manager
already has, so a layout-driven font list could never show a font the layout author forgot.

## State

```text
RECORD DebugFontsScreen EXTENDS DialogScreen
  background : Picture          # covers the whole canvas
  rows       : list<Label>      # one per font, owned by the tree
```

## Construction

**Contract** — sizes itself to the full virtual canvas, fills the row list, then dresses the
background with a fixed debug texture. Allocates one label per font; each label is owned by
the tree and freed with it.

```text
FUNCTION fill_rows()
  y := 0
  FOR EACH font IN font_manager.all_fonts
    label := new Label
    label.position := (0, y)
    label.size     := (canvas_width, canvas_height)   # deliberately over-tall; see notes
    label.text     := font.name + ":" + localize("Test_Font_String")
    label.font     := font
    label.complex_text := false                        # no inline markup in the sample
    label.align    := (horizontal centre, vertical centre)
    label.shrink_height_to_text()
    y := y + label.height + 20
    attach(label)
```

**Notes** — each row is created at full canvas height and then told to shrink to its text.
The over-tall start is not waste: vertical centring is computed against the box, so the
shrink leaves the glyphs exactly centred on their own line regardless of the font's ascent,
which differs sharply between the single-byte and multi-byte font kinds. The 20-unit gap is
the visual separation between rows and carries no other meaning.

Inline colour markup is switched off for the sample so that a string table entry containing
markup shows its markup rather than obeying it — the point is to see the glyphs.

The sample string is looked up in the localization table under a fixed identifier, so
switching language switches the sample. In release builds the row text is never composed at
all and the labels come out blank; the screen is a development tool and is left that way
rather than being compiled out entirely.

## Input

**Contract** — the quit binding closes the screen; the screenshot binding is refused so it
falls through to the engine and actually takes a screenshot of the font sheet, which is the
usual reason to open this screen. Every other key is swallowed.

**Notes** — the two keys are looked up through the *action binding* table, not by scancode,
so the screen obeys the player's remapping. Returning "not handled" for the screenshot key is
the mechanism by which a screen lets one specific engine binding through while consuming all
others; it is used the same way elsewhere in the chapter.
