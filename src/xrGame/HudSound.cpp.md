# src/xrGame/HudSound.cpp

> The sound bank held items play from: one alias names a set of interchangeable takes, a collection names many aliases, and a layered collection plays several banks at once as one sound.

**Needs** — [`HudSound.h`](HudSound.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`HudSound.h`](HudSound.h.md); callers name that, not this file.
**Tier floor** — T2: configuration parsing and handle bookkeeping over the audio seam

## Purpose

A held item does not own sound handles by name; it owns a **bank keyed by alias**. That
one decision is what the whole file exists to support, and it buys three things a rebuild
should want:

- **Variants for free.** One alias resolves to a *list* of takes, numbered in the
  configuration by suffix. Playing picks one at random, or by index when the caller has
  already chosen — which is how a reload animation with three takes plays the sound that
  matches the take.
- **Per-sound gain and delay in the data.** Each take's configuration line is a tuple of
  file, volume and delay, so an artist can push one take quieter or start it a fraction
  late without touching code.
- **Exclusivity.** An alias may be marked exclusive, meaning playing *anything* from the
  bank silences it. That is how a looping bolt-action or an idle hum is cut by the next
  action.

The layered collection on top is a later addition and solves a different problem: modern
weapon audio is several simultaneous layers (mechanism, report, tail), and it makes a
group of banks answer to one alias as though it were one sound.

## State

```text
RECORD Take
  sound  : SoundHandle
  volume : real     # multiplier, default 1
  delay  : real     # seconds before it starts, default 0

RECORD SoundItem                 # one alias
  alias      : text
  takes      : list<Take>        # one or more interchangeable variants
  exclusive  : bool              # playing anything in the owning bank stops this
  active     : optional<Take>    # the take currently playing, if any

RECORD SoundBank
  alias : optional<text>         # set only when this bank is one layer of a layered bank
  items : list<SoundItem>        # looked up by alias, linearly

RECORD LayeredBank
  layers : list<SoundBank>       # several banks, each carrying the same alias
```

Invariants:

- An alias is unique within a bank; adding a duplicate is a data error caught at load.
- `active` points into the take list, so anything that reallocates the list invalidates
  it. Loading always precedes playing, and destruction clears it.
- Setting the position of a two-dimensional (non-positional) sound is meaningless, so the
  position setter *drops the active take* when it finds one — the caller stops tracking a
  sound that does not move.

## `InitHudSoundSettings`, `psHUDSoundVolume`, `psHUDStepSoundVolume`

**Contract** — two global gain multipliers, read once from one configuration section, that
scale every sound played *in first-person mode* and the player's footsteps respectively.
They exist so the player's own weapon and steps can be balanced against the world without
re-authoring the files.

## `LoadSound` — one alias

**Contract** — read a take list from configuration. The first take is the named key; each
further take is that key with an ascending integer appended, and reading stops at the
first missing one. Each take's value is a comma-separated tuple: the sound file, then an
optional volume, then an optional delay.

**Invariants** — the numbering starts at one for the *second* take, so `snd_shoot`,
`snd_shoot1`, `snd_shoot2` is a three-take alias. A missing number ends the list, so the
numbering may not have gaps.

```text
FUNCTION load_alias(section, key) -> SoundItem
  takes = empty ; n = 0 ; name = key
  WHILE section has a line named `name`
    takes.append(load_take(section, name))
    n = n + 1 ; name = key followed by n
  RETURN takes

FUNCTION load_take(section, line) -> Take
  fields = the line's value split on commas         # at least one required
  sound  = create an effect sound from fields[0]
  volume = fields[1] if present and non-empty, else 1
  delay  = fields[2] if present and non-empty, else 0
```

## `PlaySound` — one alias

**Contract** — play one take of an alias at a position, attributed to a parent object,
optionally looped, optionally in two-dimensional mode, optionally with a chosen take
index. Stops every take of this alias first. An index past the end is clamped to the last
take rather than refused; no index means random. The chosen take's volume is applied,
scaled by the first-person gain when in that mode. A two-dimensional sound is played at
the origin rather than at the supplied position.

**Invariants** — stopping the whole alias before playing is what guarantees an alias never
overlaps itself. Two *different* aliases overlap freely; that is the point of aliases.

```text
FUNCTION play(item, position, parent, hud_mode, looped, index)
  IF the item has no takes THEN RETURN
  stop every take of this item
  flags = (two-dimensional IF hud_mode) plus (looped IF looped)
  IF no index given THEN index = random over the takes
  ELSE index = min(index, last take)
  active = takes[index]
  play active at (origin IF hud_mode ELSE position), with flags, after its delay
  active.gain = active.volume * (the first-person gain IF hud_mode ELSE 1)
```

**Notes** — the index clamp is a defensive repair rather than a decision: a caller asking
for take five of a three-take alias gets take three. A rebuild might prefer to fail, but
the shipped data relies on the leniency, because an animation may have more takes than
the sound does.

## `StopSound`, `DestroySound`, `playing`, `set_position`

**Contract** — stop every take and clear the active one; release every handle and clear
the list; report whether the active take is still sounding; and move the active take,
dropping it if it turns out not to be positional.

## `HUD_SOUND_COLLECTION` — the bank

**Contract** — `LoadSound` adds an alias, asserting it is not already present and
recording whether it is exclusive. `PlaySound` **first stops every exclusive alias in the
bank**, then plays the requested one — and silently does nothing if the alias is unknown.
`StopSound` asserts the alias exists. `SetPosition` moves an alias's active take if it is
playing. `StopAllSounds` stops everything. Destruction stops and releases every alias.

**Invariants** — the asymmetry between play and stop is deliberate and worth keeping:
playing an unknown alias is tolerated, because items share animation code and not every
item defines every sound; stopping an unknown alias is a programming error, because
something asked to stop a sound it believed it had started.

The exclusivity sweep runs over the *whole bank* on every play, so exclusivity is a
property of the silenced alias, not of the playing one. Any sound cuts every exclusive
sound.

## `HUD_SOUND_COLLECTION_LAYERED`

**Contract** — several banks, each tagged with the same alias, played and stopped as one.
Every operation walks the layers and applies itself to those whose tag matches.

**Loading** is the interesting part and has two shapes. If the configured value names a
*section*, that section's `snd_1_layer`, `snd_2_layer`, … keys each become one bank tagged
with the alias — an authored multi-layer sound. If it does not name a section, the value
is loaded as an ordinary single bank, so a weapon authored before layering still works
unchanged.

```text
FUNCTION load_layered(source, section, key, alias, exclusive)
  IF section has no line named key THEN RETURN          # optional
  first = the first comma-separated field of that line
  IF `first` is itself a section THEN
    n = 1
    WHILE that section has a line named "snd_<n>_layer"
      layers.append(a bank loaded from (first, "snd_<n>_layer"), tagged `alias`)
      n = n + 1
  ELSE
    layers.append(a bank loaded from (section, key), tagged `alias`)   # legacy shape
```

**Invariants** — layers are played *simultaneously* by the same call, each choosing its
own take. Passing an explicit index reaches every layer, so a caller selecting take two
gets take two of every layer — which is what keeps a layered reload's mechanism and report
in sync.

**Notes** — both overloads test for the key's existence against the *global* configuration
even when reading from a supplied one. That is a bug: an item whose sounds live in a
private configuration file but whose key does not also exist globally loads nothing. It is
recorded here because a rebuild copying the structure faithfully would reproduce it.

The two overloads are otherwise identical but for their source, and could be one.
