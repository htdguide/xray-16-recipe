# src/xrGame/game_cl_mp_snd_messages.cpp

> The announcement mixer: one announcement plays at a time, a more important one cuts off a less important one, and equally important ones queue behind each other.

**Needs** — [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md) · [`Level.h`](Level.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md); callers name that, not this file.
**Tier floor** — T2: schedules playback against measured sound lengths

## Purpose

Announcements are the voice that says "headshot", "team one leads", "three, two, one". They
must not overlap, because two overlapping lines are unintelligible, and they must not be
simply dropped, because the ones that matter carry information. The rule this file
implements is the whole answer: **priority decides who wins, and among equals the newcomer
waits**.

## State

```text
RECORD Announcement
  sound_id      : int         # the registry key; see game_cl_mp_snd_messages.h
  priority      : int         # higher wins
  sound         : sound_handle
  last_started  : int         # server time the playback was scheduled to begin

RECORD Mixer
  registered : list<Announcement>    # every announcement this mode loaded
  in_play    : list<Announcement>    # those currently sounding or scheduled
```

**Invariants** — an announcement appears in the registry exactly once per identifier, and
the identifier is the only key: lookup is a linear scan for it. A duplicate registration
shadows the later one permanently, and nothing detects it.

An entry in the playing list may already have finished; the list is pruned lazily rather than
on completion, because the audio device reports completion only by the handle going quiet.

## `LoadSndMessage`

**Contract** — registers one announcement from configuration: a section and a key whose value
is a comma-separated pair of sound name and priority. A missing section, a missing key, or a
value with fewer than two parts is silently ignored, not an error.

**Invariants** — silence on a missing entry is deliberate: a mode declares announcements for
every event it might raise, and a localization or a mod is free to supply only some. A
missing announcement means "say nothing here", which is why the play path's failure to find
one is a much louder error than the load path's failure to register it.

**Notes** — the line buffer is four kilobytes for a value that is a name and a number. The
size is a generic string-reading habit in this codebase, not a requirement.

## `AddSoundMessage`

**Contract** — registers an announcement directly from a name, a priority and an identifier,
bypassing the configuration lookup. This is the path a script or a mode with computed
announcement names uses.

## `PlaySndMessage`

**Contract** — requests an announcement by identifier. Finds it, and either plays it now,
schedules it for later, or drops it, depending on what is already sounding.

```text
FUNCTION play(id)
  entry = the registered announcement with this id
  FAIL WITH no_such_announcement IF none      # a mode asked for something it never loaded
  IF entry is already sounding THEN RETURN    # never overlap an announcement with itself

  delay = 0
  FOR EACH other IN in_play
    IF other is no longer sounding THEN CONTINUE
    IF other.priority > entry.priority THEN RETURN          # outranked: drop this one
    IF other.priority < entry.priority THEN other.stop()    # outrank it: cut it off
    ELSE                                                     # equal rank: queue behind it
      remaining = other.last_started + other.length - now
      IF remaining > 0 THEN delay = max(delay, remaining)

  play entry as a non-positional sound, after `delay` seconds
  entry.last_started = now + delay
  in_play.append(entry)
```

**Invariants**

- The three priority branches are exhaustive and each is a different policy: **drop**,
  **preempt**, **queue**. A rebuild that collapses them loses the design.
- The delay is the *maximum* over all equally ranked announcements still sounding, not their
  sum, because they are themselves already staggered — each was scheduled against the same
  rule, so the last one to finish is the one to wait for.
- The start time is recorded as the *scheduled* start, not the moment of the call, so an
  announcement queued behind two others computes its own end correctly for a third.
- Announcements play as two-dimensional sound at the origin: they come from the announcer,
  not from anywhere in the world, so they must not be attenuated or panned.

**Notes** — a request for an unregistered identifier is a hard assertion. Given that the load
path silently accepts a missing entry, a mode with an incompletely localized announcement set
crashes at the moment it first tries to speak. The two halves disagree about how tolerant to
be; a rebuild should make the play path silent too.

## `UpdateSndMessages`

**Contract** — prunes the playing list of announcements the audio device has finished. Called
each scheduled tick.

**Invariants** — pruning is what makes the delay computation converge; without it every past
announcement would be considered and the list would grow without bound. It is also the only
place an announcement leaves the playing list — a preempted one is stopped but removed here,
one tick later.
