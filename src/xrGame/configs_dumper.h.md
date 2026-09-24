# src/xrGame/configs_dumper.h

> Declares the background configuration dumper implemented in [`configs_dumper.cpp`](configs_dumper.cpp.md).

**Needs** — [`configs_dumper.cpp`](configs_dumper.cpp.md) · [`mp_config_sections.h`](mp_config_sections.h.md) · [`xrCore/Crypto/xr_dsa_signer.h`](../xrCore/Crypto/xr_dsa_signer.h.md) · [`xrEngine/ISheduled.h`](../xrEngine/ISheduled.h.md)
**Used by** — [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md) · [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`configs_dumper.cpp`](configs_dumper.cpp.md) · [`game_cl_mp.h`](game_cl_mp.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`configs_dumper.cpp`](configs_dumper.cpp.md). The
multiplayer client game owns one. It is a scheduler participant so that completion can be
noticed on the main thread without blocking.

Exported units:

- **`dump_config`** — start a dump; the callback receives the compressed buffer, its size
  and the uncompressed size.
- **`shedule_Update`** — the non-blocking completion poll.
- **`dump_signer`** — the signing context, built from the shared domain parameters with the
  private key written in.
- **the identity-section key names** — `config_dump_info`, and within it the player name,
  the player digest, the signature and the creation date. These are shared with the
  verifier and are frozen protocol.

**Notes** — the two signalling handles are named in platform-specific terms, and the whole
implementation is compiled on that platform only. See the implementation twin.
