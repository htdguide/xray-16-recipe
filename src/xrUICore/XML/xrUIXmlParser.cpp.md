# src/xrUICore/XML/xrUIXmlParser.cpp

> Specialises the engine's XML document with the one rule that is specific to layouts — a document opened from a UI path silently resolves to its widescreen variant when one exists.

**Needs** — [`xrUIXmlParser.h`](xrUIXmlParser.h.md) · [`ui_base.h`](../ui_base.h.md) · [`xrCore/XML/XMLDocument.hpp`](../../xrCore/XML/XMLDocument.hpp.md)
**Used by** — [`xrUIXmlParser.h`](xrUIXmlParser.h.md)
**Tier floor** — T3: a string decision made before a file is opened.

## Purpose

The engine's document loader lets a subclass rewrite a filename before opening it. This file
is the whole of the layout module's use of that hook, and it implements one rule: a document
requested from a UI path — either the active style's or the default's — is resolved through
the widescreen name substitution described in [`ui_base.cpp`](../ui_base.cpp.md), so that
`main_menu.xml` becomes `main_menu_16.xml` on a wide display when that file exists.

Documents requested from any other path are passed through untouched. That is the whole
point of testing the path: only *layout* documents have widescreen variants, and applying
the substitution to, say, a configuration document would look for a file that will never
exist.

## State

`Stateless.` The type carries one debug identifier field that nothing reads.

## `correct_file_name`

**Contract** — given the logical path a document is being opened from and the filename
requested, return the filename to actually open. Pure; opens nothing.

```text
FUNCTION correct_file_name(path, name) -> text
  IF path IS the active style's UI root
    RETURN widescreen_resolved(active style's delimited root, name)
  IF path IS the default UI root
    RETURN widescreen_resolved(default delimited root, name)
  RETURN name
```

**Invariants** — the two branches differ only in which root the existence check for the
`_16` variant runs against, so a style may ship a widescreen variant of a screen for which
the default has none, and vice versa.

**Notes** — the whole body is compiled only into the shared-library build of this module and
vanishes in the statically linked one, where the substitution therefore never happens. That
is a build-configuration hazard, not a design decision: the rule should apply
unconditionally, and a rebuild should make it so.
