# src/xrGame/ui/UIServerInfo.cpp

> The multiplayer pre-round briefing: a map picture and a description, either authored in the level
> data or pushed by the server, with "join" and "spectate" as the only two ways out.

**Needs** — [`UIServerInfo.h`](UIServerInfo.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIMapInfo.h`](UIMapInfo.h.md) · [`../UIGameCustom.h`](../UIGameCustom.h.md) · [`../Level.h`](../Level.h.md) · [`../game_cl_mp.h`](../game_cl_mp.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrCore/Media/Image.hpp`](../../xrCore/Media/Image.hpp.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIServerInfo.h`](UIServerInfo.h.md)
**Tier floor** — T1: decodes a compressed image out of a network payload and hands the bytes to the
texture loader through a temporary file

## Purpose

Shown once when a client joins a multiplayer round, before the player picks a team or a skin. Its
content comes from one of two places: the level's own authored description, or two payloads the
server pushes (a logo image and a rules text). The screen exists in two shipped layouts under two
different element vocabularies, and this file picks whichever one the installed data provides —
which is why nearly every element here is optional.

## State

```text
RECORD BriefingScreen extends ModalDialog
  image      : Widget             # map picture, or the server's logo
  text_body  : optional<Widget>   # present only in the newer layout
  has_info   : bool               # true once anything was actually displayed
```

Invariants:

- `has_info` is the screen's only output. It is set when a description was found or when the server
  pushed either payload, and the caller uses it to skip showing an empty briefing.
- In the older layout `text_body` is absent, and the description path goes through the map-info
  widget instead; the two paths are mutually exclusive.

## `CUIServerInfo`

**Contract** — Builds the screen, choosing its layout document and element vocabulary by what is
installed, then fills in the picture and the description. Allocates; does not block; runs once per
round.

```text
FUNCTION construct()
  IF document "server_info.xml" loads THEN root <- "server_info"
  ELSE  load "map_desc.xml";              root <- "map_desc"
  build window from document[root]; descend into it

  background, caption            <- statics (required)
  image                          <- static (required)
  choose the picture:
    IF game config names a texture for this level THEN use it
    ELSE IF the conventional per-level picture file exists THEN use it
    ELSE use the noise placeholder
  restore the image's authored texture rectangle and stretch the picture into it

  text_view  <- scroll container (required)
  text_body  <- static (optional)
  IF text_body exists THEN
    text_body.complex_text_mode <- true          # wrapping and inline markup
    text_body.width <- text_view.desired_child_width
    text_view.insert(text_body)
  ELSE
    map_info <- map-info widget built from document["map_info"]
    map_info.load(level name)
    IF map_info has a long description THEN
      add it to text_view as text
      has_info <- true

  three optional decorative frames
  next_button, spectator_button  <- buttons (required)
  bind next_button      -> join
  bind spectator_button -> spectate
```

**Notes** — The per-level picture is found by convention: a fixed prefix plus the level's name. The
lookup order — explicit configuration, then convention, then placeholder — means a level that ships
no picture gets a neutral one rather than a missing-texture artifact. Restoring the authored texture
rectangle after binding the texture is what keeps all three sources framed identically despite
having different pixel dimensions.

## `SetServerLogo`

**Contract** — Takes a compressed image payload the server sent, decodes it to validate it, writes
the payload to a fixed temporary file under the saves root, binds the picture widget to that file,
then deletes it. Reports and gives up if either the decode or the file creation fails. Sets
`has_info` on success.

```text
FUNCTION SetServerLogo(bytes)
  IF NOT decode_as_jpeg(bytes) THEN report and RETURN
  writer <- create temporary file "tmp_sv_logo.dds" under the saves root
  IF writer IS none THEN report and RETURN
  write bytes verbatim; close
  image.bind_texture(that file)
  delete the file
  has_info <- true
```

**Notes** — This is the one place in the chapter where the recipe records a defect rather than a
decision. The payload is a compressed photographic image, the texture loader wants the engine's own
texture format, and the code **writes the compressed payload out under the texture format's name
without converting it** — the decode above it is only a validity check, its result discarded. The
source marks this with a to-do. A rebuild should decode once and hand the decoded surface to the
texture loader; both the temporary file and the round trip through the filesystem exist only because
the texture loader is addressed by path.

The bind-then-delete ordering assumes the texture loader reads the file synchronously during the
bind. A rebuild with an asynchronous loader must keep the file alive until the load completes.

## `SetServerRules`

**Contract** — Takes a text payload the server sent and puts it in the description, if this layout
has a description widget at all. Truncates to a fixed buffer, then rewrites every carriage-return /
line-feed pair into the engine's own two-character newline escape, in place, so the text widget's
markup parser breaks the lines. Grows the widget to fit and sets `has_info`.

```text
FUNCTION SetServerRules(bytes)
  IF text_body IS none THEN RETURN                  # older layout: nothing to put it in
  text <- first 4095 bytes of the payload, terminated
  FOR EACH occurrence of CR-LF IN text
    replace the two bytes in place with (escape marker, 'n')
  text_body.text <- text
  text_body.fit_height_to_text()
  has_info <- true
```

**Notes** — The rewrite is in place and two bytes to two bytes precisely because it must not change
the length; a one-to-two expansion would need a second buffer. Bare line feeds and bare carriage
returns are left alone, which is a real limitation for servers that do not send both. The 4095-byte
cap is the buffer the text is staged in, not a protocol limit.

## `OnKeyboardAction`

**Contract** — The jump and enter actions both mean "join" — the same as pressing the next button.
Nothing else is handled here, so everything else falls through to normal routing.

## Joining and spectating

**Contract** — Both exits hide the screen first and then tell the multiplayer game state which was
chosen; the ordering matters, because the game state may immediately show another screen. Both reach
a matchmaking-era service described at
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts),
which no longer exists — the screen still builds and still works against a private server, but the
path that would have pushed the logo and rules is unreachable without one.
