# src/xrGame/level_sounds.cpp

> The level's ambience: authored point sources that loop or chirp on a schedule, and a music playlist that picks a track appropriate to the hour.

**Needs** — [`level_sounds.h`](level_sounds.h.md) · [`Level.h`](Level.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`level_sounds.h`](level_sounds.h.md)
**Tier floor** — T3: schedules sound playback against two clocks

## Purpose

Two unrelated kinds of ambience, kept in one file because both are owned by the level and
both are driven by the same per-frame call. *Static sounds* are authored into the level's
own data — a generator hum, a dripping pipe, a crow — each pinned to a point and each with
its own idea of when it should be audible. *Music tracks* are a playlist declared in
configuration per level, one of which plays at a time, chosen by the in-game hour.

## State

```text
RECORD StaticSound
  source      : sound handle
  position    : (real, real, real)
  volume      : real
  frequency   : real            # pitch multiplier
  active_from, active_to : int  # GAME day time, milliseconds; both zero = always
  play_min, play_max     : int  # burst length range, milliseconds; both zero = whole file
  pause_min, pause_max   : int  # gap range, milliseconds; both zero = continuous loop
  next_time   : int             # ENGINE clock: when the next burst may start
  stop_time   : int             # ENGINE clock: when the current burst must end

RECORD MusicTrack
  stereo, left, right : sound handles   # exactly one arrangement is populated
  active_from, active_to : int  # GAME day time, milliseconds
  pause_min, pause_max   : int  # gap after this track, milliseconds
  volume      : real

RECORD LevelSoundManager
  statics      : list<StaticSound>
  tracks       : list<MusicTrack>
  current      : optional<int>  # index into tracks
  next_track_time : int         # ENGINE clock
```

**Invariant** — two different clocks are in play and they are not interchangeable. The
*active window* of a sound or a track is compared against **game day time** — milliseconds
since in-game midnight, which runs at the game's time factor and wraps daily. Every burst
and gap deadline is against the **engine clock** — real milliseconds since start. A rebuild
that unifies them makes night ambience speed up whenever time is accelerated.

**Invariant** — an all-zero range always means "not configured" and selects the degenerate
behaviour, never a zero-length one: no active window means always audible, no play window
means play the whole file, no pause window means loop continuously.

## `SStaticSound::Load`

**Contract** — reads one authored source out of a chunk of the level's static-sound file:
the sound's path, then position, volume, pitch, and the three time ranges as pairs. The first
chunk must be present; a malformed file is a hard failure. Creates the sound handle as an
*effect* with a source-type attenuation profile, so it is positional and attenuates with
distance like any world sound. Both deadlines start at zero, which makes the source eligible
to begin on the first update.

## `SStaticSound::Update`

**Contract** — advances one source, called every frame with the current game day time and
the engine clock. Starts, stops, and re-schedules; never blocks. The structure is a small
state machine over "am I in my active window" and "am I currently sounding".

```text
FUNCTION update(game_time, engine_time)
  in_window = (no active window) OR (active_from <= game_time < active_to)

  IF NOT in_window THEN
    IF sounding THEN stop when convenient
    RETURN

  IF sounding THEN
    IF engine_time >= stop_time THEN stop when convenient
    RETURN

  # not sounding, and allowed to
  vol = volume * occlusion at my position       # sampled fresh on every start
  IF no pause window THEN
    start looping ; stop_time = never
  ELSE IF engine_time >= next_time THEN
    IF no play window THEN
      start once, unlooped
      stop_time = never
      next_time = engine_time + file length + random gap in [pause_min, pause_max)
    ELSE
      start looping
      stop_time = engine_time + random burst in [play_min, play_max)
      next_time = stop_time + random gap in [pause_min, pause_max)
```

**Invariants** — occlusion is sampled **once per start**, not per frame, and multiplies the
authored volume. A source behind a wall starts quiet and stays at that volume for the whole
burst; walking around the wall does not change it until it restarts. That is a deliberate
economy — the occlusion query is a set of rays — and it is why intermittent sources sound
more responsive to the player's position than continuous ones.

A source with a play window loops and is *cut* at the burst deadline; a source without one
plays the file through. The difference matters for authoring: a looping ambience wants a
burst window, a one-shot bark does not.

**Notes**

- The "no pause window" branch leaves `next_time` untouched, which is correct because the
  source now loops forever until it leaves its active window.
- The current-burst deadline is compared against the engine clock read from the device
  rather than the one passed in. They are the same value; a rebuild passes one.
- Stops are *deferred* — requested, then performed by the audio system at a point where
  cutting does not click. Not an optimization: stopping a sound mid-waveform is audible.

## `SMusicTrack::Load`

**Contract** — creates a track from a file name and a five-field parameter string. Tries a
single stereo file first; failing that, falls back to a *pair* of files with `_l` and `_r`
suffixes played as two positioned mono sources. The fallback exists because the shipped
games' music is partly authored as split channels, and playing them as two positioned
sources is what reproduces the original's width.

The parameter string is five comma-separated fields in fixed order: active hour from, active
hour to, volume, pause seconds minimum, pause seconds maximum. **Hours are converted to
milliseconds and seconds to milliseconds at load**, so nothing downstream converts again.
An equal pause pair is widened by one so that the later random draw over a half-open range
is non-degenerate — an incidental fix for the range convention, not a design decision.

## `SMusicTrack::in`

**Contract** — whether a track is eligible at a given game day time. Handles a window that
**crosses midnight** — an end earlier than its start means the window wraps — which is the
only reason this is not an inline comparison. A night-time track is authored as, say, hour
22 to hour 5, and would otherwise never be eligible.

**Notes** — a window starting at hour zero with any non-zero end is treated as *always
eligible*, not as the window it declares. That is almost certainly a mistake — the test was
meant to detect "no window configured", which is both fields zero — and it means a track
authored for midnight-to-6am plays at any hour. It is the shipped behaviour and mod music
playlists are tuned around it.

## `SMusicTrack::Play` · `Stop` · `IsPlaying` · `SetVolume`

**Contract** — start, stop, test and set the volume of whichever source arrangement the
track has; the unpopulated handles are inert, so all four act on the whole track without
branching.

Playback is **two-dimensional** — head-relative rather than world-positioned — and
**ignores the game's time factor**, so accelerating time does not pitch-shift the music. The
mono pair is placed slightly left and right and slightly in front of the listener, which is
what turns two mono files back into a stereo image.

A track counts as playing when the stereo source is sounding, or when **both** mono sources
are. Requiring both is what makes a track whose halves have drifted out of sync end when the
shorter one does, rather than leaving one channel playing alone.

Volume is always the requested level multiplied by the track's own authored volume, so the
playlist can fade without losing per-track balance.

## `CLevelSoundManager::Load`

**Contract** — called once per level load. Reads every static source from the level's
static-sound file, if the level has one — the file is a flat sequence of chunks, one per
source, and its absence is normal. Then reads the music playlist: the game configuration may
declare a section per level, and that section may name a *further* section whose every line
is a track — file name on the left, the five parameters on the right. The two levels of
indirection let several levels share one playlist.

**Invariant** — the collections must be empty on entry; loading twice without unloading
would double the ambience.

## `CLevelSoundManager::Unload` · construction

**Contract** — drops both collections, releasing every sound handle. Construction starts with
no current track and an immediately-expired selection deadline, so that music begins as soon
as the first update runs.

## `CLevelSoundManager::Update`

**Contract** — the per-frame drive. Does nothing while the game is paused or while the level
is still precaching — a sound started during precache would play over the loading screen.
Updates every static source, then advances the playlist.

```text
FUNCTION update()
  IF paused OR still precaching THEN RETURN
  game_time   = current game day time
  engine_time = engine clock

  FOR EACH source IN statics : source.update(game_time, engine_time)

  IF tracks is empty THEN RETURN

  IF nothing is current AND engine_time > next_track_time THEN
    eligible = []
    FOR EACH track IN tracks
      IF track is playing THEN track.stop()      # belt and braces
      IF track.in(game_time) THEN eligible += track
    IF eligible is non-empty THEN
      current = a UNIFORM RANDOM choice from eligible ; current.play()
    ELSE
      next_track_time = engine_time + 10 seconds  # re-check later

  IF something is current AND it has stopped THEN
    next_track_time = engine_time + (its own random pause, if it declares one)
    current = none
```

**Invariants** — selection is a uniform random draw over the eligible set with no memory, so
the same track can repeat immediately. That is the shipped behaviour; a rebuild adding a
shuffle changes the feel of every level.

The gap after a track is that **track's** pause range, not a global one — a heavy piece can
declare a long silence after itself. The ten-second retry when nothing is eligible is a
different constant serving a different purpose: it is how often the manager re-asks whether
the hour has moved into some track's window. Neither has a derivation beyond taste.

**Notes** — the stop-everything sweep inside the selection loop is defensive; nothing should
be playing when there is no current track. It is cheap and it papers over a track that ended
without being noticed.
