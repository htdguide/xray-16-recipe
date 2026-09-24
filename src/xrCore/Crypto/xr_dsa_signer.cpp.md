# src/xrCore/Crypto/xr_dsa_signer.cpp

> Digests a buffer and signs the digest, in one call or in yieldable pieces.

**Needs** — [`xr_dsa_signer.h`](xr_dsa_signer.h.md) · [`xr_dsa.h`](xr_dsa.h.md) · [`xr_sha.h`](xr_sha.h.md) · [`../FS.h`](../FS.h.md) · [`../LocatorAPI.h`](../LocatorAPI.h.md)
**Used by** — [`xr_dsa_signer.h`](xr_dsa_signer.h.md)
**Tier floor** — T2: two calls and a timestamp format.

## Purpose

The signature primitive ([`xr_dsa.cpp`](xr_dsa.cpp.md)) signs 20 bytes. Callers have megabytes. This closes that gap, and it is the only place the two halves are joined — which is also why the size relationship between them is asserted here rather than anywhere else.

## `sign`

**Contract** — digest the caller's buffer, then sign the digest with the private key the subclass installed. Returns the signature as hexadecimal text, or the empty string if the build has no cryptography provider. Blocks for the whole buffer with no opportunity to yield.

```text
FUNCTION sign(data, n) -> text
  h := digest(data, n)                       # 20 bytes
  RETURN sign_digest(private_key, h, 20)
```

**Invariants** — the digest length and the private key length must be equal, and the file asserts it at build time. The assertion is the documentation: the signature domain's subgroup order is 160 bits, so the thing signed must be 160 bits, so the digest algorithm is fixed by the domain.

**Notes** — in a diagnostic build both this and the verifier dump the exact bytes they hashed, followed by a literal marker and the digest, to a log file. That exists because the two sides disagreeing is otherwise undiagnosable — the only symptom is a signature that does not verify, with no indication whether the message differed or the key did. A rebuild should keep the capability and make it a run-time switch rather than a build flavour, since the failure it diagnoses happens in shipped builds.

## `sign_mt`

**Contract** — identical, except the digest is computed with the caller's yield callback firing once per 64-byte block. The signing step itself is not interruptible and is short. The name promises threading and delivers cooperative yielding on the calling thread — the same misnomer as in [`ppmd_compressor.cpp`](../Compression/ppmd_compressor.cpp.md), and the same honest description applies.

## `current_time`

**Contract** — format the current local wall-clock time into a caller-supplied 64-byte buffer as `DD.MM.YYYY_HH:MM:SS`, zero-padded, and return the buffer. Uses local time, not UTC.

**Invariants** — this exact layout is written into the signed configuration dump and read back from it, so it is **part of that file's format**: two-digit day, two-digit month, four-digit year, underscore, then a 24-hour clock with colons. The separators matter as much as the fields.

**Notes** — using local time means the recorded timestamp is only meaningful alongside the machine that produced it, and two servers in different zones stamp the same instant differently. Nothing compares timestamps across machines, so it has never mattered; a rebuild should use UTC and accept that old dumps read an offset hour.
