# src/xrCore/Crypto/xr_dsa_signer.h

> Declares the digest-then-sign wrapper, and the private key slot a subclass is expected to fill.

**Needs** — [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md) · [`xr_dsa.h`](xr_dsa.h.md) · [`xr_sha.h`](xr_sha.h.md)
**Used by** — [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md) · [`configs_dumper.h`](../../xrGame/configs_dumper.h.md) · [`gsc_dsigned_ltx.h`](../../xrGame/gsc_dsigned_ltx.h.md)
**Tier floor** — T2: a digest call and a sign call, over fixed-width key material.

## Purpose

Declares the surface implemented in [`xr_dsa_signer.cpp`](xr_dsa_signer.cpp.md). Its one structural decision is worth naming here because it shapes every user: **the private key is a protected field, not a constructor argument.** A concrete signer derives from this type and fills the field in its own constructor, typically by assembling the bytes from scattered constants so the key is not a contiguous literal in the binary. That is the entire reason the type is meant to be inherited from rather than used directly.

## Exported units

- **`xr_dsa_signer(p, q, g)`** — construct over a signature domain. The private key starts zeroed and is the subclass's responsibility.
- **`m_private_key`** — the 20-byte slot the subclass fills.
- **`sign(data, size)`** — digest the buffer, sign the digest, return the signature as hexadecimal text. Blocks for the whole buffer.
- **`sign_mt(data, size, yielder)`** — the same, invoking the yielder once per digest block so a scheduled task can give up control.
- **`current_time(buffer)`** — a free function, not part of the signer: formats local wall-clock time as `DD.MM.YYYY_HH:MM:SS` into a 64-byte buffer and returns it.

**Notes** — `current_time` lives here because the signed configuration dump carries a creation timestamp in that exact format, and the format is therefore part of what a verifier reads. It has nothing else to do with signing, and a rebuild should move it beside the dump writer.

The default constructor is private and builds a signer over an absent domain. It exists only so that a subclass with its own construction order can compile; it cannot produce a working signer.
