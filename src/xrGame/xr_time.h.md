# src/xrGame/xr_time.h

> Declares the script-visible game-time value: comparison, arithmetic, calendar split and formatting over one millisecond count.

**Needs** — [`xr_time.cpp`](xr_time.cpp.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`saved_game_wrapper_script.cpp`](saved_game_wrapper_script.cpp.md) · [`script_game_object.h`](script_game_object.h.md) · [`xr_time.cpp`](xr_time.cpp.md)
**Tier floor** — T3: a value type over one 64-bit integer

## Purpose

Declares the surface implemented in [`xr_time.cpp`](xr_time.cpp.md). This is a *value*
type, exported to the script layer, and the fact that it is a value rather than a handle is
its main design decision: scripts copy, compare and store game times freely without any
lifetime question.

Exported units:

- construction from nothing (zero), from a copy, or from a raw millisecond count.
- the six comparisons — less, greater, less-or-equal, greater-or-equal, equal, and by
  omission *not* not-equal, which scripts must express as a negated equality.
- `+` and `-` — non-mutating arithmetic. **Neither saturates**, unlike `add`/`sub`.
- `add`, `sub` — mutating arithmetic; subtraction saturates at zero.
- `diffSec` — the signed difference in seconds.
- `setHMS`, `setHMSms`, `set` — replace the value from calendar components.
- `get` — split into seven calendar components.
- `dateToString`, `timeToString` — localized formatting at a requested precision.
- `get_time` — the current game time truncated to 32 bits.
- `get_time_struct` — the current game time, full width, wrapped.

## State

```text
RECORD GameTime
  time : int (64-bit)      # milliseconds since the game's epoch
```

**Notes** — Arithmetic is exposed twice, once as operators and once as named methods, and
the two forms **disagree on subtraction**: the named one clamps at zero, the operator does
not. Both are in the script surface and so both are frozen; a rebuild reproducing the script
surface must reproduce the disagreement, and should document it rather than fix it.

The underlying count is the alife simulation's own time type. Sharing the type is what lets
a game time be compared against an alife event timestamp without conversion, and it is why
this file depends on the alife declarations for nothing but that one alias.
