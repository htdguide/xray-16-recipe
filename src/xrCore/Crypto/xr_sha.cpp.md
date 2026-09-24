# src/xrCore/Crypto/xr_sha.cpp

> Produces the 20-byte digest of a buffer, block by block, yielding to the caller between blocks. Returns an all-zero digest when the engine is built without a cryptography provider.

**Needs** — [`xr_sha.h`](xr_sha.h.md)
**Used by** — [`xr_sha.h`](xr_sha.h.md)
**Tier floor** — T2: the hash itself is an external dependency; this file is a loop and a failure policy.

## Purpose

The multiplayer anti-tamper path signs a dump of the server's configuration so that clients can verify they are playing the same rules. Signing a megabyte directly is impractical, so it signs a digest instead, and this produces it. The digest algorithm is SHA-1, chosen in 2007 and now only as strong as the threat model deserves — which, since the private key ships inside a client binary, is not very.

The engine does not implement the hash. It is an **external cryptography provider**, and the whole file is conditional on one being linked. Everything below therefore splits into two behaviours.

## `calculate_with_yielder`

**Contract** — hash a buffer and return the digest. Feeds the provider in 64-byte blocks, invoking the caller's yielder after each with the number of blocks completed *before* it. Returns an all-zero digest on any failure: a null or empty input, a provider context that cannot be created, or an algorithm that cannot be initialized. Allocates a provider context and always releases it, including on the failure paths. Does not block except inside the caller's yielder. Thread-safety is the provider's.

```text
FUNCTION digest(data, size, yielder) -> bytes[20]
  IF no provider is available
    RETURN all zeroes                      # see the note — this is the danger
  IF data IS ABSENT OR size == 0
    RETURN all zeroes
  ctx := new digest context OR RETURN all zeroes
  initialize ctx for SHA-1 OR (release ctx; RETURN all zeroes)
  done := 0
  WHILE size > 0
    take := min(64, size)
    feed ctx the next `take` bytes
    yielder(done)                          # fires BEFORE the count increments
    done := done + 1
  result := finalize ctx                   # 20 bytes
  release ctx
  RETURN result
```

**Invariants** — the yielder sees a zero-based count of blocks *already fed*, and it fires for the final partial block too, so a caller computing a percentage must use the total block count including the remainder.

**Notes** — the failure policy is the load-bearing decision on this page, and it is a bad one that a rebuild must fix rather than reproduce. **Every failure returns an all-zero digest, which is a valid-looking digest.** A verifier handed two all-zero digests compares them equal. So a build without a provider does not fail closed — it signs the zero digest and verifies the zero digest, and every signature "matches". The signing side at least logs a warning (see [`xr_dsa.cpp`](xr_dsa.cpp.md)); this side does not.

A rebuild should make the result an explicit "no digest" that no comparison accepts, and make the absence of a provider a build-time failure rather than a run-time zero.

Feeding the provider in 64-byte blocks matches SHA-1's own block size, so the provider buffers nothing between calls — but that is incidental. The block loop exists for the yielder, and a provider with any other internal block size would be fed identically.
