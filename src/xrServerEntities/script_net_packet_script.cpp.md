# src/xrServerEntities/script_net_packet_script.cpp

> Exports the wire buffer to scripts, so that a script-declared entity can serialize its own state into exactly the same stream the engine's records use.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`xrNetServer/NET_Shared.h`](../xrNetServer/NET_Shared.h.md) · [Data: network protocol](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`script_reader_script.cpp`](script_reader_script.cpp.md)
**Tier floor** — T1: the exported operations are exact widths and quantizations of a frozen byte stream.

## Purpose

This is the single most consequential script export in the chapter. A script-declared entity
class implements the same three serializations every native record does — spawn, save,
network update — and this is the buffer it writes them into. Every width, every quantization
and every ordering rule the native records obey is therefore *also* part of the script
surface, and a mod's record has to interleave correctly with engine records in the same save
file.

## What is exported

**Writing** — begin a message, tell the write position, and one writer per shape:
three-component vector, float, the signed and unsigned integer widths from 8 to 64 bits, a
boolean (written as one byte, so it costs the same as a small integer), a null-terminated
string, a full transform, a client identifier, and the two quantized forms below.

**Reading** — begin reading (yielding the message type), seek, tell, and the matching
readers. Plus: elapsed time since the message was begun, advance by a byte count, and
end-of-buffer.

**The quantized writers are the part that matters.**

```text
w_float_q16(value, min, max)   # value mapped into 16 bits across [min, max]
w_float_q8 (value, min, max)   # ... into 8 bits
w_angle16(angle)               # an angle in 16 bits across a full turn
w_angle8 (angle)               # ... in 8 bits
w_dir(direction)               # a unit direction in 16 bits
w_sdir(vector)                 # a direction in 16 bits plus its magnitude as a float
```

**Invariants** — the bounds passed to a quantized write must be the *same* bounds the
corresponding read uses. Nothing in the stream carries them, so they are a convention
between the two sides, and a script that writes with one range and reads with another gets
silently wrong numbers rather than an error. This is the same trap the native ragdoll
serialization sits in — see [`PHNetState.cpp`](PHNetState.cpp.md).

**The chunked write bracket** — `open8`/`close8` and `open16`/`close16` write a chunk
header, let the caller write a payload, then go back and fill in the length. It is the
recursive (identifier, size, payload) container the whole engine uses, exposed so a script
record can nest its state the same way.

## Notes

**Boolean is not a wire type.** It is exported as a convenience pair that writes and reads
one byte, added by this project rather than the original. Scripts that predate it write a
byte by hand, and the two interoperate exactly because they are the same byte.

**String reading returns a borrowed pointer.** The read helper interns the string and hands
back a pointer into the intern table; it is stable, but a rebuild whose script layer copies
strings across the boundary is safer and no slower here.

**The client identifier** is a multiplayer concept — it names a connection, not an entity —
and is exported alongside so that a script can read a message's sender.

**A script cannot seek the write cursor.** The raw write-at-position and read-at-position
entry points are deliberately not exported; only the chunk bracket can rewrite earlier
bytes. That is the one guard rail on this surface, and it exists because a script writing
backwards into a shared buffer can corrupt a record that is not its own.
