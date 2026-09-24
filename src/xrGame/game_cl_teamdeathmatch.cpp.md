# src/xrGame/game_cl_teamdeathmatch.cpp

> Team deathmatch on the client: the team-selection gate in front of everything else, friendly identification, the two-team scoreboard, and the lead-change announcements.

**Needs** — [`game_cl_teamdeathmatch.h`](game_cl_teamdeathmatch.h.md) · [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md) · [`UIGameTDM.h`](UIGameTDM.h.md) · [`game_cl_teamdeathmatch_snd_messages.h`](game_cl_teamdeathmatch_snd_messages.h.md) · [`game_base_menu_events.h`](game_base_menu_events.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`xrServerEntities/clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`ui/TeamInfo.h`](ui/TeamInfo.h.md) · [`ui/UISkinSelector.h`](ui/UISkinSelector.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`ui/UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)

**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: session rules and screen state over the network layer

## Purpose

Deathmatch already supplies rounds, frags, the buy menu, the skin menu and the ready
sequence. Team deathmatch inserts **one more step in front of all of it** — choosing a team
— and then makes every other screen and every other question team-aware.

Four things change:

1. **A team must be chosen before anything else.** The skin menu, the buy menu, the ready
   signal and the inventory are all gated on it, and losing a skin selection invalidates the
   team selection too.
2. **Who is an enemy is a team comparison**, not "everybody else". This is the predicate
   friendly fire, the map, the name tags and the indicators all consult.
3. **Teammates are visibly identified** — names over their heads, an indicator under them,
   markers on the map — and the server decides per match whether either is allowed.
4. **The score is a pair**, and crossing or restoring a tie between the two is announced
   aloud.

## State

```text
RECORD TeamDeathmatchClient EXTENDS DeathmatchClient
  game_ui               : TeamDeathmatchUI        # the mode's screen, re-resolved on connect
  team_selected         : bool                    # the gate in front of everything
  show_players_names    : bool                    # the local preference
  friendly_names        : bool                    # the server's permission, from the snapshot
  friendly_indicators   : bool                    # the server's permission, from the snapshot
  preset_items_team_1   : PresetItems             # the default loadout per team,
  preset_items_team_2   : PresetItems             #   loaded once and swapped on team change
```

Invariants:

- **`team_selected` is cleared whenever the skin selection is lost.** The two are a unit: a
  team implies a skin set, so an invalidated skin re-opens the team question. It is also
  cleared when the player chooses to spectate.
- **Team numbering is one-based here and zero-based in the scoreboard.** The player's team
  is 1 or 2; the score array is indexed 0 and 1; a dedicated conversion (`ModifyTeam`)
  subtracts one and passes -1 through unchanged. Every reader must be clear which it holds,
  and the conversion is the only sanctioned bridge.

## `net_import_state`

**Contract** — decodes the mode's snapshot: the inherited deathmatch state followed by two
permission flags — may teammate names be shown, may teammate indicators be shown. Detects a
tie being broken or restored **across** the decode and announces it.

```text
FUNCTION net_import_state(packet)
  was_tied = teams exist AND team[0].score == team[1].score
  base.net_import_state(packet)                 # updates the team scores
  friendly_indicators = packet.read_byte() != 0
  friendly_names      = packet.read_byte() != 0

  IF teams exist AND there is a view entity
    IF was_tied AND scores now differ
      announce(team[0] leads) or announce(team[1] leads)
    ELSE IF NOT was_tied AND scores now equal
      announce(teams are level)
```

**Invariants** — the tie state must be sampled **before** the base decode, because the base
decode is what overwrites the scores. That read-before-delegate ordering is the whole
mechanism; a rebuild that samples afterwards announces nothing, ever.

**Notes** — the announcement is suppressed when there is no view entity, which is the
loading and disconnected case. A lead change is only interesting to someone watching.

The two permission flags are **server policy pushed to the client every snapshot**, not
local settings: a match can forbid teammate names, and the client obeys. The local
preference is separate and is ANDed with the permission at draw time.

## `TranslateGameMessage`

**Contract** — renders two mode-specific events into the message feed: a player joining a
team, and a player switching teams. Both name the team in the team's own colour. Anything
else falls through to deathmatch.

**Notes** — the switch message prints the old team's colour on the player's name and the new
team's colour on the destination, so the feed reads as a transition rather than as a
statement. Both messages also go to the log. Text comes from the string table, never from a
literal — the surrounding format and the colour tags are the only literals.

## `OnTeamSelect` / `OnGameMenuRespond_ChangeTeam`

**Contract** — the request and the reply of a team change. The request is sent unless the
player picked the team he already has **and** still has a valid skin, in which case nothing
goes to the server. Sending one invalidates the skin selection. The reply applies the new
team, fires the team-changed hook if it actually changed, and re-opens the skin menu.

```text
FUNCTION on_team_select(chosen)            # chosen is zero-based from the menu
  send = true
  IF chosen is not "none"
     AND chosen + 1 == my team AND skin still selected
    send = false
  IF send
    event = new game event from my entity
    event.write_int16(PLAYER_GAME_MENU)
    event.write_byte(PLAYER_CHANGE_TEAM)
    event.write_int16(chosen + 1)          # one-based on the wire
    send(event)
    skin_selected = false                  # the new team has a different skin set
  team_selected = true

FUNCTION on_change_team_response(packet)
  old = my team
  my team = packet.read_int16() masked to a byte      # see Notes
  IF old != my team: on_team_changed()
  rebuild the skin menu for the new team
  show it, preselected to my current skin, if it may be shown
```

**Notes** — the reply truncates the team to a byte after reading a signed sixteen-bit value.
The wire carries a wider field than the value needs; the mask is what actually assigns it.
Reproduce the field width, not the mask.

`team_selected` is set optimistically, before the server has agreed. That is deliberate — the
gate is a *local* one about which screens to show, and blocking the player's progress on a
round trip would stall the pre-round flow. The server's reply only corrects the team number.

## `SetCurrentBuyMenu` / `SetCurrentSkinMenu`

**Contract** — build the buy menu and the skin menu for the local player's team. The buy menu
is built once, with that team's authored default loadout and the rank-based default items,
and thereafter only has its ignore-money flag toggled. The skin menu is *rebuilt* whenever
the team changes, hiding the old one first if it is on screen.

```text
FUNCTION set_current_buy_menu()
  IF no local player OR no team OR no skin: RETURN
  IF no buy menu yet
    team_index = (my team == 1) ? 1 : 2
    buy_menu = init_buy_menu(base_cost_section, team_index)
    load that team's default preset items into it
    current_preset = that team's preset list
    load the rank-based default items
  buy_menu.ignore_money_and_rank = (warm-up time is non-zero)
```

**Notes** — during warm-up the buy menu ignores money and rank entirely, so everyone can
equip anything. That is the warm-up's purpose, and the flag is re-evaluated every time the
menu is opened rather than watched, which is why this runs on every call.

The two preset item lists are held as separate members rather than one swapped list, because
a player switching teams repeatedly should not re-read the configuration each time. The buy
menu itself *is* thrown away and rebuilt on a team change (see the team-changed hook), so
only the presets are cached.

The team index collapses anything that is not team 1 to team 2. There are exactly two teams;
the expression exists because the team field can also hold the spectator value.

## The `CanCall…` gate family

**Contract** — five predicates deciding which screens may open. All of them are false while
the team-select window is up; most are also false until a team is chosen.

```text
FUNCTION can_call_team_select_menu() -> bool
  IF the round is not in progress: RETURN false
  IF no local player:              RETURN false
  IF the actor menu, the buy menu or the skin menu is shown: RETURN false
  preselect the window to my current team ; RETURN true

FUNCTION can_call_buy_menu() -> bool
  round in progress, team selected, skin selected, not spectating,
  the team-select and skin windows closed, the actor menu closed,
  the buy menu's data is ready, and buying is currently enabled

FUNCTION can_call_skin_menu() / can_call_inventory_menu() -> bool
  the team-select window is closed (and, for skins, a team is chosen),
  then whatever deathmatch says
```

**Invariants** — exactly one of these windows is ever open. The mutual exclusion is written
out longhand in each predicate rather than held as a state machine, which is why the same
"is that window shown" test appears many times.

**Notes** — the team-select predicate has a side effect: it preselects the window to the
player's current team. A predicate that mutates is a trap, and a rebuild should split the
preselection out — every caller here immediately opens the window, so nothing depends on
the coupling.

## `shedule_Update`

**Contract** — the per-update screen pass. Closes the team-select window if it has become
illegal, refreshes the score, and in each phase puts the right prompt on screen.

```text
FUNCTION scheduled_update(delta)
  base.scheduled_update(delta)
  IF no game ui: RETURN
  IF the team-select window is shown AND may no longer be: hide it
  clear the buy prompt

  SWITCH phase
    TEAM1_SCORES / TEAM2_SCORES:
      caption = "<team> wins" ; refresh the team panels ; show the player list ; set score
    IN_PROGRESS:
      IF I am playing and currently spectating
         and no menu is up and the indicators are shown
         and no team is chosen: prompt "press jump to select a team"
      set score
      buy_enabled = on base and not permanently dead, OR permanently dead
```

**Notes** — the buy rule is the interesting line and it reads strangely: buying is enabled
while standing on a base, disabled while away from it, and enabled again once permanently
dead. The last case is the round-over state, where a dead player is spending on his next
round's loadout. So "on base" and "out of the round" are the two times you may shop, and
being alive in the field is the only time you may not.

The team-select window is closed *reactively*, by testing its own legality every update,
rather than by whoever made it illegal. That is a robust shape for a screen that many
unrelated events can invalidate, and a rebuild should keep it.

## `OnRender`

**Contract** — draws a name and an indicator over each living teammate, but only for the
local player looking through his own eyes, and only where both the local preference and the
server's permission agree. Enemies, the dead and the local player himself are skipped.

```text
FUNCTION on_render()
  IF I am the player being looked through
     AND (names wanted OR indicators permitted)
    style = my team's visual style
    FOR EACH player state
      skip if permanently dead, not a spawned actor, an enemy, or me
      IF names wanted
        draw the name above the style's indicator position, in team green
        remember the text height
      IF indicators permitted
        draw the indicator at the style's position, offset upward by that height
  base.on_render()
```

**Notes** — the name's height is measured and fed back as the indicator's vertical offset,
so the two stack instead of overlapping. That coupling is the only reason the name is drawn
first.

The name colour is a fixed green rather than the team's colour — teammates are always green
from your point of view, whichever team you are on. That is a readability decision: the
colour means "friendly", not "team two".

Note the asymmetry: the name test consults the *local preference* and the indicator test
consults the *server permission*. The corresponding checks (server permission for names,
preference for indicators) are missing — the name's permission test is present in the source
but commented out. A rebuild should apply both gates to both.

## `UpdateMapLocations` / `GetMapEntities`

**Contract** — teammates appear on the map and on the minimap; enemies and the dead do not.
The map-location pass adds a friendly marker for each living teammate, removes it when he
dies or turns out to be an enemy, and is idempotent. The minimap pass yields one green point
per living teammate.

**Notes** — both are re-derived every update from the player list rather than maintained on
events. Wasteful and correct: team membership changes at arbitrary times and a missed event
would leave an enemy permanently marked as a friend.

## `IsEnemy`

**Contract** — two spellings. Against a player record: different team from the local player.
Against two live entities: different team numbers. Both are the foundation for friendly
fire, identification and the map.

**Notes** — the player-record form compares against the **local** player and not against a
supplied observer, so it is strictly "is this person my enemy". A spectator with no local
player is told nobody is an enemy.

## `GetTeamMenu` / `getTeamSection` / `GetBaseCostSect`

**Contract** — the configuration section names for the two teams' menus and for the price
list, and the per-team section a loadout is read from. Three names, keyed by team number.

**Notes** — team 0 has a menu section of its own, used for the pre-selection state before a
team is chosen; teams 1 and 2 are the real ones. The section-name lookup used for loadouts
omits team 0 and yields nothing for it, which is the difference between the two nearly
identical lookups in this file.

## `OnMapInfoAccept` / `OnSkinMenuBack` / `OnTeamMenuBack` / `OnTeamMenu_Cancel`

**Contract** — the pre-round navigation. Accepting the map briefing opens team select;
backing out of the skin menu returns to team select; backing out of team select shows the
server information only for a spectator.

Cancelling team select is the interesting one: if the player has chosen neither a team nor
spectating, the window is **reopened immediately**.

```text
FUNCTION on_team_menu_cancel()
  hide the team-select window
  IF no team chosen AND not spectating
    IF the window may be shown and is not: show it again ; RETURN
  menu_called_from_ready = false
```

**Notes** — that is a deliberate modal trap: there is no way out of team select except
choosing a team or choosing to spectate. Hiding and immediately reshowing rather than
refusing to hide is a UI-framework artefact; the decision is that the choice is mandatory.

## `CanBeReady`

**Contract** — the ready signal is refused while no team is chosen, and refusing opens the
team-select window instead. Otherwise deathmatch decides.

**Notes** — this is the gate's most visible effect: pressing ready before picking a team
takes you to team select rather than doing nothing. The flag marking "this menu was opened
by the ready key" is set and then cleared again on that path, so the menu does not
auto-continue afterward.

## `OnSwitchPhase` / `OnSwitchPhase_InProgress`

**Contract** — on entering either team's victory phase, play that team's win announcement
(only if there is a view entity). On entering the in-progress phase, hide the buy menu and
clear the team selection **if the skin selection was also lost**.

**Notes** — the conditional clearing is the round-to-round rule: a player who kept his skin
keeps his team through the round boundary and spawns straight in; a player whose skin was
invalidated goes back through the whole pre-round flow.

## `OnTeamChanged`

**Contract** — destroys and rebuilds the buy menu for the new team, then delegates.

**Notes** — the buy menu is rebuilt rather than reconfigured because the price list, the
available items and the default loadout are all per team. The *preset* lists survive, which
is the cache described above.

## `LoadSndMessages` / `PlayRankChangesSndMessage`

**Contract** — loads fourteen announcements: each team winning, each team taking the lead,
the teams levelling, and four rank-up announcements per team. The rank announcement picks
the local player's team's variant, indexed by the new rank, and says nothing at rank zero.

**Notes** — the rank announcements are per team because they are spoken in the team's own
voice. Rank zero is silent because it is the starting rank, not an achievement.

## `SetScore` / `GetGameScore` / `GetGreenTeamScore` / `GetBlueTeamScore`

**Contract** — push the two scores to the screen, and render them as a bracketed pair for
the compact display. The accessors name the two teams by colour.

**Notes** — the screen update is guarded on the local player having a non-negative team,
which excludes a spectator; the compact form is not. Spectators therefore see the score in
the compact display but not in the main caption. That looks unintended.

## `createGameUI` / `SetGameUI` / `OnConnected`

**Contract** — instantiate the mode's screen by class identifier, load it, bind it to this
mode and load the message menu definitions. `SetGameUI` and `OnConnected` both re-resolve
the cached screen handle. Nothing is created on a dedicated server.

**Notes** — the handle is re-resolved in three places because the screen can be recreated
underneath the mode (a level change, a reconnection). Every path that could have replaced it
re-binds. A rebuild that keeps one owner for the screen needs one of these, not three.

## `IsPlayerInTeam` / `ModifyTeam` / `GetTeamCount`

**Contract** — membership test, the one-based-to-zero-based conversion, and the constant two.
The membership test handles spectators specially: a spectator is in the spectator team and
in no other.

**Notes** — `ModifyTeam` passing -1 through unchanged is what makes "no team" survive the
conversion. Every indexed access into the team style array goes through it.
