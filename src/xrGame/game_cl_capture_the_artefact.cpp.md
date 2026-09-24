# src/xrGame/game_cl_capture_the_artefact.cpp

> Capture the artefact on the client: two artefacts, two bases, who holds which, the money and buy cycle, warm-up, voting and the time limit.

**Needs** — [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`UIGameCTA.h`](UIGameCTA.h.md) · [`game_cl_capture_the_artefact_captions_manager.h`](game_cl_capture_the_artefact_captions_manager.h.md) · [`game_cl_capturetheartefact_snd_msg.h`](game_cl_capturetheartefact_snd_msg.h.md) · [`game_cl_teamdeathmatch_snd_messages.h`](game_cl_teamdeathmatch_snd_messages.h.md) · [`game_cl_artefacthunt_snd_msg.h`](game_cl_artefacthunt_snd_msg.h.md) · [`game_cl_deathmatch_snd_messages.h`](game_cl_deathmatch_snd_messages.h.md) · [`game_base_menu_events.h`](game_base_menu_events.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`Actor.h`](Actor.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`Artefact.h`](Artefact.h.md) · [`Weapon.h`](Weapon.h.md) · [`Level.h`](Level.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`ui/TeamInfo.h`](ui/TeamInfo.h.md) · [`ui/UIVote.h`](ui/UIVote.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`ui/UISkinSelector.h`](ui/UISkinSelector.h.md) · [`ui/UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md) · [`ui/UIDemoPlayControl.h`](ui/UIDemoPlayControl.h.md) · [`xrServerEntities/clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)

**Used by** — reached through its declarations in [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md); callers name that, not this file.
**Tier floor** — T2: snapshot decoding, timed screen state and map bookkeeping

## Purpose

Two teams, two artefacts, two bases. Each team's artefact sits at their base; a team scores
by carrying the *enemy's* artefact home, and defends by returning their own when it has been
taken. That symmetry is the whole mode and it is why this is a sibling of team deathmatch
rather than a descendant — it derives straight from the multiplayer base and re-implements
team selection, skins, buying and readiness in its own terms.

Four things beyond the objective are this mode's own:

- **A money and buy cycle tied to death.** A dead player shops for his next life; the client
  tracks a provisional balance because the server has not charged him yet.
- **Warm-up**, during which nothing costs anything and no rewards count.
- **A time limit** and a **countdown**, both published by the server.
- **Voting**, with the vote text assembled and localised on the client.

## State

```text
RECORD CaptureTheArtefactClient EXTENDS MultiplayerClient
  game_ui                : CaptureTheArtefactUI
  captions_manager       : CaptionsManager           # owns every on-screen prompt

  green_artefact         : int (16-bit)              # the two artefact entities
  blue_artefact          : int (16-bit)
  green_artefact_owner   : int (16-bit)              # who holds each; 0 = at home or loose
  blue_artefact_owner    : int (16-bit)
  green_team_rpoint      : point                     # each team's home point
  blue_team_rpoint       : point
  base_radius            : real

  max_score              : int
  green_team_score       : int
  blue_team_score        : int

  team_selected          : bool                      # the pre-round gates
  skin_selected          : bool
  read_map_desc          : bool                      # the briefing has been shown once
  winner_team_showed     : bool                      # the end-of-round announcement fired once

  in_warmup              : bool
  current_warmup_time    : int (ms)
  time_limit             : int (ms)                  # 0 = none
  cur_reinforcement_time : int (ms)                  # the wave countdown, from every update
  max_reinforcement_time : int (ms)

  spawn_cost             : int (money, negative)     # default -10000
  buy_amount             : int (money)               # provisional, not yet charged
  total_money            : int                       # what the indicator shows
  last_money             : int                       # what it showed last, to avoid redraw

  player_on_base         : bool                      # edge-detected from the player flags
  allow_buy              : bool
  friendly_indicators    : bool                      # server policy, per snapshot
  friendly_names         : bool
  bearer_cannot_sprint   : bool
  can_activate_artefact  : bool
  show_players_names     : bool                      # local preference
  have_got_update        : bool                      # no accessor may be used before this
  vote_end_time          : int (ms)                  # absolute server time
```

Invariants:

- **Nothing about the objective may be read before the first snapshot.** Every accessor for
  the artefacts, the home points and the base radius asserts `have_got_update` first,
  because the objects that ask (the artefact entities themselves — see
  [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md)) can spawn before the mode has been
  told anything. Returning a plausible zero would place an artefact at the world origin.
- **`buy_amount` is a client-side promise, not a balance.** It is what the player has agreed
  to spend but has not yet been charged for, added to his real money only for display and
  for affordability tests. It is cleared when he actually spawns.
- **Team numbering here is *not* offset.** Unlike team deathmatch, the conversion method is
  the identity, and the team enumeration's values are used directly as array indices. A
  reader moving between the two modes must re-check this every time; the source comment
  calls it out as a problem.

## `net_import_state`

**Contract** — decodes the mode's snapshot: the two artefacts, the two home points, the score
triple, four policy flags, the base radius, the warm-up flag and the time limit. Detects a
score change across the decode and announces the new standing. Marks the mode ready to be
queried.

```text
FUNCTION net_import_state(packet)
  base.net_import_state(packet)
  green_artefact = packet.read_int16() ; blue_artefact = packet.read_int16()
  green_team_rpoint = packet.read_vector() ; blue_team_rpoint = packet.read_vector()

  old_total = green_team_score + blue_team_score
  max_score = packet.read_int32()
  green_team_score = packet.read_int32() ; blue_team_score = packet.read_int32()
  IF (green + blue) != old_total AND (green + blue) != 0
    announce the new standing                     # leads, or level

  show the score on screen

  friendly_indicators   = packet.read_byte() != 0
  friendly_names        = packet.read_byte() != 0
  bearer_cannot_sprint  = packet.read_byte() != 0
  can_activate_artefact = packet.read_byte() != 0
  base_radius           = packet.read_real()
  in_warmup             = packet.read_byte() != 0
  time_limit            = packet.read_int16() * 60000      # minutes on the wire

  have_got_update = true
  update_map_locations()
```

**Invariants** — the time limit travels as **minutes in a signed sixteen-bit field** and is
converted to milliseconds here. That narrow field is a frozen wire decision and the source
flags it; a rebuild must send minutes, not milliseconds, or the limit overflows above about
nine hours.

**Notes** — the score-change test sums the two scores, so it fires on any score anywhere, but
it cannot distinguish "green scored" from "blue scored" — the announcement is re-derived
from the resulting standing instead. The extra "and the total is not zero" clause suppresses
the announcement at round start, where the scores are reset to zero and the change is an
artefact of the reset.

## `net_import_update`

**Contract** — the frequent, small update: the reinforcement countdown pair and the warm-up
countdown. These change every update where the snapshot does not, which is why they are in a
separate message.

## `TranslateGameMessage`

**Contract** — the three objective events. Each names the *artefact's owning team*, which is
what makes "taken" and "returned" distinguishable: a player touching his own team's artefact
has **returned** it, and a player touching the enemy's has **captured** it.

```text
FUNCTION translate_game_message(event, packet)
  SWITCH event
    ARTEFACT_TAKEN:
      read owning_team, client
      ps = the player record for that client
      IF ps.team == owning_team
        feed "<player> returned the artefact" ; announce returned
      ELSE
        feed "<player> captured the artefact" (or "you captured it")
        announce captured
        record ps as the owner of that team's artefact

    ARTEFACT_DROPPED:
      read owning_team, client
      remove the map marker for that artefact's previous owner
      clear that team's owner
      feed "<player> dropped the artefact", or "the artefact was dropped"
          when the player has since disconnected

    ARTEFACT_ONBASE:
      read delivering_team, deliverer
      feed "the artefact is on our base" or "on the enemy base", by my team
      announce delivered
  update_map_locations()
```

**Invariants** — the owner field is set from the *take* message and cleared from the *drop*
message, not from the snapshot. The snapshot carries the artefacts but not their bearers, so
the client's picture of who is carrying what is built entirely from these events. Missing
one leaves a permanently wrong marker, which is why the map is refreshed after every message
of any kind.

**Notes** — the drop case tolerates the player having disconnected between dropping and the
message arriving, and falls back to an impersonal message. That is a real race in a game
where dropping usually means dying.

Note the asymmetry in the identifiers: take and drop carry a *connection* identifier and
delivery carries a *game* identifier. Both name a player and the two are looked up
differently. A rebuild should pick one.

## The announcement selectors

**Contract** — three routines — captured, returned, delivered — each choosing one of six
recordings: the listener's team (green or blue) crossed with the actor's relationship to the
listener (me, my teammate, an enemy).

```text
FUNCTION play_captured(capturer)
  IF no local player or no capturer: RETURN
  base = (my team is green) ? green_take_ids : blue_take_ids
  IF capturer is me:                announce base.by_me
  ELSE IF capturer.team == my team: announce base.by_teammate
  ELSE                              announce base.by_enemy
```

**Notes** — everything is spoken from the *listener's* point of view and in the listener's
team's voice, which is why the actor's team is only used to decide friend-or-foe. The same
event therefore produces six different sounds across the players in a match.

## `shedule_Update`

**Contract** — the per-update screen pass: the briefing on first entry, the money indicator,
the paid-spawn affordability prompt, the reinforcement display, and the three countdowns.
Ends by letting the captions manager flush everything it has been told.

```text
FUNCTION scheduled_update(delta)
  base.scheduled_update(delta)
  IF dedicated: RETURN
  IF a demo is playing: refresh rank and money for the watched player

  SWITCH phase
    IN_PROGRESS:
      IF I am playing
        IF the briefing has not been shown and I have an entity
          show it ; ask for any active vote
        update the money indicator
        IF I am permanently dead and not a spectator
          captions.can_call_buy_spawn(I can afford the spawn AND I am not already ready)
      IF warming up: show (0, 1) instead of the reinforcement pair
      ELSE           show (current, max)
      update the voting countdown, the warm-up countdown and the time limit

    PENDING:
      show the team panels ; clear the winner-shown flag

    PLAYER_SCORES:
      IF the winner has not been announced yet
        announce the higher score's team as the winner ; set the flag

  captions.flush()
```

**Notes** — the winner is decided by comparing the two scores, with a tie resolving to the
*blue* team because the comparison is a strict greater-than on green. That is a real
tie-break and it is silent; the server's own decision may differ.

The reinforcement display is fed a zero countdown against an interval of one during warm-up,
which is the "no countdown" idiom used across these modes.

The briefing flag is set from the *result* of showing it, so a briefing that cannot be shown
yet is retried next update.

## `UpdateMoneyIndicator`

**Contract** — shows the watched player's money, adding the local provisional spend for a
dead local player. Redraws only when the number changed.

```text
FUNCTION update_money_indicator()
  p = the player being watched ; IF none: RETURN
  IF p is permanently dead
    total = (p is me) ? p.money + buy_amount : p.money
  ELSE
    total = p.money
  IF total != last_shown: redraw ; last_shown = total
```

**Notes** — the provisional spend is added only for the *local* player and only while dead,
because that is the only window in which the client holds money the server has not accounted
for. A spectator watching someone else always sees the authoritative figure.

## `UpdateMapLocations`

**Contract** — rebuilds the whole map picture every time it is called: a marker on each
living teammate, and a marker on each artefact or on its bearer. Four artefact marker kinds:
the enemy's loose artefact (neutral), my team's loose artefact (a distinct "free friendly"
kind), the enemy's artefact in friendly hands, and my artefact in enemy hands.

```text
FUNCTION update_map_locations()
  IF dedicated, no local player, or I am a spectator: RETURN

  FOR EACH player
    IF same team AND not me AND not permanently dead: add a friend marker
    ELSE                                              remove every marker for him

  IF both artefacts exist (neither is held)
    remove every marker for both
    mark MY artefact as "free friendly" and the ENEMY's as neutral, by my team

  IF my artefact has an owner: remove its marker ; mark that OWNER as "enemy artefact"
  IF the enemy artefact has an owner: remove its marker ; mark that OWNER as "friendly artefact"
```

**Invariants** — "remove then add" per object, every pass, is what keeps exactly one marker
per object across every transition. The pass is idempotent and is called after every
objective message and after every snapshot.

**Notes** — the marker *follows the bearer*, not the artefact: a carried artefact is tracked
by putting a marker on the player carrying it. So a stolen artefact shows you its thief, and
your own team's carrier shows as the objective. That is the right information and it is why
the owner identifiers are tracked separately at all.

The "both artefacts exist" test is doing double duty as "neither is being carried", which
holds only because a carried artefact's entity is not independently visible. It is fragile
and a rebuild should test the owners.

## `OnPlayerFlagsChanged`

**Contract** — the client's reaction to its own state changing, edge-detected from the player
flags: entering and leaving a base enable and disable buying; becoming permanently dead
opens buying and closes the inventory and buy windows; coming back to life closes the
paid-spawn box. Also applies the invincibility flag to *any* player's actor.

```text
FUNCTION on_player_flags_changed(ps)
  base.on_player_flags_changed(ps)
  IF ps is me
    IF not on base AND flag says on base:  on_player_enter_base() ; on_base = true
    IF on base AND flag says not on base:  on_player_leave_base() ; on_base = false
    IF permanently dead
      captions.can_call_buy(true) ; hide the actor menu and the buy menu
    ELSE
      hide the paid-spawn box if it is shown
  set_invincible(ps.game_id, ps has the invincible flag)
```

**Invariants** — base entry and exit are **edge-detected against a local copy** rather than
read from the flag, because the two handlers must run once per crossing and the flag is
present in every update. The local copy is the whole reason those handlers are not called
continuously.

**Notes** — invincibility is applied by turning off the actor's ability to be harmed. It is a
short spawn protection; the server owns the timing and the client only mirrors the flag.

## `OnKeyboardPress` / `OnKeyboardRelease`

**Contract** — the mode's keys: scores shows the team panels while held, skin and team open
their menus, inventory toggles the actor menu, buy opens the buy menu. All of them require
the round in progress and the local player active. During demo playback only the scores and
crouch keys are accepted.

**Notes** — the scores key is the only hold-to-show key, which is why it needs the release
handler. Everything else is a toggle or an open.

## The gate family

**Contract** — `CanCallBuyMenu` allows buying when the menu's data is ready and either the
player is alive and on a base, or he is dead and not a spectator. `CanCallInventoryMenu`
refuses while dead. `CanCallSkinMenu` always allows. `CanCallTeamSelectMenu` requires the
round in progress and the window not already up. `CanBeReady` refuses while team or skin is
unchosen, opening the relevant menu instead.

**Notes** — the buy rule is the objective one: **a dead player may shop and a living one may
shop only at his base.** That is the mode's economy — you plan your next life while waiting
for the next wave.

The team-select gate's mutual-exclusion checks against the buy and skin menus are present
but commented out, so in this mode those windows can overlap. That is a live difference from
team deathmatch and looks like an oversight.

## `OnTeamSelect` / `OnGameMenuRespond_ChangeTeam` / `OnSkinMenu_Ok` / `OnGameMenuRespond_ChangeSkin`

**Contract** — the two pre-round choices, each a request and a reply. A team request is
suppressed when the player picks the team he already has and still holds a valid skin;
sending one invalidates the skin. The replies set the value, mark the gate satisfied, and —
for the team — rebuild everything that depends on it.

**Notes** — unlike team deathmatch, the team is sent **unmodified** (no offset) and comes back
as a byte rather than a truncated signed value. Same protocol shape, different convention,
in a sibling mode. A rebuild should unify them.

## `OnTeamChanged` / `OnGameRoundStarted` / `OnRankChanged`

**Contract** — a team change rebuilds the buy menu and the skin menu against that team's
configuration section, refreshes the rank display and re-reads the rank-based default items.
A round start replays the same rebuild. A rank change refreshes the rank display, re-reads
the default items and plays the rank announcement.

**Notes** — re-running the team-changed path at round start is how a player who did not change
teams still gets a buy menu appropriate to his current rank, since rank carries across
rounds and unlocks items.

The rank announcement is skipped at rank zero and above rank three, the latter with a source
note that no recording exists yet. That is a data gap, not a rule.

## `NeedToSendReady_Spectator` / `OnBuySpawnMenu_Ok` / `SpawnMe`

**Contract** — while dead, the spawn key opens the paid-spawn confirmation instead of
readying, unless the player is already ready, is warming up, cannot afford it, or is a
spectator. Confirming sends the buy-spawn event. `SpawnMe` sends a plain ready event and is
currently unused.

**Notes** — affordability is money plus the negative spawn cost plus the provisional buy
amount, all of which must leave a non-negative balance. So a player who has queued an
expensive loadout cannot also pay to skip the wave — the two compete for the same money,
which is the intended tension.

## `OnBuyMenu_Ok` / `OnBuyMenu_Cancel` / `OnBuyMenuOpen` / `LocalPlayerCanBuyItem`

**Contract** — the buy cycle: confirm a purchase, cancel one, announce that the menu opened,
and ask whether an item may be bought at all. Implemented in
[`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md), where
the provisional-spend rule and the purchase wire format are set out.

## `OnSpeechMessage`

**Contract** — an authored radio phrase from another player: chat text and a map marker for
teammates, and a sound whose spatialisation depends on whether the speaker is a friend.
Implemented in
[`game_cl_capture_the_artefact_messages_menu.cpp`](game_cl_capture_the_artefact_messages_menu.cpp.md).


## `OnVoteStart` / `UpdateVotingTime` / `OnVoteStop` / `OnVoteEnd`

**Contract** — a vote arrives as a raw command string, the proposing player's name and a
duration. The client splits the command into a verb and up to five arguments, translates the
verb through a fixed table of six known commands, translates each argument through the string
table, and assembles a localised sentence. The countdown display shows the time remaining and
the fraction of players who have agreed.

```text
FUNCTION on_vote_start(packet)
  read command, proposer ; vote_end_time = server_time + packet.read_int32()
  verb = first word of command
  args = up to 5 further words
  translated_verb = table lookup of verb among
      restart, restart_fast, kick, ban, changemap, changeweather
      (untranslated verb if unknown)
  sentence = translated_verb + " " + each translated argument
  show "voting started: <sentence> by <proposer>"
  open the vote response window on that sentence

FUNCTION update_voting_time(now)
  IF voting is enabled and active and the deadline has not passed
    remaining = deadline - now
    agreed = count of players who voted yes
    show the remaining minutes and seconds and agreed / total
```

**Notes** — the command is composed on the server as an English string and *taken apart again
on the client* to be localised. That is the load-bearing awkwardness: the protocol carries a
human-readable command rather than a structured vote, so the client must re-parse it. A
rebuild should send a vote kind plus typed arguments and lose the whole parsing step.

The agreement fraction is over *all* players including those who have not voted, so an
undecided match shows a low fraction rather than a split of those who answered.

An unknown verb passes through untranslated and is displayed as-is, which is the right
failure: an unrecognised vote is still shown.

## `OnRender`

**Contract** — teammate names and indicators, for the local player looking through his own
eyes, gated on the local preference and the server's permission. Names are uppercased.

**Notes** — same shape as team deathmatch, with two differences: names are forced to upper
case, and an invincibility indicator for protected teammates is present but commented out.

## `PlayerCanSprint`

**Contract** — an artefact carrier may not sprint.

**Notes** — **the rule is inverted end to end, and the fault is on the server.** This mode's
server sends the *negation* of the configured setting — "the bearer **can** sprint" — into a
client field named for the opposite sense, which this then tests as though it meant "cannot".
Turning the restriction on therefore lets the carrier sprint, and turning it off forbids it.
The client's own test is self-consistent; its field is simply misnamed, and the sibling mode
([`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md)) sends the same setting
un-negated and behaves correctly. One console setting drives both modes, so it cannot be
configured to satisfy them both. A rebuild should send the setting un-negated in both and
implement the intended rule.

## The guarded accessors

**Contract** — the artefact identifiers, the owner identifiers, the two home points and the
base radius are all exposed with a hard assertion that a snapshot has arrived.

**Notes** — the callers are the artefact entities themselves, which spawn from the level and
ask the mode where they belong. The assertion is what turns a race into a diagnosable stop
instead of an artefact silently placed at the origin. See
[`cta_game_artefact.cpp`](cta_game_artefact.cpp.md), which instead *tolerates* the
not-yet-known case by retrying every frame — the two guards disagree about whether this is an
error, and the artefact's tolerant version is the one that runs.

## `OnSpawn`

**Contract** — a spawning artefact gets a neutral map marker. A spawning actor gets a friend
marker if he is a teammate, and if he is *me*, clears the provisional spend and closes the
buy menu. A spawning weapon with an owner is recorded in the weapon usage statistics as a
purchase.

**Notes** — clearing the provisional spend on my own spawn is the other half of the money
model: the promise is discharged the moment the server acts on it.

The disabled block here would have moved each artefact to its home point on spawn, from the
client, with a source comment calling it a hack and noting that server logic belongs on the
server. The live arrangement puts that in the artefact entity instead.

## `createGameUI` / `SetGameUI` / `OnConnected` / `Init` / `getTeamSection` / `GetGameScore`

**Contract** — instantiate the mode's screen by class identifier, bind the captions manager to
it, load both teams' data and the spawn cost, and re-resolve the screen handle on connection.
The team section lookup and the compact score round it out.

**Notes** — the team section lookup here is indexed **zero-based** while the buy and skin
menus are built from a lookup indexed by the raw team value. Two conventions in one file,
which is the numbering hazard the class itself warns about.
