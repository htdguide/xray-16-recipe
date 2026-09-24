# src/xrGame/gsc_dsigned_ltx.h

> Declares the writer and reader for a digitally signed configuration file.

**Needs** — [`xrCore/Crypto/xr_dsa_signer.h`](../xrCore/Crypto/xr_dsa_signer.h.md) · [`xrCore/Crypto/xr_dsa_verifyer.h`](../xrCore/Crypto/xr_dsa_verifyer.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [Seam: Cryptography](../xrCore/Crypto/README.md)
**Used by** — [`gsc_dsigned_ltx.cpp`](gsc_dsigned_ltx.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`gsc_dsigned_ltx.cpp`](gsc_dsigned_ltx.cpp.md): a pair
of types that write and read an `ltx` configuration file carrying a signature over its own
contents, used for server-authoritative settings a client must not be able to edit.

Exported units:

- `gsc_dsigned_ltx_writer` — holds a configuration under construction; signs and emits it.
  Built from the three signature-scheme parameters plus a callback that supplies the private
  key, so that the key never appears as an argument and can be assembled at the call site
  rather than stored.
- `gsc_dsigned_ltx_reader` — built from the same three parameters plus the public key;
  parses a buffer, checks the signature, and exposes the configuration only if it verified.
- `get_ltx` on both — access to the configuration itself.
