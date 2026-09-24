# src/xrGameSpy/xrGameSpy.cpp

> Answers three questions about *this* copy of the game that the matchmaking layer has to
> put on the wire: what version it is, which title it authenticates as, and which
> distribution the installer laid down.

**Needs** — [`xrGameSpy.h`](xrGameSpy.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3 for two of the three; T1 for the distribution query, which reads the
operating system's installation settings store, and only because the original chose that
store. Move the value into the engine's own settings file and the whole file is T3.

## Purpose

Three values travel outward from this module and nowhere else produces them: the version
string that is advertised in every server record and compared client-to-server before a
connection is allowed, the numeric title id that the key-authentication and
statistics-reporting paths both address, and a *distribution* number identifying which
retail edition is installed.

They are here rather than in the constants header because two of them are lookups, not
constants: the title id is chosen per build variant, and the distribution number is read
back from whatever the installer wrote.

## State

`Stateless.` Every call recomputes.

## `GetGameVersion`

**Contract** — returns the build's version string (`"1.6.02"` as shipped). No failure
path. The string is stable for the lifetime of the process and is safe to hand out
without copying.

This value is load-bearing in three places at once: it is advertised as a server-record
field so a browsing client can see it, it is compared during connection setup so that
mismatched builds are refused with a distinct error rather than a protocol desync, and it
is reported alongside statistics. A rebuild must have *some* such string and must compare
it exactly; it need not match the original's.

## `GetGameID`

**Contract** — writes the numeric title id into the caller's slot. Takes a variant
selector which the shipping build ignores entirely: only a demo build maps the selector
onto three alternative ids, and the demo build is switched off.

```text
FUNCTION get_game_id(variant : int) -> int
  RETURN title.numeric_id        # 2760 for this build
```

**Notes** — the out-parameter and the unused selector are both residue. The function
exists because the id was once per-variant; a rebuild exposes it as a constant.

## `GetGameDistribution`

**Contract** — reads the installer-written *patch identity* number out of the machine's
settings store and returns it. Returns `0` when the store, the key, or the value is
absent — which is the answer on every platform except the original's and in every
installation the engine did not itself come from. Does not block meaningfully; touches no
network.

```text
FUNCTION get_game_distribution() -> int
  value <- read_machine_setting(installation_key, "InstallPatchID")
  IF value IS none THEN RETURN 0
  RETURN value
```

**Notes**

The value distinguishes retail editions of the game — which languages the disc carried,
which copy protection it used, whether the key binds to a disc or to hardware. The source
carries a commented-out table of the eight shipped edition codes (combinations of
language set, protection scheme and binding method) which is the only surviving
description of what the number means; nothing in the engine branches on it. It is
produced here and reported outward, and that is all.

**Could not recover** — which consumer, if any, ever acted on the distribution number.
The engine reads it and advertises it; no code path in this repository changes behaviour
based on its value. A rebuild can return a constant.

The bit-mask definitions for edition languages, protection scheme and key binding that sit
above this function are unused by any code here; they document the encoding the installer
used and are otherwise inert.
