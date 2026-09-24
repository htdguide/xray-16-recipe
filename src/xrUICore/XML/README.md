# src/xrUICore/XML — layout documents and the icon registry

> The whole interface is data. This directory is the reader for that data, the closed
> vocabulary it may contain, and the registry that turns a logical icon name into a rectangle
> on an atlas page.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Three things.

The **layout reader**: one function per control type, each taking a document, an element path,
an index within that path, and an already-constructed widget, and configuring the widget from
the element. Alongside it the two closed tables every document draws on — named colours, and
font names.

The **document flavour**: the engine's general XML reader, specialised with the one rule
specific to layouts — a document opened from a UI path silently resolves to its widescreen
variant when one exists. That substitution is how the shipped screens adapt to a wide display
without any widget knowing the aspect ratio.

The **icon registry**: a process-wide map from a logical name to (atlas page, sub-rectangle),
built by scanning texture-description documents, plus a cache of one material per (page, pass)
so that a hundred widgets sharing an atlas share one material.

## The load-bearing ideas

**The vocabulary is closed because the reader does not construct.** Every entry point receives
a widget the caller already made, so XML cannot introduce a widget type — it can only
configure one the engine already knows. The set of element names is exactly the set of reader
functions, which is exactly this chapter's public control list. §5 of the system requirements
marks the format frozen; this is what makes that true in practice and not only on paper.

**Readers compose by inheritance.** A control's reader calls its base's reader first, so
attribute names are inherited the way behaviour is: every element understands the window
attributes, a button understands everything a static does. A rebuild gets the same effect from
any composition mechanism, but must keep the attribute set inherited rather than restated.

**Failure is per element and chosen by the caller.** Each reader takes a fatal flag: a missing
element aborts loading with the document name and path when it is set, and returns failure
leaving the widget untouched when it is not. Optional sub-elements are how one layout stays
compatible across three games' data.

**Text content is a key, not a literal.** An element's text is an identifier into the
localization string table, resolved per language at load. The shipped tables are in
single-byte codepages (§4), not UTF-8.

**The registry stores texels, not normalised coordinates**, because the atlas page may not be
resident when the description is read. Normalisation happens at draw time against the page's
actual resolution.

**Style overrides are per entry.** Texture descriptions are read from the default style first
and then from the active style, with later definitions winning, so a style ships only the
entries it changes.

## The twins

| Twin | Role |
|---|---|
| [`UIXmlInitBase.cpp`](UIXmlInitBase.cpp.md) | The entire XML-to-widget vocabulary: one reader per control type, plus the named-colour and font-name tables |
| [`UIXmlInitBase.h`](UIXmlInitBase.h.md) | Which is to say, the enumeration of the closed set of element types a layout may contain |
| [`UITextureMaster.cpp`](UITextureMaster.cpp.md) | The icon registry: name to (page, sub-rectangle), and one shared material per (page, pass) |
| [`UITextureMaster.h`](UITextureMaster.h.md) | Its declaration and the two records it is built from |
| [`xrUIXmlParser.cpp`](xrUIXmlParser.cpp.md) | The one layout-specific document rule: silent substitution of a widescreen variant |
| [`xrUIXmlParser.h`](xrUIXmlParser.h.md) | The layout-flavoured document's declaration |
