# src/xrGame/ui/UIMPPlayersAdm.cpp

> The administrator's player page: it asks the server for the connected roster, renders one
> row per client, and turns row-plus-button into a remote moderation command.

**Needs** — [`UIMPPlayersAdm.h`](UIMPPlayersAdm.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`Level.h`](../Level.h.md) · [`game_cl_mp.h`](../game_cl_mp.h.md) · [`game_cl_base.h`](../game_cl_base.h.md) · [`xrUICore/ListBox/UIListBox.h`](../../xrUICore/ListBox/UIListBox.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/TrackBar/UITrackBar.h`](../../xrUICore/TrackBar/UITrackBar.h.md) · [`xrUICore/ComboBox/UIComboBox.h`](../../xrUICore/ComboBox/UIComboBox.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIMPPlayersAdm.h`](UIMPPlayersAdm.h.md)
**Tier floor** — T3: a list, a deferred request, and text command assembly.

## Purpose

One of the three pages of the remote-administration screen, and the only one whose content
comes from the server rather than from local configuration. The roster is not read out of a
local table: the page issues a *request* to the multiplayer game layer and supplies a
continuation, because the client's own view of the player set carries names and addresses
only after the server answers. Everything else on the page is a command emitter.

## State

```text
RECORD PlayersAdminPage EXTENDS Window
  players_list     : ListBox    # element "players_list"; each row's tag is a client id
  refresh_btn      : Button
  screen_all_btn   : Button     # element "screen_all_button"
  config_all_btn    : Button
  ping_limit_btn   : Button
  ping_limit_track : TrackBar   # element "max_ping_limit_track"
  ping_limit_text  : Static
  screen_player_btn, config_player_btn, kick_player_btn, ban_player_btn : Button
  ban_player_combo : ComboBox   # element "ban_player_combo"; tokens are ban durations
```

**Invariants**

- Every roster row carries the client identifier as its tag. This is the only link between a
  highlighted row and the player it names, and every per-player action reads it. A rebuild
  must attach the identity to the row, not derive it from the row's position, because the
  roster is rebuilt asynchronously and rows can move under the selection.
- The ping slider holds *tenths of the real limit*: its value is the ceiling divided by ten.
  The slider's authored range is small enough that the real millisecond range would not fit,
  so every read multiplies by ten and the initial load divides — rounding up.

## The ban-duration token set

A fixed, ordered table mapping a localization identifier to a duration in seconds, used to
populate the ban drop-down. Its span is deliberately logarithmic — ten minutes, thirty
minutes, an hour, six hours, a day, a week, a month, three months, and a sentinel meaning
*forever*.

```text
RECORD BanDuration
  label_id : text     # localization identifier, e.g. "ui_mp_am_1_hour"
  seconds  : int
```

**Notes** — *forever* is expressed as a very large second count (about thirty-two years)
rather than as a distinguished value, so the receiving side needs no special case. A rebuild
that wants a real sentinel must change both ends.

## `Init`

**Contract** — configure this window and its thirteen children from the `players_adm` section
of the already-open layout document; request the roster; seed the ping slider from the
server's current ceiling; render the ping label; and put the ban drop-down on its first
entry. Element names are frozen by the shipped document.

**Notes** — the slider is seeded through a global the settings protocol reads, not by calling
the slider: the slider is a *settings control* bound to a console variable, and the supported
way to preload one is to write the variable's backing value and then tell the control to pull
it. The value written is the server's current ceiling divided by ten and rounded **up**, so
the displayed ceiling is never lower than the one actually in force.

## `RefreshPlayersList` and `FillPlayersList`

**Contract** — the refresh asks the multiplayer game layer to fetch player information and
hands it a continuation; it does nothing at all when the running game is not a multiplayer
one. The continuation clears the roster and adds one row per known player, the row text being
name, identifier, address and ping joined into one line, and the row tag being the identifier.
Neither call blocks; the roster is empty between them.

```text
FUNCTION refresh()
  game <- current_game AS multiplayer_game
  IF game IS none THEN RETURN              # single-player: the page is inert
  game.request_players_info(THEN fill_players_list)

FUNCTION fill_players_list(_)
  players_list.clear()
  FOR EACH (id, player) IN current_game.players
    row <- players_list.add_text_row(
             format("{}, id:{}, ip:{}, ping:{}", player.name, id, player.address, player.ping))
    row.tag <- id
```

**Notes** — the continuation's one argument is unused; the callback signature is shared with
other consumers of the same request. The request is fire-and-forget: nothing re-arms it, so
the roster is only as fresh as the last press of the refresh button.

## The four per-player actions

**Contract** — each reads the highlighted row's tag and issues one remote-administration
command naming that client: capture a screenshot, dump the client's configuration, kick, or
ban for the duration currently selected in the drop-down. Each does nothing when no row is
highlighted. The ban duration passed is the drop-down's current token value, i.e. seconds.

## The two all-player actions and the ping ceiling

**Contract** — the two all-player buttons issue the screenshot and configuration-dump
commands with no target. The ping button commits the slider's value times ten as the server's
ping ceiling; moving the slider only re-renders the label, which reads as the localized
phrase followed by the ceiling in milliseconds. Commitment is therefore explicit: dragging
the slider changes nothing on the server until the button is pressed.

## `SendMessage`

**Contract** — every button click is dispatched by identity to one of the actions above. The
slider is handled in the same click branch, which is how a slider drag reaches the label
refresh.
