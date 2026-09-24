# src/xrGame/date_time.h

> Declares the two conversions between a calendar date and the game's single scalar clock, implemented in [`date_time.cpp`](date_time.cpp.md).

**Needs** — [`date_time.cpp`](date_time.cpp.md)
**Used by** — [`alife_time_manager.cpp`](alife_time_manager.cpp.md) · [`autosave_manager.cpp`](autosave_manager.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`date_time.cpp`](date_time.cpp.md) · [`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md) · [`game_news.cpp`](game_news.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`level_script.cpp`](level_script.cpp.md) · [`UIInventoryUtilities.cpp`](ui/UIInventoryUtilities.cpp.md) · [`UILogsWnd.cpp`](ui/UILogsWnd.cpp.md) · [`UISleepStatic.cpp`](ui/UISleepStatic.cpp.md) · [`xr_time.cpp`](xr_time.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the pair that defines the game's time representation: `generate_time` packs a
calendar date into one scalar, `split_time` unpacks it. Substance in
[`date_time.cpp`](date_time.cpp.md).

Exported units:

- `generate_time(years, months, days, hours, minutes, seconds, milliseconds)` — calendar to
  scalar. Milliseconds default to zero.
- `split_time(time, → years, months, days, hours, minutes, seconds, milliseconds)` — scalar
  back to calendar.
