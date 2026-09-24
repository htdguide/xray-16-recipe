# src/xrUICore/FontManager/FontManager.cpp

> Builds eleven configured fonts, choosing each one's glyph atlas from three resolution tiers, and flushes all of their queued glyphs once per frame.

**Needs** — [`FontManager.h`](FontManager.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`FontManager.h`](FontManager.h.md)
**Tier floor** — T2.

## Purpose

Two decisions: which atlas a font uses at a given screen height, and when glyphs actually
reach the device.

## State

```text
RECORD FontManager
  fonts : list<Font>     # eleven, in a fixed order that is also the teardown order
```

**Invariants**

- Every entry is null before initialization and non-null after; `Render` dereferences all of
  them unconditionally, so a font that failed to build would crash rather than be skipped.
- Rebuilding is in-place: an existing font is re-initialized rather than replaced, so every
  widget that captured a font reference keeps working across a UI reset. This is the reason
  the set is held as references-to-slots rather than as values.

## `initialize_fonts`

**Contract** — builds all eleven from named configuration sections. Two carry flags: the
heads-up display font is built with a gradient and in device-independent mode, and the
statistics font in device-independent mode. Device-independent means its size is expressed
as a fraction of the screen rather than in virtual UI units, which is why those two take a
different size-setting path below.

## `initialize_font`

**Contract** — reads the atlas name, the shader name, and the optional size and inter-glyph
interval from one configuration section, and either constructs the font or re-initializes an
existing one. The size is applied through the device-independent setter when that flag is
present and the ordinary one otherwise. A missing atlas is fatal.

```text
FUNCTION initialize_font(slot, section, flags)
  atlas  <- pick_atlas(section)
  shader <- config(section, "shader")           # required
  IF slot IS none THEN slot <- new Font(shader, atlas, flags)
                  ELSE slot.reinitialize(shader, atlas)
  IF config_has(section, "size")
    IF flags HAS device_independent THEN slot.set_height_fraction(config_float(section, "size"))
                                    ELSE slot.set_height(config_float(section, "size"))
  IF config_has(section, "interval")
    slot.set_interval(config_vector2(section, "interval"))
```

## `get_font_tex_name`

**Contract** — picks one of three atlas keys from the current back-buffer height, then walks
*down* through the tiers until a key that the section actually defines is found, and falls
back to the middle tier's key if none is.

```text
FUNCTION pick_atlas(section) -> text
  tiers <- [ "texture800", "texture", "texture1600" ]     # keys, low to high
  idx <- IF height <= 600  THEN 0
         ELSE IF height <= 1024 THEN 1
         ELSE 2
  WHILE idx >= 0
    IF config_has(section, tiers[idx]) RETURN config(section, tiers[idx])
    idx <- idx - 1
  RETURN config(section, tiers[1])      # the 1024×768 tier is the last resort
```

**Notes** — the thresholds are 600 and 1024 *pixels of height*, and the unsuffixed key is the
one authored for the toolkit's own 1024×768 virtual space. Walking downward rather than
upward means a section that defines only the base atlas is served at every resolution, which
is what most shipped sections do; only a few define the high-resolution variant. The final
fallback re-reads the base key and is fatal if even that is absent.

## `render`

**Contract** — flushes every font's accumulated glyph queue. Text drawing anywhere in the
toolkit only *queues*; nothing reaches the device until this runs, once per frame, at a fixed
point in the UI rendering order.

**Notes** — this is why text ordering across widgets is not the window tree's ordering: all
text of one font is drawn together at flush time, in queue order, regardless of which widget
queued it. Any rebuild that draws text immediately will produce a visibly different layering
and must re-check every screen where text overlaps a later-drawn quad.

## `on_ui_reset`

**Contract** — re-runs the whole initialization in place. Invoked when the language, the
style or the resolution class changes, and is the reason the atlas choice above is re-made
rather than cached.
