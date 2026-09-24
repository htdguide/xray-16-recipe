# src/xrGame/xr_time.cpp

> The game clock as a value: a single millisecond count that the scripts add to, subtract from, and format as a calendar date.

**Needs** — [`xr_time.h`](xr_time.h.md) · [`Level.h`](Level.h.md) · [`date_time.h`](date_time.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`ui/UIInventoryUtilities.h`](ui/UIInventoryUtilities.h.md)
**Used by** — [`xr_time.h`](xr_time.h.md)
**Tier floor** — T3: arithmetic on one integer plus two formatting delegations

## Purpose

In-game time is a 64-bit millisecond count since a fixed epoch, and scripts need to do
calendar arithmetic on it — "three days from now", "how many seconds since I last saw
him", "print the date". This wraps the raw count so that arithmetic on it is total: it
cannot go negative, and the calendar split is one call rather than a chain of divisions at
every use site.

The free functions at the top answer a different question: *which* clock is the game clock
right now. That answer is not constant and the distinction is load-bearing.

## State

```text
RECORD GameTime
  time : int (64-bit)      # milliseconds since the game's epoch; never negative
```

**Invariants** — the count is unsigned and saturating at zero on subtraction through the
mutating operation. The *operator* form of subtraction is not saturating, which is an
inconsistency — see Notes.

## `__game_time` and `get_time`

**Contract** — the current game time. If the alife simulation is running, its time manager
is authoritative; otherwise the loaded level's own clock is. `get_time` truncates the
result to the lower 32 bits; `get_time_struct` returns the full value wrapped.

**Notes** — **There are two clocks and the alife one wins when it exists.** The level clock
runs only while a level is loaded; the alife clock runs across the whole world including
while nothing is loaded, and is what a save restores. A rebuild must keep the precedence: a
query answered by the level clock during a period the alife simulation was running would
report a time that never advanced while the player was elsewhere.

The truncation to 32 bits is what every script-visible "what time is it" call returns, which
caps the representable span at about 49 days of game time before the value wraps. The games
are shorter than that, so it has never bitten, but it is a real limit and a rebuild that
exposes the full width is strictly safer. The truncated form exists because the script layer's
number type could not carry the full width faithfully.

## `add`, `sub`

**Contract** — in-place addition and subtraction of another time value. Subtraction
**saturates at zero**: subtracting a larger time yields zero rather than wrapping.

**Notes** — the saturation is the whole reason these exist alongside the operators. A script
computing "now minus the interval" on a fresh game, where now is small, would otherwise get
an enormous positive value and conclude that a very long time had passed. The operator form
of subtraction does *not* saturate, so the two forms disagree; that is a defect in the
original and a rebuild should make both saturate.

## `setHMS`, `setHMSms`, `set`

**Contract** — three constructors-by-assignment of increasing specificity: hours/minutes/
seconds, the same plus milliseconds, and a full year/month/day/hour/minute/second/
millisecond. Each *replaces* the value rather than adding to it.

**Invariants** — the two short forms place the time on **year 1, month 1, day 1**. They are
therefore *durations expressed as a time of day*, not absolute moments: a script uses them
to build an interval to add or subtract, not to name a date. Using one where an absolute
time was meant yields a moment near the epoch, which compares less than every real game time.

## `get`

**Contract** — split the value into its seven calendar components. Pure.

## `diffSec`

**Contract** — the signed difference between two times, in seconds. Positive when this time
is the later one. Unlike `sub`, this does not saturate — the sign is the point.

**Notes** — a signed difference in seconds and a saturating subtraction in milliseconds
coexist because they answer different questions: "how long ago" wants a signed real, "what
time was it then" wants a clamped time. A rebuild should keep both and name them so the
difference is visible.

## `dateToString`, `timeToString`

**Contract** — format the value as a localized date or time string at a requested precision.
Delegates to the inventory-screen formatting helpers, which own the localized month names
and the precision levels. Returns a borrowed string that is valid only until the next call
— the helpers format into shared storage.

**Notes** — that the *time* type reaches into the *inventory user interface* for its
formatting is backwards; the formatting belongs with the clock and the interface should call
it. A rebuild should invert the dependency. The precision argument is passed as a plain
integer because it crosses the script boundary, where the enumeration would not survive; the
values are the enumerations' own and are therefore frozen by the script surface.
