# src/xrGame/game_cl_capture_the_artefact_messages_menu.cpp

> An authored radio phrase from another player: text and a map marker for teammates, and a sound that an enemy hears positionally — so shouting gives your position away.

**Needs** — [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`Level.h`](Level.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`ui/UIMessagesWindow.h`](ui/UIMessagesWindow.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an indexed lookup into authored data and a positional sound

## Purpose

One method of the capture-the-artefact client mode, split out because it is the only place
the authored phrase menus are consumed. It belongs to the class declared in
[`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md).

The mode ships a set of *message menus*: a tree of authored phrases a player can send without
typing — "enemy spotted", "cover me". A sender picks a menu, a phrase and a variant; those
three small indices are all that travels, and the receiver reconstitutes text and sound from
its own copy of the authored data.

## State

`Stateless.` It reads the mode's loaded message menus and the player list.

## `OnSpeechMessage`

**Contract** — decodes one phrase message and presents it. Teammates get the translated text
in the chat and a temporary radio marker on the speaker's map position. Everyone in earshot
gets a sound, but *which* sound and *where* differ: your own phrase plays flat, a teammate's
plays flat over the radio, and an enemy's plays **positioned at the speaker in the world**.
Does nothing if the local player is dead or absent. Every index from the wire is
bounds-checked and a bad one drops the message silently.

```text
FUNCTION on_speech_message(packet)
  IF no local player OR I am permanently dead: RETURN

  speaker = player by game identifier from the packet
  IF none: RETURN

  menu_index = packet.read_byte()
  IF menu_index out of range: RETURN
  phrase_index = packet.read_byte()
  IF phrase_index out of range for that menu: RETURN
  phrase = menus[menu_index].messages[phrase_index]

  IF speaker.team == my team
    chat.add(translate(phrase.text), speaker.name)
    IF no radio marker exists for the speaker
      add one ; enable its pointer

  variant_index = packet.read_byte()
  IF the phrase has no variants: RETURN
  IF variant_index is non-zero and out of range: RETURN
  sound = phrase.variants[variant_index][speaker.team]     # the phrase in that team's voice

  IF speaker.team == my team
    IF speaker is me: play sound.voice flat
    ELSE              play sound.radio flat
  ELSE
    IF the speaker's entity exists: play sound.voice positioned at it
```

**Invariants** — three indices arrive from another client and all three index into local
arrays. Each is checked before use; that is what makes the handler safe against a peer
sending nonsense, and a rebuild must not lose the checks.

**Notes** — the sound selection has two independent dimensions and both matter.

*Which recording*: the phrase is recorded once per team, so a phrase spoken by a green player
is heard in the green voice regardless of who is listening. Indexed by the **speaker's**
team, not the listener's — which is the opposite of the objective announcements in
[`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md), and correct, since
this is a person speaking rather than a narrator.

*Where it comes from*: a teammate reaches you over the radio and is therefore not
spatialised; an enemy is heard through the air, at his actual position, attenuated by range.
So using the phrase menu near an enemy tells him where you are. That is a genuine tactical
rule and it is expressed entirely by which of two sound handles is played and whether a
position is supplied.

The radio variant exists only in the teammate case; your own phrase plays the voice
recording flat, which is what you would hear yourself say.

The map marker is added for a speaking teammate and never removed here — the marker kind is
expected to expire on its own. The idempotence check (add only if absent) is what stops a
talkative player accumulating markers.

A dead local player is deliberately deaf to phrase messages. That is consistent with the
mode's other dead-player rules, which put him in a shopping screen rather than in the world.
