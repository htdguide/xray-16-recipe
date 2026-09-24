# src/xrUICore/FontManager — the closed font set

> Eleven named fonts, each picking its glyph atlas from three resolution tiers, all flushed
> once per frame.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Owning the fixed set of fonts the whole interface draws with, building each from a named
configuration section, and performing the one per-frame flush that turns every widget's queued
glyphs into draw calls.

The set is **closed**: a layout document may name a font only from a fixed table, and an
unknown name aborts loading. Adding a font to the game means adding it to this manager first.

## The load-bearing ideas

**Atlas selection is by screen height, in three tiers.** Each font names three glyph atlases
and the manager picks one from the current resolution, so a high resolution gets crisper
glyphs from a bigger page without the layout changing. The choice happens at build time and is
redone on a device reset.

**Two fonts are device-independent.** The heads-up display font and the statistics font
express their size as a fraction of the screen rather than in canvas units, and take a
different size-setting path because of it. Everything else is in canvas units like the rest of
the chapter.

**Rebuilding is in place.** On a reset an existing font is re-initialized rather than replaced,
so every widget that captured a reference to a font keeps working. That is why the set is held
as slots rather than as values, and it is the reason a rebuild cannot simply hand out font
values.

**One flush per frame, for all of them.** Glyphs queued by every text control across the whole
frame are emitted in one pass at the end. This is the opposite of the quad emitter's per-item
batch, and it is why text is cheap and pictures are not.

## The twins

| Twin | Role |
|---|---|
| [`FontManager.cpp`](FontManager.cpp.md) | Building the eleven fonts from configuration, the three-tier atlas choice, and the per-frame flush |
| [`FontManager.h`](FontManager.h.md) | The named set and its fixed order, which is also the teardown order |
