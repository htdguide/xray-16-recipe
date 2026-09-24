# src/xrGame/game_cl_mp.cpp

> The multiplayer client: input routing by match phase, the kill feed, the money-bonus display, voting, the spectator policy, and the anti-cheat that pulls a suspect's configuration files across the wire and diffs them.

**Needs** — [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Spectator.h`](Spectator.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`ExplosiveItem.h`](ExplosiveItem.h.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`UIGameMP.h`](UIGameMP.h.md) · [`ui/KillMessageStruct.h`](ui/KillMessageStruct.h.md) · [`ui/UIMessagesWindow.h`](ui/UIMessagesWindow.h.md) · [`ui/UIVotingCategory.h`](ui/UIVotingCategory.h.md) · [`ui/UIMPAdminMenu.h`](ui/UIMPAdminMenu.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`xrCore/Compression/ppmd_compressor.h`](../xrCore/Compression/ppmd_compressor.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`game_cl_mp.h`](game_cl_mp.h.md)
**Tier floor** — T2: decodes a message stream and drives screens; one raw compression buffer

## Purpose

The client half of every multiplayer mode. Three things in it are worth reading as designs
rather than as feature lists: **input is gated by match phase**, so the game is playable only
while it is running and a fixed handful of keys always work; **the kill feed is composed
rather than templated**, assembling a victim, a killer, a weapon icon and a decoration from
whatever the death message carried; and **the anti-cheat is a file transfer**, which is why
half the file is about compression, receive channels and diffs.

## State

Declared in [`game_cl_mp.h`](game_cl_mp.h.md). The fields this file establishes:

```text
  just_restarted      : bool   # true from entering the pending phase until the ready cue plays
  spectator_modes     : int (8-bit)   # a bit per camera, from the server
  current_menu        : int    # which quick-speech menu is open, or none
  ready_to_open_buy   : bool   # the server has not yet told us the buy screen is closed
  voting_active       : bool
  compress_buffer     : bytes  # grown, never shrunk
  receive_channels    : Channel[32]
  detected_cheaters   : list<(file, difference, detected_at)>
```

**Invariants** — the spectator permission mask arrives inside the **full snapshot only**, as
one byte appended after the base class's fields. Adding a field to the snapshot therefore
means changing two decoders that do not know about each other; this is the one place a
derived mode extends the base's wire format, and the pattern repeats in every mode below.

## `OnKeyboardPress`

**Contract** — the input gate. Four layers, in order.

```text
FUNCTION on_key(key)
  IF the base swallows it (no local player, or our record is marked skip) THEN swallow

  IF the key is jump or fire THEN
    decide whether it means "I am ready" and, if so, send it and swallow
    otherwise pass it through

  IF the phase is not in-progress THEN
    swallow everything EXCEPT quit, console, chat, the admin menu and the four vote keys

  IF the phase is in-progress or pending THEN handle:
    chat / team chat  -> open the chat box with the right prefix and scope
    admin menu        -> open it, or the login box if we lack rights
    begin a vote      -> if voting is enabled and none is running
    vote              -> if voting is enabled and one is running
    yes / no          -> if voting is enabled and one is running
    speech menu 0 / 1 -> toggle that quick-speech menu, closing any other

  close any open speech menu and pass the key on
```

**Invariants** — the "always allowed" set is the decision: a player in the interlude between
rounds can still quit, talk, vote and open the console, and can do nothing else. Everything
about the match is frozen; everything about being a person on a server is not.

The ready-up condition differs for an actor and a spectator, and both are worth stating:

- An **actor** declares ready with the fire key, but only while the match is pending or while
  he is permanently dead.
- A **spectator** declares ready with the fire key while pending, *or* with the jump key while
  the match is in progress and he has been dead for more than one second — which is how a
  spectator asks to join a match already running.

The one-second death delay stops the key press that killed you from immediately respawning
you.

Failing to send ready returns "not swallowed", so the key still reaches the actor. Sending it
swallows.

## `TranslateGameMessage`

**Contract** — the multiplayer message dispatch, extending the base's three membership
messages with fourteen more: the kill announcement, the four vote messages, a name change,
a quick-speech line, a money change, the game-menu response, round start and end, two
server-to-client text messages (one shown in the log, one terminating the session with a
dialog), the anti-cheat request/response exchange, the server logo transfer, the buy-screen
close, and the player-info reply.

**Invariants** — the session-terminating message goes to the *main menu*, not the in-game log,
because it is what a client sees when the server refuses or drops it. The other text message
goes to the log in red.

## `OnPlayerKilled` — the kill feed

**Contract** — decodes a death and composes one feed entry. The message carries the kill type,
the victim, the killer, the weapon and the special kind.

```text
FUNCTION on_killed(packet)
  kill_type, victim_id, killer_id, weapon_id, special = packet
  victim_record = the player owning victim_id; RETURN IF none
  killer_record = the player owning killer_id   # may be none: an anomaly has no player

  entry.victim = victim's name, coloured by his team
  entry.killer = none, white

  IF kill_type is a hit THEN
    IF the weapon resolves to an inventory item THEN
      IF it is explosive THEN use the explosion icon and say "by explosion"
      ELSE use the item's own kill-feed icon and say "from <item>"
    ELSE IF it resolves to an anomaly THEN use the explosion icon and say "by anomaly"

    IF the killer is an anomaly rather than a player THEN
      use the explosion icon and STOP — an anomaly has no name to print
    IF the killer is a player THEN set his name and team colour

    decorate by the special kind:
      headshot / eye shot / backstab -> look the decoration up in the bonus table
                                        and add the matching phrase
      none                           -> if we are the killer and it was a knife,
                                        play the butcher announcement
    headshot plays its own announcement; eye shot and backstab share the assassin one

    IF victim == killer THEN this was a suicide: clear the victim field and use the
      suicide icon, so the entry reads as one person rather than two

  ELSE IF kill_type is bleeding THEN
    use the blood-loss icon; attribute to the anomaly or to the last player who hit

  ELSE IF kill_type is radiation THEN
    use the radiation icon; nobody is attributed

  append the entry to the message window
```

**Invariants**

- **An announcement plays only for the killer**, tested by whether the killer is the entity
  the local camera is attached to. Everyone sees the feed entry; only the killer hears the
  voice.
- The suicide case clears the *victim* and keeps the killer, because for a suicide they are
  the same person and the feed shows one name.
- Bleeding and radiation deaths still name a killer where one exists: a player who wounded
  you and let you bleed out is credited. Radiation never is.

**Notes** — the icon rectangles for the explosion, suicide, blood-loss and radiation
decorations are **hard-coded pixel coordinates** into a shared atlas, repeated at six call
sites. They are a contract with the shipped user-interface texture and must be reproduced
exactly for the feed to show the right glyph. A rebuild should name them once.

Four of the five icon accessors were collapsed into one: the kill-feed, radiation and
blood-loss accessors all now return the *equipment* material, with their original
atlas-specific loads commented out. So every decoration is indexed into one atlas, and the
coordinates above are coordinates into that one. The named constants for the three original
atlases remain in the file, unused.

## `OnEventMoneyChanged` — the bonus display

**Contract** — the server sends the new balance, the delta, and a list of bonus lines. Each
line has an amount and a reason; a streak bonus additionally carries the streak length.

**Invariants** — the bonus's **icon** is looked up in the client's bonus table by a name
derived from the reason: the special-kill kinds map to fixed names, and a streak maps to a
name built from its length. The rank bonus is the exception — it selects among *ten*
rectangles by `rank * 2 + team`, so each of five ranks has a per-team icon, and the team is
asserted to be one of the two playing teams.

The streak bonus displays the streak length as a second field of the entry, which is why it
reuses the kill-feed entry record: one field for the money, one for the count.

## `LoadBonuses`

**Contract** — reads the bonus table from configuration. Each line is a reason name mapped to
an amount and a display name. Icons come from a parallel section, keyed by the reason name
with five suffixes (shader, x, y, width, height); every streak length shares one icon key, and
the rank bonus instead pulls ten named rectangles out of the user-interface texture registry,
five per team.

**Invariants** — the rectangles from the texture registry are converted from
corner-to-corner to corner-plus-size on load, because the bonus record stores extents while
the registry stores corners. The other path stores extents directly. Both are read back as
extents at display time.

## `ChatSay` / `OnChatMessage`

**Contract** — a chat line carries a scope (a team number, or all), the speaker's name, the
text, and a colour index. A team-scoped line reaches only that team; the server enforces it.

**Invariants** — **two different team encodings travel in one message**: the scope field
carries the raw team or minus one for all, while the colour field carries the mode-remapped
team plus one. The receiver reads only the colour field and clamps it into range. That is
redundant and inconsistent; a rebuild sends one team and derives both.

## the voting flow

**Contract** — four send calls (start, vote, yes, no) each guarded by whether voting is enabled
and whether one is running, and four inbound handlers that set or clear the active flag and
close the response window. A vote's subject is a free-text command string.

**Invariants** — the active flag is *client-side belief*, set by the server's start message
and cleared by its stop or end. Nothing reconciles it, so a client that missed the stop keeps
believing a vote is running until the next one starts.

Each player's vote is announced as it lands, driven by the tri-state vote field changing in a
snapshot (see [`game_cl_base.cpp`](game_cl_base.cpp.md)) — the "not yet voted" value is
skipped, so the reset at the start of a vote is silent.

## `OnSwitchPhase`

**Contract** — phase transitions drive the screen. Entering the in-progress phase shows the
indicators and leaves pending mode; entering pending sets the restart flag and, if coming
from in-progress, shows the indicators in pending mode. Every scoring and elimination phase
closes the quick-speech menus. Anything else hides the indicators.

**Invariants** — the pending case deliberately falls through into the scoring cases so that
the menus close there too. The fallthrough is load-bearing.

## `shedule_Update`

**Contract** — per tick: prune finished announcements, then by phase — play the ready cue once
on entering pending and the view exists; hide the speech menus while the local player is dead
or absent; otherwise close the chat box. Then refresh the map markers, and outside the
in-progress and pending phases close the three voting windows.

**Invariants** — the ready cue waits for a *view entity* to exist, not just for the phase, so
it is not played into a still-loading level. The restart flag is what makes it play once.

## the spectator policy

**Contract** — a byte of permission bits, one per camera mode, arrives with each snapshot. A
camera is permitted if its bit is set; during demo playback every camera is permitted.

**Notes** — the team camera's bit is the one named for the *count* of camera modes, one past
the last real camera — so the mask's width is one more than the number of cameras and the
team camera occupies the slot after them. An explicit per-camera switch is commented out
beside the bit test; the bit test is what ships.

## the anti-cheat and file transfer

**Contract** — the server can ask a client for a screenshot or for its configuration files.
The client compresses and uploads; another client (the administrator) downloads, decompresses,
saves and, for configuration, **diffs it against its own** and flags a suspect.

```text
FUNCTION on_data_request(kind)
  IF a configuration dump is asked for THEN
    dump the configuration asynchronously; on completion, upload it
  IF a response is announced THEN
    claim a free receive channel and begin receiving from the named client

FUNCTION on_receive_complete(channel)
  grow the shared decompression buffer to the announced original size
  decompress
  warn if the decompressed size differs from the announced one
  save under the screenshots root, with the extension the kind implies
  IF it is a configuration THEN
    verify it against ours; on a difference, record a suspect with the difference text
```

**Invariants**

- The **announced original size** governs how much is written out, while the decompressor
  reports how much it actually produced. They are compared and a mismatch is only warned
  about — the announced size is still what gets written, so a truncated transfer writes
  uninitialised buffer tail. A rebuild should write the produced size and treat the mismatch
  as an error.
- The compression buffer is **grown to twice the needed size and never shrunk**, and is shared
  by every channel. Two concurrent transfers completing at once corrupt each other. Nothing
  prevents it.
- The screenshot path is decompressed on the calling thread; the configuration path uses the
  multi-threaded decompressor with a yield callback that is left **default-constructed** — so
  the decompressor's cooperative yield does nothing. That is probably unintended.
- The suspect list self-prunes: an entry is shown for ten seconds and then dropped, in the
  same pass that draws it. The drawing pass is therefore also the expiry pass, and a build
  that does not draw never expires anything.

**Notes** — received file names are sanitized by replacing a fixed set of path and wildcard
characters — including the dot — with underscores, and prefixed with a timestamp. Replacing
the dot is what stops a transferred name from carrying its own extension, since the extension
is chosen by the receiver from the transfer kind.

The screenshot half of the request path is explicitly removed with a note that it should be
reimplemented, so a screenshot request arrives and does nothing.

The download progress overlay is drawn by a function whose gate is a console flag, and the
gate function itself is empty — so the overlay is dead in the shipped build. What it drew is
worth keeping: a percentage per active channel and a red line per suspect.

## `extract_server_info`

**Contract** — the server's identity bundle arrives as one transfer holding two buffers: a
logo image and a rules text. Split, and hand each to the multiplayer screen. A failed or
aborted transfer clears the logo rather than leaving the previous one.

## `RequestPlayersInfo` / `ProcessPlayersInfoReply`

**Contract** — an asynchronous request for every player's address and account digest, used by
the administration screen. One request may be outstanding; a second is refused. The reply
streams (client identifier, address, digest) triples until the packet is exhausted, filling
the matching player records, then fires the stored callback with the count and clears it.

**Invariants** — the "one outstanding request" rule is enforced by the callback slot being
non-empty, and the slot is cleared *before* the callback is invoked so that the callback may
issue another request.

A triple naming an unknown client still has its two strings read and discarded — the same
must-step-over-it discipline as the player snapshot.

## the remaining hooks

- **`OnRankChanged`** — prints the new rank's localized name. Fired by the base's snapshot
  handling when the rank field changes.
- **`net_import_state` / `net_import_update`** — sample the local rank and team before the
  base decodes, and fire the change hooks afterwards. The snapshot form additionally reads the
  spectator mask.
- **`OnGameRoundStarted`** — announces the match start, plays the cue, refreshes team and money
  displays if the local player is in the table, reports started, and re-arms the buy screen.
- **`SendPlayerStarted`** — reports the map name the client believes it is on. The server uses
  it to detect a client on the wrong map.
- **`OnGameMenuRespond`** — routes the server's answer to a menu request to one of three
  per-mode hooks (spectator, team, skin).
- **`OnWarnMessage`** — the server's high-ping warning: a count out of a total and the measured
  ping, shown as a temporary overlay. The overlay's identifier is built from the warning count,
  so successive warnings address different screen elements.
- **`OnRadminMessage`** — remote administration: an authentication result that opens the admin
  screen on the exact reply text "Access permitted." and otherwise shows the server's message,
  and a plain command echo. Matching on an untranslated English sentence is fragile; a rebuild
  sends a status code.
- **`LoadTeamData`** — reads one team's marker geometry and two materials from its
  configuration section. A missing section yields a team with no marker rather than an error.
- **`OnPlayerChangeName`** — renames the object, prints the change, and — when it is the local
  player — writes the new name into the persistent user settings.
