# src/xrGame/ui/UISpawnWnd.cpp

> The team picker: two selectable pictures, three buttons, and five keys, all resolving to one
> question — team 0, team 1, automatic, spectate, or back.

**Needs** — [`UISpawnWnd.h`](UISpawnWnd.h.md) · [`UIStatix.h`](UIStatix.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`../UIGameCustom.h`](../UIGameCustom.h.md) · [`../Level.h`](../Level.h.md) · [`../game_cl_teamdeathmatch.h`](../game_cl_teamdeathmatch.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UISpawnWnd.h`](UISpawnWnd.h.md)
**Tier floor** — T3: a five-way choice over a layout document

## Purpose

Shown before spawning in a team game. Structurally the simplest screen in the chapter: every
interaction collapses to one call into the multiplayer game state carrying a team number, where
**-1 means "pick for me"**. The two team pictures are selectable-picture widgets (see
[`UIStatix`](UIStatix.cpp.md)), which is how a picture behaves as a button.

## State

```text
RECORD TeamPicker extends ModalDialog
  team_a, team_b : SelectablePicture
  auto_select, spectator, back : Button
  current_team : int     # -1 not yet chosen, 0 or 1
```

Invariants:

- At most one of the two pictures is in its selected state; `SetCurTeam` enforces this by setting
  both from the same value.
- The screen holds no notion of which teams are available — that is the game state's business, and
  it hides buttons via `SetVisibleForBtn` rather than the screen deciding.

## `CUISpawnWnd`

**Contract** — Builds from the fixed layout document `spawn.xml` under the element `team_selector`.
The frozen element names are `background`, `caption`, `image_0`, `image_1`, `image_frames_tl`,
`image_frames_tr`, `image_frames_bottom` (optional), `text_desc`, `btn_autoselect`,
`btn_spectator`, `btn_back`.

**Notes** — The two team pictures are attached and configured but **not given textures**. The helper
that would load them from the `team_logo` configuration section exists and is not called — the
shipped layout supplies the pictures' textures itself, and the configuration path is dead. It is
kept here because the configuration section it reads is still shipped and a rebuild that revives the
path will find the data waiting.

## `SendMessage`

**Contract** — Any click from any of the five interactive elements hides the screen and reports the
choice. Ordering matters: hide first, then report, because the game state may open another screen
immediately.

```text
FUNCTION SendMessage(sender, message, payload)
  IF message == BUTTON_CLICKED THEN
    hide()
    IF sender == team_a       THEN game.choose_team(0)
    ELSE IF sender == team_b  THEN game.choose_team(1)
    ELSE IF sender == auto    THEN game.choose_team(-1)
    ELSE IF sender == spectator THEN game.choose_spectator()
    ELSE IF sender == back    THEN game.back_out()
  base.SendMessage(sender, message, payload)
```

## `OnKeyboardAction`

**Contract** — The **scores** action, held, hides the screen's children and shows the scoreboard,
restoring both on release — the same arrangement as the skin picker, so the scoreboard stays
reachable from a screen that captures the keyboard. Otherwise: the `1` and `2` keys choose teams 0
and 1 directly; quit backs out; jump auto-selects; enter confirms **whichever picture is currently
selected**, falling back to automatic if neither is.

```text
FUNCTION on_enter()
  hide()
  IF team_a.selected      THEN game.choose_team(0)
  ELSE IF team_b.selected THEN game.choose_team(1)
  ELSE                         game.choose_team(-1)
```

**Notes** — The digit keys are matched as raw keys rather than bound actions, so they are not
rebindable — deliberate, because they pair with the "1." and "2." labels the layout paints on the
pictures.

## `SetCurTeam`

**Contract** — Records the team and sets exactly one picture's selected state, rejecting anything
outside -1, 0, 1. Selecting a picture starts its attention animation; see
[`UIStatix`](UIStatix.cpp.md).

## `SetVisibleForBtn`

**Contract** — Shows or hides one of the three named buttons. Rejects an unknown name. This is the
only way the game mode narrows the screen's choices — for instance, hiding "back" when there is
nothing to go back to.
