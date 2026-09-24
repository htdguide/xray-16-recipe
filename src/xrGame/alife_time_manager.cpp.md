# src/xrGame/alife_time_manager.cpp

> The game clock: an in-world calendar that advances at a configurable multiple of real time, with the multiplier changeable mid-game without the clock jumping.

**Needs** — [`alife_time_manager.h`](alife_time_manager.h.md) · [`date_time.h`](date_time.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an affine map from the device clock, persisted

## Purpose

Everything in the game that has a time — the weather cycle, an anomaly's respawn, a
trader's restock, a creature's sleep schedule, a quest deadline — reads this clock. It is
not the frame clock and not the wall clock: it is a **calendar** with a start date, a
multiplier, and an anchor.

The decision the whole file rests on: game time is not accumulated per frame. It is
**derived** from the device's monotonic clock by an affine map, and the map is re-anchored
whenever the multiplier changes. Deriving rather than accumulating means the clock cannot
drift, cannot be advanced twice by a duplicated update, and costs nothing to read.

## State

```text
RECORD TimeManager
  start_game_time  : int (64-bit)   # the authored calendar start; never changes
  game_time        : int (64-bit)   # the calendar value at the last anchor
  start_time       : int (32-bit)   # the device clock reading at that anchor
  time_factor      : real           # game milliseconds per device millisecond
  normal_time_factor : real         # the authored default, for restoring after a change
```

Invariant tying them: at any moment,

```text
current_game_time = game_time + time_factor * (device_clock - start_time)
```

Every mutation preserves this by **re-anchoring**: recompute the current value into
`game_time`, reset `start_time` to now, then change the multiplier. Changing the
multiplier without re-anchoring would retroactively rescale all the time since the last
anchor, and the clock would jump.

## `init`

**Contract** — reads the calendar's start and its speed from a configuration section, and
anchors the clock at the current device time. Called when a new game starts.

```text
FUNCTION init(section)
  parse configuration(section, "start_time") as hours:minutes:seconds
  parse configuration(section, "start_date") as days.months.years
  start_game_time    = calendar_to_time(years, months, days, hours, minutes, seconds)
  time_factor        = configuration(section, "time_factor")
  normal_time_factor = configuration(section, "normal_time_factor")
  game_time          = start_game_time
  start_time         = device_clock
```

**Invariants** — the start date is authored per game, in the configuration, and is the
reason the three shipped titles begin on different in-world dates. The calendar
conversion is a real date computation, not an offset: the game has months and years and
the weather system reads them.

Two multipliers are read. The **time factor** is the one in force; the **normal time
factor** is the authored default, kept so that a script that accelerated time (during a
sleep, or a cutscene) can restore the correct speed without knowing what it was. A rebuild
that keeps only one loses the ability to restore.

The multiplier is large — game time runs many times faster than real time — which is what
makes a day/night cycle and an offline economy observable within a play session.

## `game_time`

**Contract** — the current calendar value. Pure, cheap, called constantly.

```text
FUNCTION game_time() -> int
  RETURN game_time_anchor + round(time_factor * (device_clock - start_time_anchor))
```

**Notes** — the device clock reading is 32-bit milliseconds and the subtraction is done at
that width before being scaled into the 64-bit calendar. A session long enough to wrap the
device clock would produce a discontinuity; §4 requires a monotonic clock and the
practical session length makes this unreachable, but a rebuild should do the subtraction
at 64 bits and remove the question.

## `set_time_factor` and `set_game_time_factor`

**Contract** — change the speed of time, and (in the second form) also jump the calendar
to a given value. Both re-anchor.

```text
FUNCTION set_time_factor(new_factor)
  game_time  = game_time()          # freeze the current value
  start_time = device_clock         # re-anchor
  time_factor = new_factor

FUNCTION set_game_time_factor(new_time, new_factor)
  game_time   = new_time            # jump
  start_time  = device_clock
  time_factor = new_factor
```

**Invariants** — the first two lines of each are the re-anchor and must precede the
change. This is the file's one real rule and it appears four times (here, in the second
setter, and in save and load); a rebuild should extract it once.

The second form is how "sleep until morning" works: the script computes the target
calendar value and sets it directly, rather than accelerating time and waiting.

## `change_game_time`

**Contract** — adds an offset to the anchored calendar value without re-anchoring or
changing the multiplier. Used to nudge the clock forward by a known amount.

**Notes** — this one does **not** re-anchor, and it does not need to: adding a constant to
the anchor shifts the whole affine map by that constant, which is exactly the intent. It
is the only mutation that is safe without re-anchoring, and the reason is worth stating
because it looks inconsistent with its neighbours.

## `save`

**Contract** — writes the clock into its own chunk of the save. Re-anchors first.

```text
FUNCTION save(stream)
  game_time  = game_time()      # freeze
  start_time = device_clock     # re-anchor
  open chunk GAME_TIME_CHUNK_DATA
    stream.write_bytes(game_time)          # 64-bit, raw
    stream.write_real(time_factor)
    stream.write_real(normal_time_factor)
  close chunk
```

**Invariants** — the anchor is **not** saved, and must not be: the device clock means
nothing across a restart. What is saved is the frozen calendar value and the two
multipliers; the anchor is re-established on load from whatever the device clock then
reads.

Freezing before writing is why saving does not lose the time elapsed since the last
anchor.

The calendar value is written as raw bytes of a 64-bit integer, little-endian by the
platform assumption in §4. **Frozen.**

Note the asymmetry with the simulator header: the start date is not saved, so a loaded
game has no record of when it began. Nothing reads it after a load, which is why it works.

## `load`

**Contract** — reads the clock back and re-anchors to the current device time. Fails hard
if the chunk is missing.

```text
FUNCTION load(stream)
  REQUIRE stream has chunk GAME_TIME_CHUNK_DATA
  game_time          = stream.read_bytes(8)
  time_factor        = stream.read_real()
  normal_time_factor = stream.read_real()
  start_time         = device_clock       # the anchor is always re-established here
```

**Invariants** — the multiplier in force at the moment of saving is restored, not the
authored default. So a game saved during an accelerated period resumes accelerated, which
is correct — the script that accelerated it is also restored and will restore the default
when it is done.
