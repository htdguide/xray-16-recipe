# src/xrGameSpy/GameSpy_Available.cpp

> The reachability probe: asks whether the matchmaking service is answering for this title
> at all, and blocks until it knows. This is the call that returns "no" forever now, and
> the whole chapter's null path hangs off it.

**Needs** — [`GameSpy_Available.h`](GameSpy_Available.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_Available.h`](GameSpy_Available.h.md)
**Tier floor** — T3. It is a request, a poll loop and a three-way branch.

## Purpose

Before the engine opens an account session, queries a server list or registers a server,
it asks one question: *is this title's online service live?* The answer is three-valued,
and the three values mean different things to a player — gone for good, down for
maintenance, or fine — so the probe returns a message alongside the verdict.

This is the only place in the chapter that blocks. Everything else is polled from the
frame loop.

## State

`Stateless.`

## `CheckAvailableServices`

**Contract** — starts an availability query for this title, then spins until the query
settles, sleeping 5 ms between polls. Returns `true` only on a definite *available*;
otherwise returns `false` and writes a player-facing sentence into the caller's slot
explaining which kind of unavailable it is. Blocks the calling thread — at startup that
is the main thread, which is why the sleep exists at all: without it the probe would burn
a core waiting on a network round trip. No allocation beyond the message. Not
thread-safe and does not need to be; it is called once per service bring-up.

```text
ENUM Availability
  available          # the service is answering for this title
  unavailable        # the service has been retired for this title — permanent
  down_temporarily   # the service exists but is in maintenance — retry later
  waiting            # no answer yet

FUNCTION check_available_services() -> result<text, text>
  start_availability_query(title.short_name)
  verdict <- waiting
  WHILE verdict == waiting
    verdict <- poll_availability_query()
    sleep(5 ms)                    # a network round trip, not a spin
  IF verdict == available
    RETURN ok("Success")
  IF verdict == unavailable
    RETURN error("! Online Services for STALKER are no longer available.")
  IF verdict == down_temporarily
    RETURN error("! Online Services for STALKER are temporarily down for maintenance.")
  RETURN error("")                 # no other verdict is produced
```

**Invariants** — the poll must eventually leave `waiting`; the probe has no timeout of its
own and relies on the query itself to fail rather than hang. That is a real hazard in a
rebuild: a query against a hostname that does not resolve settles quickly, but one against
a host that accepts and never answers would stall the engine's startup indefinitely. **Put
a timeout here.**

**Notes**

The two sentences are literal English embedded in the engine, not entries in the localized
string table that every other player-facing message in the game comes from. That is an
inconsistency, not a decision; a rebuild should route them through the string table.

The distinction between *retired* and *in maintenance* is the load-bearing part. A
rebuild that stubs this module out should return *retired*, not *in maintenance*: the two
lead to different behaviour upstream — [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md) reports
the failure to the player exactly once and then stops complaining, which is the correct
behaviour for a permanent condition and the wrong one for a transient.

**This call is the null path's origin.** Today it returns *unavailable* — the service was
retired in 2014 and no host answers. Every other call in this chapter still executes
against a dead endpoint and fails on its own timetable; only this one fails
*deliberately*, early, and with a reason the rest of the engine reads.
