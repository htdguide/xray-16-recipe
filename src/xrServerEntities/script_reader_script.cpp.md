# src/xrServerEntities/script_reader_script.cpp

> Exports the read-only chunked stream to scripts, as the exact mirror of the wire buffer's read half.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [`script_net_packet_script.cpp`](script_net_packet_script.cpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: every exported operation is a fixed width or a quantization of a frozen byte stream.

## Purpose

Publishes the engine's read-only stream as the script type `reader`. A script obtains one by
opening a file through the virtual filesystem and uses it to parse engine data files
directly — the spawn file, a save, a custom data blob a mod ships. It is the read companion
to the wire buffer in
[`script_net_packet_script.cpp`](script_net_packet_script.cpp.md), and the two surfaces are
deliberately name-for-name identical so that a script that knows how to *write* a record
knows how to read one back.

## the exported surface

Positioning: seek, tell, advance by a byte count, elapsed bytes since the start, and
end-of-stream.

Fixed widths: the signed and unsigned integers from 8 to 64 bits, a float, a
three-component vector, a boolean, and a null-terminated string.

Quantized forms: a float decoded from 16 or 8 bits across a caller-supplied range, an angle
from 16 or 8 bits across a full turn, a unit direction from 16 bits, and a direction plus
magnitude.

**Invariants** — the range passed to a quantized read must be the same range the writer
used. Nothing in the stream records it, so wrong bounds give silently wrong numbers rather
than an error. This is the same trap the wire buffer sits in and the same trap the ragdoll
snapshot sits in — see [`PHNetState.cpp`](PHNetState.cpp.md).

## Notes

**Boolean is not a stream type.** It is exported as a one-byte read compared against zero,
so a script reading a boolean and a script reading an eight-bit integer are reading the same
byte. Scripts that predate the boolean export do the latter, and the two interoperate
exactly.

**The string read hands back a borrowed reference** into the intern table. It is stable for
the process's lifetime, but a rebuild whose script layer copies strings across the boundary
is safer and costs nothing here — the source itself carries a note questioning the
borrowing.

**No write surface exists on this type.** Writing happens through the wire buffer, which can
be handed to the filesystem; the split is the engine's own (a stream is read-only by
construction) and is worth preserving, because a script that could seek and overwrite inside
an arbitrary opened file is a much larger surface than this chapter wants to own.
