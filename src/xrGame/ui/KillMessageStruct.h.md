# src/xrGame/ui/KillMessageStruct.h

> The shape of one "X killed Y with Z" line in the multiplayer kill feed.

**Needs** — [`../../xrUICore/ui_defs.h`](../../xrUICore/ui_defs.h.md)
**Used by** — [`UIGameDM.cpp`](../UIGameDM.cpp.md) · [`game_cl_mp.cpp`](../game_cl_mp.cpp.md) · [`UIPdaKillMessage.h`](UIPdaKillMessage.h.md)
**Tier floor** — T3: a record declaration

## Purpose

A pure data declaration, shared between the code that composes a kill message from a network
event and the widget that lays it out. It is a separate header only so those two need not
include each other.

## State

```text
RECORD ColouredName
  text   : text        # already resolved: a player name, not a localization identifier
  colour : int (32-bit, packed ARGB)   # the team colour of that player

RECORD Icon
  rect     : rect      # sub-rectangle of the icon atlas
  material : material  # the atlas page, resolved at compose time

RECORD KillMessage
  victim    : ColouredName
  initiator : Icon      # the weapon or cause
  killer    : ColouredName
  extra     : Icon      # a modifier: headshot, a special kill
```

Invariants worth stating, because nothing in the type enforces them:

- The two names are **already localized and already coloured**. The kill feed does not look
  up a string table entry at draw time; the colour is the team colour resolved when the event
  arrived, so a player who changes team afterwards keeps the old colour on old lines.
- Either icon may be empty, and an empty icon is drawn as nothing rather than as a blank box —
  a kill with no special modifier omits the second icon entirely and the line closes up.
- The field order is the reading order of the composed line: victim, cause, killer, modifier.
  The widget relies on that, which is why the record is ordered this way and not alphabetically.
