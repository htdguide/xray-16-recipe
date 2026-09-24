# src/xrCore/Compression/rt_compressor9.cpp

> The strong compressor, with the shared dictionary the multiplayer protocol compresses against.

**Needs** — [`rt_compressor.h`](rt_compressor.h.md) · [`../LocatorAPI.h`](../LocatorAPI.h.md) · [`../FS.h`](../FS.h.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`rt_compressor.h`](rt_compressor.h.md)
**Tier floor** — T1: per-thread scratch with an alignment requirement, and a process-wide dictionary buffer whose lifetime spans the session.

## Purpose

Maximum-effort compression, used where the same bytes are compressed once and consumed many times — principally the multiplayer server's outgoing state. Its distinguishing feature is the **shared dictionary**: a block of bytes both ends possess, which the compressor may match against as though it preceded the input.

Why that matters: a network message is a few hundred bytes and has almost no internal repetition to exploit, so plain compression on it barely pays. But messages across a session are *extremely* similar to one another. A dictionary built from representative traffic gives every small message a large history to match against.

## State

```text
dictionary      : optional<bytes>     # process-wide, loaded once
dictionary_size : int
scratch         : bytes per thread    # library-published size for the strong
                                      # level, which is much larger than for
                                      # the fast one
initialized     : bool                # guards the one-time load
```

**Invariants** — the dictionary is **optional**. If the data file is absent, both directions fall back to the dictionary-free form, and that is a *compatible* fallback only if both ends make the same choice. Two peers with different dictionary availability cannot talk: one compresses against a history the other does not have. Nothing in this file enforces that agreement.

## `rtc9_initialize`

**Contract** — on first call, initialize the library and attempt to load the dictionary from a fixed logical path — the multiplayer subdirectory of the configuration root, named for the compressor. Its presence is logged either way; its absence is not an error. Subsequent calls return immediately.

```text
FUNCTION initialize()
  IF already_initialized THEN RETURN
  library_init() OR FAIL
  path := resolve("$game_config$", "mp/lzo-dict.bin")
  IF exists(path) THEN
    r := open_for_read(path)
    dictionary := read_all(r)             # copied out; the reader is closed
    close(r)
    log "using dictionary"
  ELSE log "dictionary not found"
  initialized := true
```

**Notes** — the dictionary is copied out of the reader rather than kept as a pointer into it, because the reader would otherwise have to stay open for the session and, for an archived file, would hold a mapping alive. It is small enough that the copy is free.

Initialization is invoked lazily from both compress and decompress rather than being required up front. That makes the first compression on a cold process pay a file open; for the network path that happens during connection setup, where it is invisible.

## `rtc9_uninitialize`

**Contract** — release the dictionary. Does **not** clear the initialized flag, so a later compress will run dictionary-free rather than reloading. That asymmetry is almost certainly unintended; a rebuild should make teardown reversible.

## `rtc9_compress` / `rtc9_decompress`

**Contract** — one-shot compression at the maximum-effort level and its inverse, each selecting the dictionary-assisted or plain entry point depending on whether a dictionary is loaded. The destination capacity is passed in and the real length returned. Both ensure initialization first.

**Invariants** — the decompressor uses the **bounds-checked** dictionary variant when a dictionary is present and the **unchecked** plain variant when it is not. That inconsistency means a dictionary-free peer decompressing hostile input has no protection. A rebuild should use the checked form on both paths, since this is the network path.

## `rtc9_csize`

Contracted in [`rt_compressor.h`](rt_compressor.h.md): the worst-case expanded size, identical to the fast pair's.

## Could not recover

How the shipped dictionary file was generated, or what traffic it was trained on. Only the path it is read from survives.
