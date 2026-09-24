# src/xrGame/UIGameMP.cpp

> What every multiplayer mode's interface has in common: the server's greeting screen, and the playback controls for a recorded match.

**Needs** — [`UIGameMP.h`](UIGameMP.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`Level.h`](Level.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`ui/UIDemoPlayControl.h`](ui/UIDemoPlayControl.h.md) · [`ui/UIServerInfo.h`](ui/UIServerInfo.h.md) · [`xrUICore/Cursor/UICursor.h`](../xrUICore/Cursor/UICursor.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two windows and their visibility rules

## Purpose

The thin layer between the in-game interface base and the individual multiplayer modes. It
owns exactly two things, and both exist because they are true of *every* multiplayer mode
and of no single-player session:

- **the server information screen** — a logo and a rules text the server pushes to the
  client on connection, which must be acknowledged before the match starts;
- **the recorded-match playback controls**.

Everything else a mode needs — scores, money, buy menus — belongs to the mode.

## State

```text
RECORD MultiplayerUI
  demo_play_control : optional<window>   # created on first use, never destroyed early
  server_info       : optional<window>   # recreated every time the session changes
  game              : the multiplayer game mode this interface serves
```

Invariant: the server-information window is destroyed and rebuilt on every session change,
because its content is the *new* server's. The playback controls are built once and reused.

## `ShowServerInfo`

**Contract** — show the server's greeting, if there is one. Returns whether the client may
proceed. The return value is the interesting part: it is not "was it shown" but "is this
gate satisfied".

```text
FUNCTION show_server_info() -> bool
  IF this is a recorded match THEN RETURN true        # nobody to greet
  REQUIRE the window exists                           # the client UI must have been created
  IF the window has no content THEN
    tell the game mode the information was accepted
    RETURN true
  IF it is not already shown THEN show it modally
  RETURN true
```

**Invariants** — a server with nothing to say **acknowledges on the client's behalf**. That
is what keeps the connection handshake from stalling on a server with no logo and no rules,
and it is why the acceptance notification appears in the "nothing to show" branch rather than
after the player dismisses the window.

**Notes** — the function returns true on every path it can reach, which makes the return value
meaningless as written. A disabled branch above it — for information the player aborted —
was the one that could return otherwise. Not recoverable as an intention; a rebuild should
decide what the gate means and say so.

## `SetClGame`

**Contract** — bind to a new session. Hides and destroys the previous server-information
window and builds a fresh one.

**Invariants** — the mode is required to be a multiplayer mode; a single-player session
reaching this layer is a programming error.

## `IR_UIOnKeyboardPress`

**Contract** — one key is intercepted here: while playing back a recorded match, the crouch
action opens the playback controls. Everything else falls through.

**Notes** — reusing the crouch action rather than adding a binding means a player watching a
recording needs no new key, and the action is meaningless during playback anyway.

## `ShowDemoPlayControl`

**Contract** — show the playback controls, creating them on first use, and **restore the
mouse cursor to where it was when they were last closed**.

**Notes** — the cursor restoration is a small courtesy with a real reason: the controls are
opened and closed repeatedly while scrubbing through a recording, and a cursor that snapped
to the screen centre each time would make that unusable.

## `SetServerLogo`, `SetServerRules`, `IsServerInfoShown`

**Contract** — three forwards into the server-information window. The two setters take raw
bytes as they arrived from the server — an image and a text — and the window decodes them.

**Notes** — the bytes are the server's, so a rebuild must treat both as untrusted input: the
image goes to an image decoder and the text to a text renderer, neither of which may be
allowed to fail fatally on malformed content.
