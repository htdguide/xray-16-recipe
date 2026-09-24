# src/xrGame/xrServer_updates_compressor.cpp

> Packs a frame's worth of per-entity updates into as few packets as possible: drop what has not changed, pile the rest into a buffer, compress the buffer, and split at the packet boundary.

**Needs** — [`xrServer_updates_compressor.h`](xrServer_updates_compressor.h.md) · [`Level.h`](Level.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrCore/Compression/ppmd_compressor.h`](../xrCore/Compression/ppmd_compressor.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrServer_updates_compressor.h`](xrServer_updates_compressor.h.md)
**Tier floor** — T1: compresses raw byte ranges between fixed-size buffers with hand-computed headroom

## Purpose

The world broadcast is the server's bandwidth budget, and it is almost all of it. Three
independent reductions are applied, in this order, and each is separately switchable at run time:

1. **Drop the unchanged.** An entity whose update is byte-identical to the last few is not sent.
2. **Batch.** Updates accumulate into one buffer rather than each becoming a packet.
3. **Compress.** The batch is compressed with a model trained on this game's own update traffic.

The third is the interesting one: a *trained* compressor, not a generic one. Update traffic is
extremely repetitive — the same field layouts, the same value ranges, frame after frame — and a
model trained offline on that traffic does far better than an adaptive one starting cold every
frame.

## State

See [`xrServer_updates_compressor.h`](xrServer_updates_compressor.h.md).

## The repeat cache

### `add_update`

**Contract** — record an entity's update and answer how many consecutive times it has been
identical. Finds the entity's slot, or evicts one. Compares the new update against the stored one
byte for byte; equal increments the counter, different resets it to zero. Always stores the new
update and stamps the time.

```text
FUNCTION add_update(entity_id, update) -> int
  slot := cache slot whose object_id == entity_id
  IF slot is none
    slot := most_expired_slot(now, update.length)
    IF slot is none THEN RETURN 0          # nothing evictable; deduplicate nothing
  slot.object_id := entity_id
  IF slot.payload == update                # exact byte comparison
    slot.equal_count := slot.equal_count + 1
  ELSE
    slot.equal_count := 0
  slot.update_time := now
  slot.payload := update
  RETURN slot.equal_count
```

**Invariants** — **the comparison is exact bytes, not a semantic equality.** Two updates that mean
the same thing but differ in a padding byte are "different", and that is the safe direction: a
false difference costs a packet, a false equality drops state.

### `search_most_expired`

**Contract** — choose a slot to evict: the one written longest ago, breaking ties toward the one
holding the *smaller* payload. Refuses to evict when the best candidate was written **this very
tick** and already holds at least as much data as the incoming update.

```text
FUNCTION most_expired(now, incoming_size) -> optional<slot>
  best := the slot with the smallest update_time
          tie-broken by the smaller stored payload
  IF best.update_time == now AND best.payload.length >= incoming_size
    RETURN none                 # every slot is fresh and this one is not worth displacing
  RETURN best
```

**Notes** — the refusal is the subtle part. All 32 slots being written in the same tick means the
frame has more than 32 updating entities, and evicting a fresh entry would thrash the cache into
uselessness. Preferring to evict the *smaller* payload is the same instinct: a large entry saves
more bandwidth when it does deduplicate, so it is worth keeping.

The cache is therefore **a bandwidth heuristic, not a correctness mechanism**. Dropping an update
is safe only because the receiver retains the last state it was given; an entity that genuinely
stops changing looks the same whether or not the server says so. A rebuild must guarantee that
retention before enabling this.

## The compressor

### `begin_updates`

**Contract** — start a frame's accumulation. Resets the output index to the first packet. **The
two modes differ in which buffer carries the message header**: with compression on, the output
packet begins with a compressed-batch header and a byte naming which optimizations are active,
and the accumulator starts bare; with compression off, the accumulator itself begins with a plain
batch header and the output packet starts bare.

**Invariants** — the active-optimization byte is sent in every compressed packet, so the receiver
needs no prior agreement about which codec is in use and the server may change codec between
frames.

### `write_update_for`

**Contract** — add one entity's update to the batch. Consults the repeat cache first and returns
without writing when the entity has been identical `max_eq_packets` times. Flushes the accumulator
first if this update would not fit with the headroom the batch's framing needs.

```text
FUNCTION write_update_for(entity_id, update)
  IF the drop-unchanged optimization is on
    IF cache.add_update(entity_id, update) >= 3 THEN RETURN

  reserved := 2 (message header) + 1 (optimization byte) + 2 (terminator)
  IF accumulator.used + update.length + reserved >= accumulator.capacity
    flush_accumulator()
  accumulator.append(update)
```

**Invariants** — **the reserved headroom is computed from the framing fields by name**, and it
must be: an accumulator filled to the byte leaves no room for the terminator and the batch cannot
be closed. The five bytes are the message type, the optimization byte, and the two-byte
zero-length terminator.

**Notes** — the cache is consulted with the *recording* call, not the read-only one, so asking
whether to drop an update also updates the cache. That coupling is why the read-only variant
exists and is unused: an earlier version asked first and recorded separately, and the two could
disagree.

### `flush_accumulator`

**Contract** — move the accumulated batch into the output. Uncompressed, the bytes are appended to
the current output packet and the next packet is started unconditionally. Compressed, the batch is
compressed into scratch and then appended **only if it fits** in the current packet's remaining
room; otherwise the current packet is terminated and the block goes into a fresh one.

```text
FUNCTION flush_accumulator()
  dst := current output packet
  IF compression is on
    compress accumulator into scratch, with whichever codec is selected
    IF dst.used + scratch.length + reserved < dst.capacity
      dst.append_u16(scratch.length)
      dst.append(scratch)
    ELSE
      dst.append_u16(0)                  # terminator: no more blocks in this packet
      dst := next output packet
      dst.append_u16(scratch.length)
      dst.append(scratch)
    accumulator.reset()
  ELSE
    dst.append(accumulator)
    next output packet
    accumulator.reset_with(BATCH header)
```

**Invariants** — **a compressed packet is a sequence of length-prefixed blocks ending in a
zero-length block.** Each block decompresses independently, which is what lets the server split at
a packet boundary without splitting a compressed stream. A rebuild must not use a codec whose
output is only decodable as one stream.

**Notes** — the uncompressed path starts a fresh output packet on every flush even when the
current one has room, so it produces more, smaller packets than the compressed path. That
asymmetry looks unintentional; nothing depends on it.

### `end_updates`

**Contract** — close the frame. Flushes any remaining accumulation — **the "remaining" test is
"more than two bytes used", which is "more than a bare header"** — writes the zero-length
terminator when compressing, and hands the caller the range of output packets to send.

**Invariants** — the returned range is the first packet through the current index **inclusive**, so
at least one packet is always returned even in an empty frame. The caller filters those out by
their small size; see [`xrServer.cpp`](xrServer.cpp.md).

**Notes** — there is also a debug path that appends every produced packet to a file with a
four-byte magic header, for offline analysis of update traffic. That capture is how the trained
model was produced in the first place, which makes it worth mentioning: **a rebuild that wants the
trained compressor needs this capture too, because the model must be trained on its own traffic.**
A model trained on the original's updates will not fit a rebuild whose field layouts differ.

### `init_compression` / `deinit_compression`

**Contract** — bring up the trained model and the dictionary-based codec, **only outside single
player**, and record in the availability mask which of the two actually came up. Either may fail —
a missing model file, a failed allocation — and failure is reported rather than fatal.

**Invariants** — the availability mask always has the drop-unchanged optimization set, because it
needs nothing external. The two codecs are set only if they initialized.

**Notes** — single player skips compression entirely because the broadcast never leaves the
process. That is the same short circuit as in [`xrServer.cpp`](xrServer.cpp.md) and it means the
compressor's whole machinery is dead weight in the shipped single-player game — allocated,
unused. A rebuild should not construct it at all.

The two codecs are alternatives, not a pipeline: the selection tests one and then the other, and
having both requested is a configuration error the code treats as "use the first". One is a
statistical model trained on this traffic; the other a dictionary-primed general compressor. The
first compresses better and costs more time, which is the trade a server operator is being given.
