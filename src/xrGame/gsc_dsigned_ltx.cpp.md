# src/xrGame/gsc_dsigned_ltx.cpp

> Writes and reads a configuration file that carries a signature over its own text, so a client can prove the server's settings were not edited.

**Needs** — [`gsc_dsigned_ltx.h`](gsc_dsigned_ltx.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [Seam: Cryptography](../xrCore/Crypto/README.md)
**Used by** — reached through its declarations in [`gsc_dsigned_ltx.h`](gsc_dsigned_ltx.h.md); callers name that, not this file.
**Tier floor** — T1: signs and verifies over an exact byte range of a text buffer, and mutates that buffer in place

## Purpose

Some configuration must be authoritative: a client that could edit it would be cheating. The
scheme here is deliberately *not* a detached signature file — the signature lives in the
configuration itself, in a trailing section, so the file remains a single readable
configuration that ordinary tooling can open and a human can inspect.

That choice creates the file's one real problem: the signature covers text that the signature
is then appended to, so both writer and reader must agree byte-for-byte on **which prefix of
the file was signed**. Everything interesting here is about maintaining that agreement.

## State

```text
RECORD SignedLtxWriter
  ltx        : Configuration     # built up by the caller before signing
  scratch    : byte buffer       # the serialized form, reused across saves

RECORD SignedLtxReader
  ltx        : optional<Configuration>   # present only after a successful verification
```

**Invariants** — the writer's configuration is created with file-inclusion, override and
save-comment behaviour all switched off: a signed file must be self-contained and must
serialize deterministically, because a comment or an expanded include would change the bytes
under the signature.

## The signed region

The frozen agreement between writer and reader, stated once because both sides re-derive it:

```text
signed bytes = <the configuration's serialized text>
             + <the timestamp, as a zero-terminated string>
```

The timestamp is inside the signature; the signature value is not. The trailing section is
written as:

```text
\r\n[dsign]\r\n	date		=	<timestamp>\r\n	sign_hash	=	<signature>
```

**Invariants** — the line endings are carriage-return/line-feed and the separators are tab
characters, and both are load-bearing: the reader locates the section by scanning for the
literal `dsign` and then stepping back exactly three bytes for `\r\n[`. A rebuild that emits
a bare line feed will produce files its own reader cannot parse.

## `gsc_dsigned_ltx_writer` (construction)

**Contract** — takes the three public signature-scheme parameters and a callback that fills
in the private key. The callback exists so the key can be reconstructed at the moment of use
— split across the binary, derived, unpacked — rather than passed as a value that would sit
in a caller's stack frame. A rebuild is free to pass the key directly; the obfuscation is
not security, and the recipe should not pretend it is.

## `sign_and_save`

**Contract** — serializes the configuration, signs the serialization plus a timestamp, then
writes the configuration followed by the signature section to the destination. Does not
return the signature; the destination gets everything.

```text
FUNCTION sign_and_save(destination)
  timestamp = current time, formatted

  scratch.rewind()
  ltx.serialize_into(scratch)
  boundary = scratch.length            # remember where the configuration text ended
  scratch.append_zero_terminated(timestamp)

  signature = sign(scratch[0 .. scratch.length])   # configuration text + timestamp

  scratch.rewind_to(boundary)          # discard the timestamp from the scratch buffer
  ltx.serialize_into(destination)      # serialize AGAIN, into the real destination
  destination.append_zero_terminated(
      "\r\n[dsign]\r\n\tdate\t\t=\t" + timestamp +
      "\r\n\tsign_hash\t=\t" + signature)
```

**Invariants** — the configuration is serialized **twice**, once into scratch to be signed
and once into the destination. The two serializations must produce identical bytes or the
signature verifies against text nobody has. That is why the configuration is configured to
be deterministic at construction.

**Notes** — rewinding the scratch buffer to the boundary at the end is pointless: the buffer
is rewound to zero at the start of the next call anyway. It is harmless and a rebuild need
not reproduce it.

## `load_and_verify`

**Contract** — takes a mutable buffer holding a whole signed file. Finds the signature
section, reads the timestamp and signature out of it, reconstructs the exact byte range that
was signed, verifies, and on success parses the configuration. Answers whether it verified.
**Mutates the caller's buffer.**

```text
FUNCTION load_and_verify(buffer, size) -> bool
  section = find the LAST occurrence of "dsign" in buffer
  IF not found THEN RETURN false
  section = section - 3                       # step back over "\r\n["
  tail_size = size - offset_of(section)

  parse the tail as a configuration; read `date` and `sign_hash` from the `dsign` section

  # rebuild the signed byte range in place: replace the whole tail with the timestamp
  buffer[section] = end-of-text
  append the timestamp at `section`
  signed_size = size - tail_size + length(timestamp) + 1   # +1 for the terminator

  IF NOT verify(buffer[0 .. signed_size], sign_hash) THEN RETURN false

  buffer[section] = end-of-text               # cut the timestamp back off
  ltx = parse(buffer)                         # the configuration text alone
  RETURN true
```

**Invariants** — the reconstruction must reproduce the writer's signed range exactly,
terminator included, which is why the size arithmetic counts the timestamp's length plus one.
Off by one here and nothing ever verifies.

The buffer is truncated in place at the section boundary before the final parse, so the
configuration the reader exposes never contains the signature section. That is deliberate:
callers read the configuration by key and must not be able to see, or be confused by, the
signature.

**Notes** — the search runs **backwards from the end** of the buffer. That is not an
optimization: the literal `dsign` could appear in a value or a comment earlier in the file,
and the last occurrence is the only one that can be the section header. A forward search
would be a vulnerability, since an attacker controls the configuration's content.

The search reads a fixed-length comparison at each step without re-checking the lower bound
against the buffer start beyond a loop counter, and it assumes the buffer is at least as long
as the literal. A rebuild should bound it properly; the original's assumption holds only
because the caller always supplies a whole file.

On failure the buffer has already been mutated. A caller that wants to retry with a different
key must re-read the file.

## `get_ltx` (reader)

**Contract** — the parsed configuration. Valid **only after a successful verification**;
there is nothing to return before one. A rebuild should make that impossible to get wrong by
returning the configuration from the verification itself.
