# src/xrGame/xrServer_updates_compressor.h

> Declares the three independent tricks the server uses to shrink its per-frame world broadcast, and the buffers each needs.

**Needs** — [`xrServer_updates_compressor.cpp`](xrServer_updates_compressor.cpp.md) · [`traffic_optimization.h`](traffic_optimization.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`Level_network_compressed_updates.cpp`](Level_network_compressed_updates.cpp.md) · [`xrServer.h`](xrServer.h.md) · [`xrServer_updates_compressor.cpp`](xrServer_updates_compressor.cpp.md)
**Tier floor** — T1: fixed-size packet buffers, an aligned working buffer for a compressor, and a trained model held across the session

## Purpose

Declares the surface implemented in
[`xrServer_updates_compressor.cpp`](xrServer_updates_compressor.cpp.md). Two types.

### The repeat cache

A small fixed table remembering each entity's last update, used to *drop* an update identical to
recent ones.

```text
RECORD CachedUpdate
  object_id     : int (16-bit)
  equal_count   : int (16-bit)   # consecutive updates byte-identical to this one
  update_time   : int            # global ms of the last write
  payload       : bytes          # the update itself, for comparison

cache : array of 32 CachedUpdate     # fixed; entries are evicted, never grown
```

**Invariants** — the table is **32 entries, linearly searched**, and both facts are deliberate:
32 is roughly the number of entities that update every frame in a busy multiplayer match, and at
that size a linear scan beats any index. An entity not in the table is simply never deduplicated.

Entry zero's object identifier is indistinguishable from an unused slot, since an unused slot is
initialized with identifier zero. Zero is a valid entity identifier, so an entity with that
identifier collides with every empty slot. In practice the player's entity takes a low identifier
and the collision is benign, but it is a real latent defect.

### The compressor

```text
RECORD UpdatesCompressor
  accumulator     : packet buffer        # updates pile up here until it is nearly full
  compress_scratch: packet buffer        # one compressed block
  cache           : the repeat cache
  ready           : list<ref packet>     # the output packets, reused every frame
  current         : index into ready
  ppmd_model      : optional<trained model>      # session-long
  lzo_dictionary  : bytes                        # session-long
  lzo_workspace   : aligned bytes
```

**Invariants** — the output packet list is **allocated once and reused**, never freed between
frames; only the index resets. That is the whole reason the broadcast allocates nothing per frame.
The initial count is derived from a target total size divided by one packet's capacity, and the
list grows on demand if a frame needs more.

**Notes** — `max_eq_packets` is **3**: an entity whose update has been byte-identical three times
running stops being sent. The source's own comment explains the sizing constant beside it — an
actor's update is about 150 bytes — which is how the initial output-list size was chosen.

Three optimizations are declared, selectable independently by a runtime bitmask, and a second
bitmask records which are *available* — a compressor whose model failed to load advertises itself
as unavailable rather than failing. That pair of masks, requested versus available, is the pattern
a rebuild should copy for any optional codec.
