# src/xrGame/RegistryFuncs.cpp

> Reads and writes a handful of values in the host's machine-wide settings store, under the key the retail installer created — the only place the engine keeps state outside its own files.

**Needs** — [`RegistryFuncs.h`](RegistryFuncs.h.md) · [`xrGameSpy/xrGameSpy_MainDefs.h`](../xrGameSpy/xrGameSpy_MainDefs.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — reached through its declarations in [`RegistryFuncs.h`](RegistryFuncs.h.md); callers name that, not this file.
**Tier floor** — T3: a key-value store wrapper

## Purpose

The matchmaking layer needs two things the game's own files cannot hold: the serial key
the retail installer recorded, and the account credentials the player asked to be
remembered. Both must survive a reinstall of the game and must be shared with the
installer and with other copies of the game, so they live in the operating system's
machine-wide settings store, under a fixed path that the installer owns.

That is the entire justification for this file. Everything else the engine persists goes
through its virtual filesystem.

The path is fixed at build time and the hive is the machine-wide one, not the per-user
one. Both choices come from the installer, not from the game, and a rebuild that has no
installer to match should put this data in the engine's own settings directory and delete
the file.

## State

`Stateless.` The store is the operating system's.

**Invariants** — three value shapes are supported and their widths are fixed, not derived
from the data:

- a string value is always read and written as exactly 64 bytes;
- a numeric value is always exactly 4 bytes;
- a binary value is variable, bounded by a caller-supplied buffer size.

The fixed 64 is a contract with the caller, not a property of the store: a caller passing
a smaller buffer to the string read has its memory overwritten. A rebuild should pass the
buffer's own size and let the store report the value's length, which every real settings
API supports.

## `ReadRegistry_StrValue` / `WriteRegistry_StrValue`

**Contract** — read or write a fixed-width string value by name under the installer's key.
The read reports whether it succeeded; the write reports nothing. Both log a diagnostic
naming the missing key or value when the store cannot be reached. Neither blocks
meaningfully.

**Invariants** — the caller's buffer must be at least 64 bytes.

## `ReadRegistry_DWValue` / `WriteRegistry_DWValue`

**Contract** — read or write a 4-byte numeric value by name. The read *does not report
failure*: on a missing value the caller's variable is left untouched, so the caller must
pre-initialize it to a meaningful default or it will act on whatever was there.

## `ReadRegistry_BinaryValue` / `WriteRegistry_BinaryValue`

**Contract** — read or write an opaque byte range by name. The read takes a destination
buffer and its capacity and returns the number of bytes actually read, or zero on any
failure; the store truncates or refuses if the value exceeds the capacity. The write takes
a source range and its length and reports nothing.

**Notes** — the read path opens the store's key and, on every exit path, does not close
it. Each call therefore consumes a handle permanently. It is called a few times at
startup so it never became visible, but a rebuild must scope the handle to the operation;
this is exactly the kind of manual-resource bug that a rebuild in almost any other tier
cannot write.

## Platform behaviour

**Contract** — this store exists only on one operating system family. Everywhere else,
every function in the file is a stub, and the stubs' return values are the load-bearing
part:

| Function | Off-platform result |
|---|---|
| string read | reports **success**, writes nothing |
| numeric read | writes nothing, reports nothing |
| binary read | reports **zero bytes read** |
| all writes | do nothing |

The string read reporting success while leaving the caller's buffer untouched is a trap: a
caller that checks the return value and then uses the buffer reads uninitialized memory on
every non-Windows platform. The binary read's zero is the honest answer and is what every
stub should return. A rebuild should give the whole idea one implementation — a small
key-value file in a per-user configuration directory — and have no platform branches at
all, which is what makes this entire file incidental.
