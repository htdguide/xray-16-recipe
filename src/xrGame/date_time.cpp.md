# src/xrGame/date_time.cpp

> The game world's calendar: a proleptic Gregorian date packed into a single 64-bit millisecond count, and the exact inverse that unpacks it.

**Needs** — [`date_time.h`](date_time.h.md)
**Used by** — [`date_time.h`](date_time.h.md)
**Tier floor** — T3: integer arithmetic on a calendar

## Purpose

Game time is one number. The alife simulation compares timestamps, the weather system
interpolates by time of day, scripts read and set the date, and save games store it — all
against the same scalar, so the scalar has to be exactly invertible and has to keep the
calendar the scripts expect. This file defines that scalar.

It is not the operating system's clock and deliberately does not use one: the game's clock
starts at a date the game data chooses, runs at a multiple of real time, and can be set
forward by a script when the player sleeps. Borrowing the host's date library would import
time zones, leap seconds and an epoch the game does not want.

## State

Stateless — two pure functions.

The representation is the state that matters:

```text
game_time : int (64-bit)   # milliseconds elapsed since the start of year 1, day 1,
                           #   in a proleptic Gregorian calendar with no time zone,
                           #   no leap seconds and no daylight saving
```

**Invariant** — the year is 1-based, the month is 1-based and the day is 1-based. Year 1,
month 1, day 1, midnight is exactly zero. A rebuild that uses a 0-based month will silently
shift every shipped date by a month.

**Invariant** — `split_time(generate_time(d)) == d` for every representable date, and
`generate_time(split_time(t)) == t` for every representable scalar. The two functions check
this against each other on every call in a debug build — the file's only test, and the
reason both directions can be trusted at all, since there is no test suite in the
repository.

**Invariant** — leap years follow the full Gregorian rule (divisible by four, except
centuries, except multiples of four hundred). The rule is applied uniformly back to year 1,
which no real calendar does; the game's dates are all in the twenty-first century so the
fiction costs nothing, but a rebuild must reproduce the same fiction or the round trip
breaks for dates nobody uses.

## `generate_time`

**Contract** — packs a calendar date into the scalar. Pure, allocates nothing, cannot fail:
out-of-range inputs are not rejected, they simply produce a scalar that unpacks to a
different date. In a debug build it additionally round-trips its own result and
**overwrites its output parameters with the unpacked values**, so a caller passing a day of
zero or a month of thirteen gets its arguments silently normalized in debug and not in
release. That asymmetry is a debug-assertion artifact a rebuild should not reproduce.

```text
FUNCTION generate_time(y, mo, d, h, mi, s, ms) -> int (64-bit)
  # days contributed by the completed years, Gregorian
  n = y - 1
  days = n*365 + n/4 - n/100 + n/400          # integer division
  # days contributed by the completed months of this year
  days += sum of month lengths for months 1 .. mo-1
          # February is 28, plus 1 when y is a leap year
  days += d - 1
  RETURN ((((days*24 + h)*60 + mi)*60 + s)*1000 + ms)
```

**Notes** — the month accumulation is written as a ladder of independent tests rather than a
table lookup. A rebuild should use a table; the ladder is only there to keep the leap-day
adjustment inline with February.

## `split_time`

**Contract** — unpacks the scalar into seven components. Pure, allocates nothing, total for
any scalar whose date is representable. In a debug build it re-packs its own result and
asserts equality.

The time-of-day half is plain successive division. The date half is the interesting part:
it peels the Gregorian cycles off in order of decreasing length, using the exact day count
of each cycle.

```text
FUNCTION split_time(t) -> (y, mo, d, h, mi, s, ms)
  ms = t mod 1000;  t = t / 1000
  s  = t mod 60;    t = t / 60
  mi = t mod 60;    t = t / 60
  h  = t mod 24;    t = t / 24        # t is now a day index, 0-based

  # peel off calendar cycles, longest first
  c400 = t / 146097 ; t = t - c400*146097   # 400*365 + 100 - 4 + 1 leap days
  c100 = t / 36524  ; t = t - c100*36524    # 100*365 + 25 - 1
  c4   = t / 1461   ; t = t - c4*1461       # 4*365 + 1
  c1   = min(t / 365, 3) ; t = t - c1*365   # clamp: see below
  y = 400*c400 + 100*c100 + 4*c4 + c1 + 1
  t = t + 1                                  # day-of-year becomes 1-based

  # walk the months, subtracting each length, February adjusted for leap
  mo = 1
  WHILE t > length_of_month(mo, y)
    t = t - length_of_month(mo, y)
    mo = mo + 1
  d = t
```

**Invariants** — the clamp on the single-year term is not decoration. On the last day of a
leap year within a four-year cycle, the remainder reaches 1460, and 1460/365 is 4, which
would name a fifth year that does not exist in the cycle. Clamping to three forces the
extra day into the day-of-year of the fourth year, which is where the leap day belongs. The
same overflow does not need handling at the hundred- and four-hundred-year levels because
their cycle lengths already absorb it.

**Notes**

- The cycle constants are the exact day counts of the Gregorian cycles and must be written
  as such: 146097 days in four hundred years, 36524 in a non-leap century, 1461 in a
  four-year span. Deriving them at run time is fine; guessing them is not.
- The month walk is written in the source as a nested ladder of eleven conditionals rather
  than a loop. It is a loop; the nesting has no meaning.
- A block of self-test calls for leap-year boundaries is present in the file but disabled.
  It documents the cases the author cared about — year 1, the first few years, and each
  century boundary from 1600 — and is the closest thing to a unit test in this chapter.
