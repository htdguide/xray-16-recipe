# src/xrGame/DemoPLay_Control.cpp

> Demo playback that can be told "run until someone kills Ivan, then stop": it subscribes to one kind of recorded game event, optionally fast-forwards until a matching one arrives, and pauses there.

**Needs** — [`DemoPlay_Control.h`](DemoPlay_Control.h.md) · [`Level.h`](Level.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Message_Filter.h`](Message_Filter.h.md)
**Used by** — reached through its declarations in [`DemoPLay_Control.h`](DemoPLay_Control.h.md); callers name that, not this file.
**Tier floor** — T2: a subscription to a recorded message stream, plus playback rate control

## Purpose

A demo is a recorded stream of the same network messages the client originally received,
so anything the game told the client about — a kill, a round start, an artefact changing
hands — is still in the file as a message. That is the insight the whole file rests on:
seeking a demo to an interesting moment needs no index and no annotation, only a
subscription to the message kind you care about and a willingness to play forward.

Two modes are offered, and they differ only in playback rate. *Pause on* watches at normal
speed and stops when the event happens. *Rewind until* watches at eight times speed and
stops when the event happens, restoring the previous rate. Both are one-shot: matching
disarms the subscription and returns to idle.

An event may be qualified by a player-name fragment. An empty fragment matches the first
event of the kind; a non-empty one is matched as a *substring* of the relevant player's
name, so a partial name is enough.

## State

```text
ENUM Action
  on_round_start          # no player
  on_kill                 # qualified by the killer's name
  on_die                  # qualified by the victim's name
  on_artefactdelivering   # qualified by the deliverer's name
  on_artefactcapturing    # qualified by the taker's name
  on_artefactloosing      # qualified by the dropper's name

ENUM Mode
  not_active | rewinding | waiting_for_actions

RECORD DemoplayControl
  mode             : Mode
  action           : Action
  param            : text            # the name fragment; empty matches anything
  previous_speed   : real            # saved across a rewind
  user_callback    : optional<callback>   # fired once, when the event lands
```

Invariants: at most one subscription is armed at a time — both entry points refuse while
the mode is not idle. The saved playback rate is only meaningful while rewinding. Matching
always disarms: the subscription is removed, the rate restored and the mode returned to
idle before the caller's callback runs.

## `pause_on`

**Contract** — arms a watch at the current playback rate. Refuses, with a log line, if
anything is already armed. Un-pauses the device first if it was paused, because a watch
that never advances never fires. Answers nothing; the caller learns of the match through
the pause.

## `rewind_until`

**Contract** — the same, plus: saves the current playback rate, sets it to eight times
normal, and takes an optional callback to run when the event lands. Refuses if anything is
armed. Answers whether it armed.

**Notes** — the fast-forward factor is a compiled-in eight. It is a compromise between
seek time and the demo stream's own decode cost; nothing tunes it.

## `stop_rewind` / `cancel_pause_on`

**Contract** — disarm without having matched. Each refuses unless the corresponding mode is
active — `stop_rewind` silently, `cancel_pause_on` with a log line. Stopping a rewind also
restores the saved playback rate.

**Notes** — the asymmetry in how the two report a wrong-mode call is unexplained and looks
accidental.

## `activate_filer` / `deactivate_filter`

**Contract** — install or remove the message subscription for the chosen action. Each action
maps to one recorded game-event kind; an unknown action is fatal.

```text
on_round_start        -> round-started event
on_kill               -> player-killed event
on_die                -> player-killed event     # same event, different field read
on_artefactdelivering -> artefact-on-base event
on_artefactcapturing  -> artefact-taken event
on_artefactloosing    -> artefact-dropped event
```

**Invariants** — kill and die share an event kind, so they cannot be armed simultaneously —
which the one-at-a-time rule already guarantees. The distinction is entirely in which
identifier is read out of the message.

## The per-event handlers

**Contract** — each handler re-reads the message it was given, confirms it is the kind
expected, and decides whether it matches. An empty name fragment matches immediately. A
non-empty one is compared against the relevant player's name as a substring; a player
identifier that resolves to nobody is ignored and playback continues.

```text
FUNCTION handle(packet)
  confirm the packet's kind and game-event kind          # both asserted
  IF param is empty THEN match(); RETURN

  skip the event's leading fields up to the identifier this action cares about
  player = look up that identifier
  IF player is none THEN RETURN                          # unresolvable: keep playing
  IF param occurs anywhere in player.name THEN match()
```

**Invariants** — the handler must consume the message's fields *positionally* to reach the
identifier it wants, so it encodes the event's wire layout. The kill event, for instance,
carries a kill type, then the victim, then the killer — which is why the "on kill" handler
skips two fields and the "on die" handler skips one.

**Notes**

- The three artefact events are laid out differently in the two modes that use them: the
  capture-the-artefact mode prefixes a team byte and identifies the player by connection
  identifier, while the artefact-hunt mode identifies by in-game identifier and has no
  team byte. Every artefact handler therefore branches on the mode before parsing, and an
  artefact event arriving under any other mode is fatal. A rebuild should make these one
  message shape and delete the branch.
- Matching by substring rather than by equality is what makes the feature usable from a
  console: the operator types part of a name. It also means a fragment can match the wrong
  player.

## `process_action`

**Contract** — the shared match path. Restores the playback rate if rewinding, pauses the
device, removes the subscription, runs the caller's callback if there is one, and returns
to idle. The order matters: the subscription is gone and the mode is settled before the
callback runs, so a callback may immediately arm the next watch.

## Could not recover

- The callback is never cleared, so the callback supplied to one rewind remains installed
  and will fire again after a subsequent `pause_on` that was given none. A rebuild should
  clear it in `process_action`.
- Nothing calls any of this in the shipped tree; it is console or tooling machinery whose
  front end is not present.
