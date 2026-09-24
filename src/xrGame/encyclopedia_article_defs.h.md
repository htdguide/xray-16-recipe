# src/xrGame/encyclopedia_article_defs.h

> What the player's copy of an article is — the identifier, when they received it, whether they have read it, and which of the four in-game document types it is.

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`Common/object_interfaces.h`](../Common/object_interfaces.h.md)
**Used by** — [`GameTask.h`](GameTask.h.md) · [`InfoPortion.h`](InfoPortion.h.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) · [`encyclopedia_article.cpp`](encyclopedia_article.cpp.md) · [`encyclopedia_article.h`](encyclopedia_article.h.md) · [`UIPdaWnd.h`](ui/UIPdaWnd.h.md)
**Tier floor** — T3: a persisted record

## Purpose

Separated from [`encyclopedia_article.h`](encyclopedia_article.h.md) because this is the
*player's* side of an article — a saved, per-playthrough record — while that file is the
*authored* side, loaded from data and shared between every player who has it. The split is a
real one and worth keeping: one is save-game state, the other is a content asset.

## State

```text
RECORD ArticleData                        # one entry in the player's document list
  article_id   : text                     # names an authored article
  receive_time : int (64-bit)             # the game clock at which it arrived
  readed       : bool = false             # the "new" marker in the UI
  article_type : ArticleType = encyclopedia
```

```text
ENUM ArticleType
  encyclopedia   # a reference entry, filed by group
  journal        # a story document, filed by when it arrived
  task           # a mission briefing
  info           # a one-off notification
```

**Invariant** — the record stores an identifier, not content. The article's text, image and
title come from the authored side and are shared; duplicating them per player would mean the
save file carried the game's whole encyclopedia.

**Invariant** — all four fields are serialized, in declaration order, with no version tag.
The type is written as its enumeration value, so the four values are **frozen by every
existing save**: reordering the enumeration silently reclassifies every document the player
has collected.

**Invariant** — the type is stored *in the player's record* even though it is a property of
the authored article. It is a denormalization; it lets the UI file a document without
resolving the authored article, and it means an article whose authored type changes keeps its
old type in existing saves.

**Notes** — the spelling of the read flag is a typo frozen in the field name only, not in the
format.

## `load` · `save`

**Contract** — read and write the four fields in order, through the shared serialization
vocabulary. No validation: an identifier naming an article the current game data does not
have is stored and reloaded happily, and fails later when the UI tries to display it.

## `ARTICLE_ID_VECTOR` · `ARTICLE_VECTOR`

**Contract** — a list of identifiers and a list of records. Both appear in the player's saved
state.

## `FindArticleByIDPred`

**Contract** — a predicate matching a record by identifier. Exists because the player's
document list is a flat list searched linearly; there is no index. The lists run to a few
hundred entries at most.
