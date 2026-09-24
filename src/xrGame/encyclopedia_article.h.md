# src/xrGame/encyclopedia_article.h

> Declares the authored, shared side of an encyclopedia article, implemented in [`encyclopedia_article.cpp`](encyclopedia_article.cpp.md).

**Needs** — [`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md) · [`xml_str_id_loader.h`](../xrServerEntities/xml_str_id_loader.h.md) · [`shared_data.h`](../xrServerEntities/shared_data.h.md) · [`xrUICore/Static/UIStatic.h`](../xrUICore/Static/UIStatic.h.md)
**Used by** — [`GameTask.cpp`](GameTask.cpp.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`encyclopedia_article.cpp`](encyclopedia_article.cpp.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CEncyclopediaArticle`, one article as the game data authored it, and `SArticleData`,
the payload that is shared between every holder of that article. Substance in
[`encyclopedia_article.cpp`](encyclopedia_article.cpp.md).

Two mechanisms meet here and both are inherited rather than written:

- **Shared payload** — the article's content is reference-counted and loaded once. A hundred
  references to one article cost one copy of its text and one icon.
- **Identifier-to-index over XML** — articles are authored in a set of XML files named by
  configuration, and are addressed by string identifier. The inherited machinery scans those
  files once at startup and builds a map from identifier to (file, position), so loading an
  article is a seek rather than a search. See [`xml_str_id_loader.h`](../xrServerEntities/xml_str_id_loader.h.md).

## State

```text
RECORD ArticleData                    # the shared payload
  name             : text             # the title, a string-table key
  group            : text             # which section of the encyclopedia it files under
  text             : text             # the body, a string-table key
  image            : UIStatic         # the icon, as a ready-to-draw widget
  article_type     : ArticleType
  ui_template_name : text = "common"  # which authored layout renders it
```

**Invariant** — the payload holds a **live UI widget**, not an image description. The icon is
built at load time and shared, which is why the article's destructor has to detach it from
whatever parent last displayed it. That coupling between a content record and the widget tree
is the file's one real design cost, and a rebuild should store the texture rectangle and let
the screen build its own widget.

## Exported units

- construction and destruction — destruction detaches the shared icon from its parent.
- `Load(id)` — bind to an authored identifier and pull in the shared payload.
- `load_shared` — parse the payload out of XML; the substance.
- `InitXmlIdToIndex` — declare the element tag (`article`) and the list of files to scan,
  the latter read from configuration.
- `Id` / `data` — the identifier and the shared payload.
