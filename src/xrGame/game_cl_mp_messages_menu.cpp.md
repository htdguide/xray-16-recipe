# src/xrGame/game_cl_mp_messages_menu.cpp

> The quick-speech radio: a small menu of authored phrases, each with several recorded variants per team, heard as radio by your own side and as a voice in the world by the other.

**Needs** — [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_cl_mp_messages_menu.h`](game_cl_mp_messages_menu.h.md) · [`ui/UISpeechMenu.h`](ui/UISpeechMenu.h.md) · [`ui/UIMessagesWindow.h`](ui/UIMessagesWindow.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`Level.h`](Level.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`game_cl_mp_messages_menu.h`](game_cl_mp_messages_menu.h.md)
**Tier floor** — T2: probes the filesystem at load time and mixes positional and non-positional playback

## Purpose

The multiplayer chat shortcut: a key opens a small menu of authored phrases, and picking one
broadcasts it. The interesting decisions are not in the menu but in **how the result is
heard**, which differs three ways depending on who is listening.

## State

```text
RECORD MessageSound
  voice : sound_handle     # the line as spoken aloud
  radio : sound_handle     # the same line as heard over the radio

RECORD MenuMessage
  text     : text                       # a localization key
  variants : list<list<MessageSound>>   # [variant][team]

RECORD MessageMenu
  screen   : speech_menu_widget
  messages : list<MenuMessage>          # at most ten

RECORD Client                            # the relevant part
  menus        : list<MessageMenu>      # at most ten
  current_menu : int                    # which menu is open, or none
```

**Invariants** — the variant table is indexed **variant first, team second**: one phrase has
several recordings of the same words, and each recording exists once per team because each
team has a different voice actor. A team's entry must exist for every variant, and the load
path asserts it.

Both the phrase count per menu and the menu count are capped at ten, by the loop bounds
rather than by a declared limit. The cap is arbitrary and generous; treat it as a limit to
document, not to reproduce exactly.

## `AddMessageMenu`

**Contract** — builds one menu from a configuration section. Reads phrase entries named by
index until one is missing; each entry is a comma-separated pair of a localization key and a
sound base name. For each phrase, probes the sound filesystem for numbered variants until one
is missing, and for each variant loads one voice and one radio recording per team.

```text
FUNCTION add_menu(section, sound_path, team_prefix)
  IF the section does not exist THEN RETURN      # a mode may declare menus it did not author
  menu = new menu with a widget built from the section

  FOR i IN 0..9
    IF no phrase entry i THEN BREAK              # missing means "the list ends here"
    (key, sound_base) = the entry's two parts
    message = new message with that key

    FOR variant IN 1..16
      IF the team-1 voice recording for this variant does not exist THEN BREAK
      FOR team IN 1..team_count
        FAIL WITH missing_recording UNLESS both the voice and the radio recording exist
        load both into the variant's team slot
```

**Invariants** — **existence of the first team's recording decides how many variants there
are**, and every other team must then have the same count. That is why the probe loop and
the per-team loop are separate, and why the per-team failure is fatal while the probe's is a
normal stop: a missing variant is the end of the list, a missing *team* of an existing
variant is an incomplete sound bank.

Probing the filesystem rather than declaring the count in configuration means a mod adds a
variant by dropping a file in. It also means the load cost is a filesystem probe per
candidate, which is why the cap is sixteen.

**Notes** — a phrase entry that parses to zero parts is skipped without consuming a message
slot, so the phrase indices in the menu can differ from the configuration's. Since the
selection is sent as an index into the *loaded* list, that is consistent as long as every
client loads the same configuration — which the anti-cheat configuration comparison exists to
check.

The sound path is composed as `<path><team_prefix><team>/voice_<base><variant>`, so the team
appears in the directory name and the variant in the file name. That layout is a contract
with the shipped sound bank.

## `LoadMessagesMenu`

**Contract** — reads the sound root and the optional team-directory prefix from a top-level
section, then builds each menu named by an indexed entry, stopping at the first missing
index. Clears the existing set first.

## `OnMessageSelected`

**Contract** — the local player picked a phrase. Resolves which menu the widget belongs to,
bounds-checks the phrase, picks a variant **at random**, and sends menu, phrase and variant
upstream as a game event addressed to the player's own body.

**Invariants** — the *sender* chooses the variant, not each receiver, so everyone hears the
same recording. That is what makes the line a shared event rather than a private one, and it
is why the variant index travels on the wire.

With one variant the choice is skipped rather than drawn, so a single-variant phrase costs no
randomness — and, more importantly, always sends index zero.

The menu identifier is the menu's *position in the loaded list*, narrowed to a byte. Two
clients with different configurations will disagree about what was said.

## `OnSpeechMessage`

**Contract** — somebody said something. Three listeners, three behaviours.

```text
FUNCTION on_speech(packet)
  IF we have no local player OR the local player is permanently dead THEN RETURN
  speaker = the player whose body id the packet names; RETURN IF unknown
  menu, phrase = the packet's indices; RETURN IF either is out of range
  variant = the packet's index

  IF speaker is on our team THEN
    show the phrase's localized text in the chat window, attributed to the speaker
    IF the speaker has no radio marker on the map THEN add one and enable its pointer
    IF the speaker is us THEN play the VOICE recording, non-positional
    ELSE                      play the RADIO recording, non-positional
  ELSE
    play the VOICE recording POSITIONALLY at the speaker's body
```

**Invariants** — this three-way split is the whole design.

- **Your own line** you hear as your own voice, in your head: non-positional, unprocessed.
- **A teammate's line** you hear over the radio: non-positional, and radio-processed, because
  it did not travel through the air to you. It also puts a marker on your map, which is the
  tactical value of the feature.
- **An enemy's line** you hear as a voice *in the world*, attenuated by distance: you can only
  hear it if you are close enough, and it tells you where he is. This is why the enemy branch
  is positional and why the enemy's text is not shown and no marker is added.

A permanently dead player hears nothing at all.

The recording is selected by the **speaker's** team, through the mode's team-remapping hook,
so the voice belongs to the speaker's side regardless of who is listening.

**Notes** — the variant bounds check passes index zero unconditionally even when the variant
list is shorter, which is safe only because an empty list is rejected on the line above. The
team index is not bounds-checked at all: a packet naming a team the client did not load
recordings for reads past the end. Both are consequences of trusting the server's relay of a
client's own numbers; a rebuild validates both.

The map marker is added but never removed here — the marker type's own lifetime governs how
long the radio contact stays visible.

## `DestroyMessagesMenus`

**Contract** — stops and releases every recording in every variant of every phrase of every
menu, and destroys the menu widgets.

**Notes** — it does not clear the list it just emptied, so the structures remain holding
released handles. The only caller discards the whole object immediately afterwards, which is
why it never bites.

## `HideMessageMenus`

**Contract** — closes any menu that is currently open. Called when the game takes input back.
