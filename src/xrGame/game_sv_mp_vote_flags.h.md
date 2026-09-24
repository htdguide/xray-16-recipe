# src/xrGame/game_sv_mp_vote_flags.h

> The bit set naming which kinds of player vote a server permits.

**Needs** — _(none)_
**Used by** — [`game_cl_base.cpp`](game_cl_base.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`UIVotingCategory.cpp`](ui/UIVotingCategory.cpp.md)
**Tier floor** — T3: a constant set

## Purpose

A server operator enables voting per kind, not all-or-nothing, so the permission is a mask
in a console variable. The values are frozen because they appear in shipped server
configuration files as a number.

## State

```text
ENUM VoteKind          # bit positions in a 16-bit mask
  full_vote      = 1 << 0   # the pre-existing vote syntax, kept for old configurations
  restart        = 1 << 1
  restart_fast   = 1 << 2
  kick           = 1 << 3
  ban            = 1 << 4
  change_map     = 1 << 5
  change_weather = 1 << 6
  change_gametype = 1 << 7
```

**Invariants** — bit zero is a compatibility flag, not a vote subject: it selects the older
command grammar rather than permitting a kind of vote. A rebuild that treats the mask
uniformly will silently enable or disable the wrong thing.
