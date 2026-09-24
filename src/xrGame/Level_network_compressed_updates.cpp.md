# src/xrGame/Level_network_compressed_updates.cpp

> Unpacks a server update packet that carries several entity updates squeezed together under a shared dictionary, and uses its arrival time to decide how many physics steps the client owes.

**Needs** — [`Level.h`](Level.h.md) · [`xrServer_updates_compressor.h`](xrServer_updates_compressor.h.md) · [`xrCore/Compression/ppmd_compressor.h`](../xrCore/Compression/ppmd_compressor.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: decompression into a fixed-size stack buffer whose overrun is the failure mode, with pre-trained dictionaries held as raw byte ranges

## Purpose

Entity updates dominate a multiplayer server's outbound bandwidth, and they are extremely
repetitive: the same field layout, similar values, packet after packet. The server
therefore concatenates a batch of them and compresses the batch against a *dictionary* —
a fixed corpus of representative update bytes known to both sides — so the encoder can
reference material it never has to transmit. This file is the client half: it takes the
compressed batch apart and feeds each recovered update to the object registry as if it had
arrived uncompressed.

The second thing it does is only here because it is where a server update's timestamp is
first available: it converts "how stale is this update" into "how many physics steps must
the client run to catch up".

## State

```text
RECORD ClientCompressionContext      # lives on the level
  trained_stream    : opaque         # the pre-trained arithmetic model, shared by all packets
  lzo_working_memory: bytes          # scratch the decompressor needs; sized once
  lzo_working_buffer: bytes          # the allocation the scratch was carved from
  lzo_dictionary    : bytes          # the shared corpus; must be byte-identical on both ends
```

**Invariants** — both ends must hold the *same* dictionary and the *same* trained model, or
decompression produces garbage rather than an error. Nothing on the wire identifies which
dictionary was used; it is fixed by the build.

## `ProcessCompressedUpdate`

**Contract** — given a received packet and the flag set saying which compressor produced it,
recovers every update inside and applies it. Consumes the whole packet. Fails hard on an
unknown compression kind. Writes to a single reused decompression buffer, so it is not
reentrant and not thread-safe. On a client it also advances the server-time estimate and
sets the physics catch-up step count.

**Invariants** — the decompressed size must not exceed the destination buffer. The packet
is attacker-controlled, so this is the file's one real safety boundary; a rebuild must make
it a checked failure, not an assertion compiled out of the shipping build.

```text
FUNCTION ProcessCompressedUpdate(packet, compress_kind)
  # The batch is a sequence of length-prefixed compressed blobs, terminated by a zero
  # length. Each blob decompresses to exactly one entity update.
  read next_size : int (16-bit)
  WHILE next_size != 0
    IF compress_kind has PPMD THEN
      REQUIRE trained model exists
      out = ppmd_decompress(packet bytes at cursor, next_size, trained model)
    ELSE IF compress_kind has LZO THEN
      REQUIRE dictionary exists
      out = lzo_decompress_with_dictionary(packet bytes at cursor, next_size,
                                           working memory, dictionary)
    ELSE
      FAIL WITH unknown compression kind
    END IF
    REQUIRE size of out <= decompression buffer capacity
    advance packet cursor by next_size
    apply out to the object registry as a net-update import
    read next_size
  END WHILE

  IF running as a client THEN update the server-time estimate from the server clock

  # Decide how far behind this update leaves the client's physics.
  ping = client statistics ping
  IF (server_time + ping) < packet.time_received THEN
    lag = ping                       # the packet is from the future: trust the ping alone
  ELSE
    lag = server_time - packet.time_received + ping
  END IF
  physics_catch_up_steps = physics world's step count for lag
```

**Notes** — the two compressors are alternatives, not layers, and the choice is the server's
per-packet decision carried in a flag byte. Two exist because they trade differently: the
statistical coder compresses harder and costs more per packet, the dictionary coder is
nearly free and compresses less. A rebuild is free to ship one, but must then reject
packets marked with the other rather than misdecode them.

The clock comparison reads as defensive but encodes a real case: the client's estimate of
the server's clock can run *behind* the stamps it receives, in which case the difference is
negative and meaningless, so the ping alone is used as the lag estimate.

Converting lag into a *step count* rather than a time is how the fixed-timestep guarantee
survives network jitter: the client never runs a variable-length physics step, it runs the
right number of fixed ones.

## `init_compression` · `deinit_compression`

**Contract** — allocate and release the trained model, the working scratch and the shared
dictionary. Called once around a session. Idempotent on the release side only in the sense
that it checks before releasing.

**Notes** — the engine has no negotiation for this: a client that cannot decompress simply
fails at the first compressed packet. A rebuild should advertise the compressors it
supports at connect time and let the server either restrict itself to what the client
understands or refuse the connection with a readable reason.
