# src/xrCore/Crypto/xr_sha.h

> Declares the 20-byte digest used as the thing a signature actually signs, and the yield hook that keeps hashing a large buffer from freezing a frame.

**Needs** — [`xr_sha.cpp`](xr_sha.cpp.md) · [`../fastdelegate.h`](../fastdelegate.h.md)
**Used by** — [`configs_dump_verifyer.cpp`](../../utils/mp_configs_verifyer/configs_dump_verifyer.cpp.md) · [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md) · [`xr_dsa_signer.h`](xr_dsa_signer.h.md) · [`xr_dsa_verifyer.cpp`](xr_dsa_verifyer.cpp.md) · [`xr_dsa_verifyer.h`](xr_dsa_verifyer.h.md) · [`xr_sha.cpp`](xr_sha.cpp.md)
**Tier floor** — T2: a digest, a block size, and a callback per block.

## Purpose

Declares the surface implemented in [`xr_sha.cpp`](xr_sha.cpp.md). It also declares the *yielder* — a callback invoked once per processed block, carrying a progress count — which is not really about hashing at all and the source says so: it is the engine's general shape for "a long computation that must be interruptible from a game thread", and it lives here only because hashing is the first thing that needed it.

## Exported units

- **`xr_sha1::DigestSize`** — 20 bytes. This is a *constraint*, not a convenience: the signature layer requires its private key to be exactly this size, and asserts it.
- **`xr_sha1::BlockSize`** — 64 bytes. It is the granularity at which the yielder fires, not an algorithmic parameter the caller may change.
- **`xr_sha1::hash_t`** — the 20-byte digest as a value.
- **`calculate(data, size)`** — the whole digest, with no yielding.
- **`calculate_with_yielder(data, size, yielder)`** — the same, calling the yielder once per block with a running block count.
- **`yielder_t`** — the callback shape: takes a progress count, returns nothing.
- **`EmptyYielder`** — the do-nothing yielder that the non-yielding form passes.

**Notes** — the type cannot be instantiated; it exists only to group two functions. That is a C++ way of writing a namespace and is incidental.

A yielder firing every 64 bytes is a very fine granularity — a megabyte of configuration fires it sixteen thousand times — and the real cost of the design is that the callback dominates the hash. The one caller that uses it is the multiplayer configuration dump, which runs on a scheduled task and wants to hand control back often. A rebuild should make the yield interval a parameter.
