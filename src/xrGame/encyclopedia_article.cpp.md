# src/xrGame/encyclopedia_article.cpp

> Parses one authored article out of XML — title, group, body, icon and type — and defines how the player's copy of an article is persisted.

**Needs** — [`encyclopedia_article.h`](encyclopedia_article.h.md) · [`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`ui/UIInventoryUtilities.h`](ui/UIInventoryUtilities.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — reached through its declarations in [`encyclopedia_article.h`](encyclopedia_article.h.md); callers name that, not this file.
**Tier floor** — T3: XML parsing into a display record

## Purpose

Everything the player can read in the game — encyclopedia entries, found documents, mission
briefings, notifications — is one of these. This file turns the authored XML into the record
the document screens draw, and it defines the save format of the player's own list of
articles.

## State

See [`encyclopedia_article.h`](encyclopedia_article.h.md) and
[`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md).

## `ARTICLE_DATA::load` · `ARTICLE_DATA::save`

**Contract** — the player's article record, serialized in field order: receive time,
identifier, read flag, type. See the frozen-format invariant in
[`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md).

## `load_shared` — parsing one article

**Contract** — fills the shared payload from the XML element the identifier map located.
A missing element is a hard failure naming the identifier. Called once per article per game
run; the payload is shared thereafter.

```text
FUNCTION load_shared()
  node = the XML element at the recorded (file, position) for this identifier
  FAIL IF absent
  text  = the node's `text` child
  name  = the node's `name` attribute
  group = the node's `group` attribute

  # the icon, from one of two sources
  IF the node names an `ltx` configuration section THEN
    icon shader = the shared inventory-icon atlas
    icon rect   = that section's inventory grid cell, scaled by the grid cell size
  ELSE IF the node has a `texture` child THEN
    build the icon from that element the same way any UI texture is built

  IF the icon has a shader THEN enforce a minimum size (see below)

  article_type = the `article_type` attribute, mapped case-insensitively from
                 "encyclopedia" / "journal" / "task" / "info"; an unrecognized value
                 is LOGGED and leaves the type at whatever it was
  ui_template_name = the `ui_template` attribute, defaulting to "common"
```

**Invariants** — the icon has two authoring routes and they mean different things. The
configuration route reuses the **inventory icon atlas**, addressing a cell by grid
coordinates — so an article about a weapon shows that weapon's inventory icon with no
duplicated art. The XML route points at an arbitrary texture. A rebuild must support both,
because the shipped data uses both.

**Invariants** — the grid rectangle is built as (origin, size) and then converted in place to
(origin, corner) by adding the origin to the size. Getting that wrong yields icons anchored
at the atlas origin.

**Invariants** — an unrecognized article type is **not** a failure and **not** a default: the
payload keeps whatever value the field already held, which for a freshly shared payload is
the encyclopedia type. It logs and continues. A rebuild that defaults explicitly is doing the
same thing more honestly.

## The minimum icon size

**Contract** — an icon smaller than **65** pixels in either dimension is padded to that size,
symmetrically, by growing the destination rectangle and offsetting the texture by half the
growth so the art stays centred.

**Invariants** — this centres small art in a fixed cell rather than scaling it up. The
document screens lay articles out on a grid and a one-cell inventory icon would otherwise sit
in the corner of its slot. Sixty-five is the authored cell size and appears nowhere else.

**Notes** — the widget is marked as not auto-deleting, because the shared payload owns it and
the widget tree it gets attached to must not free it. That is the other half of the
detach-on-destruction in the destructor, and the pair is the whole reason the payload holds a
widget at all.

## `Load`

**Contract** — records the identifier and asks the shared-payload machinery for it, which
either hands back an existing payload or calls `load_shared` to build one.

## `InitXmlIdToIndex`

**Contract** — declares, once, that articles are elements named `article` and that they live
in the files listed under the `encyclopedia` configuration section's `files` key. Both are set
only if not already set, so the first caller wins.

**Invariants** — the file list is configuration, not code, so a mod adds article files by
editing one key. That is the extension point for the entire document system.

## Destruction

**Contract** — detaches the shared icon widget from its parent if it has one. Required
because the widget outlives any one screen; see the note above.
