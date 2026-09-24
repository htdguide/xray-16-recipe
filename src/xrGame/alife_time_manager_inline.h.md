# src/xrGame/alife_time_manager_inline.h

> The derived clock reading and its three mutations — the file that actually defines what game time is.

**Needs** — [`alife_time_manager.h`](alife_time_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an affine map read on every query

## Purpose

Despite being an inline-bodies file, this is where the clock's behaviour lives; the
`.cpp` holds only initialization and persistence. Read it together with
[`alife_time_manager.cpp`](alife_time_manager.cpp.md), which states the invariant these
five operations maintain.

## `game_time`

**Contract** — the current calendar value, derived rather than accumulated. Pure and
cheap; called from everywhere, every frame.

```text
FUNCTION game_time() -> int (64-bit)
  RETURN anchor_value + round(time_factor * (device_clock - anchor_device_time))
```

## `set_time_factor`

**Contract** — changes how fast time passes, without the calendar jumping.

```text
FUNCTION set_time_factor(new_factor)
  anchor_value       = game_time()
  anchor_device_time = device_clock
  time_factor        = new_factor
```

**Invariants** — freeze, re-anchor, then change. Changing the multiplier first would
rescale every millisecond since the last anchor and the clock would visibly jump.

## `set_game_time_factor`

**Contract** — jumps the calendar to a given value *and* sets the multiplier. The same
re-anchor, with the frozen value supplied by the caller instead of read from the clock.
This is how "sleep until dawn" is implemented.

## `change_game_time`

**Contract** — adds an offset to the calendar. Does not re-anchor, and does not need to:
adding a constant to the anchor shifts the entire affine map by that constant, which is
precisely the intended effect.

## `start_game_time`, `time_factor`, `normal_time_factor`

**Contract** — read the authored calendar start, the multiplier currently in force, and
the authored default multiplier. The last exists so that a script which accelerated time
can restore the correct speed without having recorded it.
