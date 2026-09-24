# src/xrSound/xr_cda.cpp

> Dead code: CD audio playback through the operating system's media control strings. Not compiled.

**Needs** — [`xr_cda.h`](xr_cda.h.md)
**Used by** — reached through its declarations in [`xr_cda.h`](xr_cda.h.md); callers name that, not this file.
**Tier floor** — T4 in spirit: it drives a subsystem by sending it text commands.

## Purpose

Played music from the game disc's audio tracks. Excluded from the build, unreachable on every
supported platform, and meaningless for an installation that has no optical drive. Recorded only
because the mirror must be complete.

**A rebuild should not implement this file.**

## What it decided

Two things survive as ideas, neither of them worth much:

- **A watchdog instead of an end-of-track notification.** The subsystem gave no completion callback,
  so the player computed the track's length at selection time, counted it down against the frame's
  elapsed time, and when the countdown expired asked the subsystem whether it had stopped — looping
  if it had, and giving up if it had not. Two seconds were added to the measured length as slack.
  A rebuild driving any device without completion events needs the same shape.
- **Pause is a distinct state from stop**, because resuming from pause used a different command than
  starting a track, and getting it wrong restarted the track.

Everything else — opening the drive, setting a time format, formatting track ranges into command
strings, parsing minutes-seconds-frames out of a reply — is the interface of a subsystem that no
longer exists.
