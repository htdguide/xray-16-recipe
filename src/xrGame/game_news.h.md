# src/xrGame/game_news.h

> Declares one news item — the record behind the message ticker that reports what the world and the plot are doing — implemented in [`game_news.cpp`](game_news.cpp.md).

**Needs** — [`game_news.cpp`](game_news.cpp.md) · [`xrServerEntities/alife_space.h`](../xrServerEntities/alife_space.h.md) · [`Common/object_interfaces.h`](../Common/object_interfaces.h.md)
**Used by** — [`Actor.h`](Actor.h.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) · [`game_news.cpp`](game_news.cpp.md) · [`UILogsWnd.cpp`](ui/UILogsWnd.cpp.md) · [`UILogsWnd.h`](ui/UILogsWnd.h.md) · [`UIMainIngameWnd.cpp`](ui/UIMainIngameWnd.cpp.md) · [`UIMessagesWindow.cpp`](ui/UIMessagesWindow.cpp.md) · [`UINewsItemWnd.cpp`](ui/UINewsItemWnd.cpp.md) · [`UITalkDialogWnd.cpp`](ui/UITalkDialogWnd.cpp.md)
**Tier floor** — T3: a serializable record

## Purpose

Declares the news record and the list it is kept in. Serialization is in
[`game_news.cpp`](game_news.cpp.md); everything else about the record is here.

## State

```text
RECORD NewsItem
  type          : { news, talk }   # a world report, or a line somebody said to you
  show_time     : int              # milliseconds the ticker holds it; default 5000
  caption       : text             # a localization key or a formatted name
  text          : text
  texture_name  : text             # the portrait or icon beside it
  receive_time  : game_time        # in-world time, not real time
```

**Invariants** — the timestamp is **in-world** time, so a news item's age is measured on the
same clock as everything else in the simulation and survives a save unchanged. The display
duration is per item rather than global because a plot message is authored to linger and a
simulation report is not.

The two types are the whole of the distinction the ticker needs: a *talk* item is attributed
to a speaker and is replayed in the conversation history, a *news* item is not.

**Notes** — the display duration is a signed integer and the only value the shipped data
uses is the default. It is per-item because a script can set it, not because the engine
varies it.

Exported units:

- `GAME_NEWS_DATA` — the record above, with load and save.
- `GAME_NEWS_VECTOR` — the list a character's news feed is kept in; ordering is arrival
  order, and the ticker reads from the end.
