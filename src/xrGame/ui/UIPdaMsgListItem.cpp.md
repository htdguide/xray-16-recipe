# src/xrGame/ui/UIPdaMsgListItem.cpp

> Builds one message-log row from its own small layout document, attaching only the widgets that
> document actually defines.

**Needs** — [`UIPdaMsgListItem.h`](UIPdaMsgListItem.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md)
**Used by** — [`UIPdaMsgListItem.h`](UIPdaMsgListItem.h.md)
**Tier floor** — T3.

## Purpose

A row shape shared by every on-screen message, defined in a document of its own so that the
overlay, the PDA and the multiplayer log all get the same row. The file's only decision is
which widgets are mandatory.

## `InitPdaMsgListItem`

**Contract** — set the row's size from the caller, open the shared row document, and build: the
icon, always and mandatorily; the timestamp, caption and body, each only if the document
defines it, the body under either of two element names; and a fourth, unnamed picture that is
created and adopted but never referred to again.

```text
FUNCTION init(size)
  self.size <- size
  doc <- load_layout("maingame_pda_msg.xml")
  adopt(icon) configured from "icon_static"                       # required
  IF doc HAS "time_static"    THEN adopt(time) configured from it
  IF doc HAS "caption_static" THEN adopt(caption) configured from it
  IF doc HAS "msg_static" OR "text_static" THEN adopt(body) configured from whichever
  IF doc HAS "name_static"    THEN create and adopt an anonymous picture
```

**Notes** — the anonymous picture is pure decoration the row never touches: a nameplate or
divider that some layouts supply. It is created here rather than by the caller because the
caller does not open this document. A rebuild treats it as a static part of the row template.

The row is not laid out here. Its contents' positions come from the document and its *height*
is set by whoever fills it in, after the body's text is wrapped.

## `SetFont`

**Contract** — apply one font to the timestamp, caption and body. The icon is unaffected.
