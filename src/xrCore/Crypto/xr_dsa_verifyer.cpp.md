# src/xrCore/Crypto/xr_dsa_verifyer.cpp

> Digests a buffer, checks a signature over that digest, and hands back the digest only if the check passed.

**Needs** — [`xr_dsa_verifyer.h`](xr_dsa_verifyer.h.md) · [`xr_dsa.h`](xr_dsa.h.md) · [`xr_sha.h`](xr_sha.h.md) · [`../FS.h`](../FS.h.md) · [`../LocatorAPI.h`](../LocatorAPI.h.md)
**Used by** — [`xr_dsa_verifyer.h`](xr_dsa_verifyer.h.md)
**Tier floor** — T2: the mirror of the signer, minus the yieldable form.

## Purpose

The client half of the multiplayer configuration check. A server publishes a dump of its configuration plus a signature; the client digests the dump, verifies, and — if it passes — keeps the digest as its record of what it agreed to play under.

## Construction

**Contract** — construct the signature primitive over the three domain numbers and copy the caller's public key into the instance. Asserts at build time that the stored key is exactly the domain's public-key width. Does not validate that the key is a member of the domain; a wrong key is indistinguishable from a wrong signature.

## `verify`

**Contract** — digest the buffer, then check the signature over that digest against the stored public key. On success return the digest; on failure return nothing. No side effects beyond the diagnostic dump below. Does not yield: the whole buffer is digested in one call, so a large dump blocks the calling thread.

```text
FUNCTION verify(data, n, signature) -> optional<bytes[20]>
  h := digest(data, n)
  IF signature_valid(public_key, h, 20, signature)
    RETURN h
  RETURN none
```

**Invariants** — the digest returned is the one that was verified, not a recomputation. That is the point of returning it: a caller cannot accidentally report a digest of different bytes than the ones it checked.

**Notes** — as on the signing side, a diagnostic build writes the exact bytes hashed, a literal marker, and the digest to a log file, so that a mismatch can be localized to the message or the key. The two dumps are written to files whose names differ by one word and are meant to be compared byte for byte.

Without a cryptography provider the digest layer returns an all-zero digest and the primitive returns false, so verification fails closed. See [`xr_sha.cpp`](xr_sha.cpp.md) for why that is luck rather than design, and what a rebuild should do instead.

There is no yielding form here, only on the signing side. The asymmetry is not principled — the client verifies dumps of the same size the server signs — and a rebuild should give verification the same interruptible path.
