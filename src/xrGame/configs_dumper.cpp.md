# src/xrGame/configs_dumper.cpp

> Builds, signs and compresses an image of the configuration a multiplayer client is actually running with, on a background thread.

**Needs** — [`configs_dumper.h`](configs_dumper.h.md) · [`configs_common.h`](configs_common.h.md) · [`mp_config_sections.h`](mp_config_sections.h.md) · [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md) · [`Level.h`](Level.h.md) · [`GameObject.h`](GameObject.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`actor_mp_client.h`](actor_mp_client.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`xrCore/Compression/ppmd_compressor.h`](../xrCore/Compression/ppmd_compressor.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [`xrEngine/ISheduled.h`](../xrEngine/ISheduled.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`configs_dumper.h`](configs_dumper.h.md)
**Tier floor** — T1: a worker thread writing into a raw growable buffer that the main thread then reads, with an explicit compressor and a byte-exact signed region

## Purpose

The client half of the configuration anti-cheat. On request it produces one byte buffer
containing everything the server needs to decide whether this client is running the shipped
configuration: the relevant configuration sections, the *live* parameters of the player and
every weapon he is holding, an identity block naming the player, and a signature.

It does the work off the main thread because it is slow — it serializes a large part of the
configuration, hashes it and runs a context-modelling compressor over it — and the request
arrives during play. The main thread's only involvement is a poll for completion.

## State

```text
RECORD configs_dumper
  state           : { idle, active }
  dump_result     : growable byte buffer   # the uncompressed image being built
  compress_buffer : bytes                  # the compressed result; grown, never shrunk
  compress_size   : int                    # valid bytes in it
  compress_capacity : int
  signer          : the signing context
  ltx_configs     : the configuration-section enumerator
  active_params   : the live-parameter dumper
  complete_callback : function(compressed bytes, compressed size, uncompressed size)
  yield_callback  : optional<function(progress)>   # present only on a single-core machine
  start_signal, done_signal : two one-shot signals between the two threads
```

**Invariants**

- Exactly one dump at a time. A request while one is active is refused, not queued. The
  buffers are single-instance and the worker owns them for the duration.
- The worker thread is created on the *first* request and then parked, waiting for the start
  signal; subsequent requests reuse it. That avoids paying thread creation on a path the
  server can trigger repeatedly.
- The completion callback runs on the **main** thread, from the scheduler poll, never on the
  worker. Everything it touches — the network layer it hands the buffer to — is main-thread
  only.
- The compression buffer is grown to twice what is needed and never shrunk, so a sequence of
  dumps allocates at most once. The compressed output is assumed to fit in the uncompressed
  size, which holds for this data and this compressor but is not checked.

## `dump_config`

**Contract** — starts a dump, refusing if one is running. Decides whether the worker must
yield cooperatively, records the completion callback, wakes (or creates) the worker, and
registers with the scheduler so completion is polled.

```text
FUNCTION dump_config(complete_callback)
  IF state == active THEN RETURN        # refuse; a dump is already being built

  IF the process is confined to a single core THEN
    install a yield callback         # the worker must hand the core back, or the game stalls
  ELSE
    install none

  state = active
  IF the worker exists THEN
    signal start; register with the scheduler; RETURN

  create start and done signals
  SPAWN worker:
    LOOP
      AWAIT start
      IF state != active THEN BREAK
      write_configs()
      sign_configs()
      compress_configs()
      signal done
  register with the scheduler
```

**Invariants** — the single-core test is load-bearing, not an optimization. On one core the
worker and the render loop compete for the same hardware thread, and a compressor that never
yields freezes the game for the length of the dump. With a yield installed, the worker gives
up its slice at intervals and the game stays responsive at the cost of a slower dump.

**Notes** — the whole file is compiled only on the platform whose signalling primitives it
names, with a note in the source that it should be moved to the engine's own event
abstraction. So in the current tree **configuration verification is inactive on every other
platform**: a dump is never produced and the server's check never runs. A rebuild should use
a portable primitive and get the feature everywhere.

## completion polling

**Contract** — on each scheduler update, tests the done signal without blocking. When it is
set, hands the compressed buffer, its size and the uncompressed size to the completion
callback, returns to idle and unregisters from the scheduler. The uncompressed size travels
alongside because the receiver must allocate for it before decompressing.

## `write_configs`

**Contract** — the first stage. Clears the result buffer, then walks the configuration
sections that participate in verification, appending each; then appends a second document
holding the live parameters of each currently relevant object, numbered from one.

```text
FUNCTION write_configs()
  clear dump_result
  ltx_configs.start()
  WHILE ltx_configs.dump_one(INTO dump_result)
    IF yield_callback exists THEN yield_callback(progress)

  objects = collect_active_objects()
  params = an empty configuration image
  FOR EACH (i, object) IN objects
    active_params.dump(object, section named by (i+1), INTO params)
  append params to dump_result
```

**Invariants** — the section-at-a-time loop exists so the yield can be taken between
sections. Dumping the whole configuration in one call would give the yield nowhere to
happen.

**Invariants** — the objects are numbered from one in the order collected, and the server
reconstructs its own expectation by walking the same numbering. The *order* is therefore part
of the protocol, not just the contents.

## collecting the objects to dump

**Contract** — gathers the objects whose live parameters matter: the player's own controlled
entity, every weapon in the numbered equipment slots from the knife up to the grenade, and
the first cartridge of each weapon's magazine — deduplicated so that two weapons sharing an
ammunition type contribute one entry. Capped at sixteen objects, on a stack buffer, because
this runs on a worker thread on a path that must not allocate.

```text
FUNCTION collect_active_objects() -> list   # at most 16
  actor = the local controlled entity
  IF none THEN RETURN empty
  append actor
  FOR EACH slot FROM knife_slot TO grenade_slot
    IF full THEN RETURN what we have
    item = actor.inventory.item_in_slot(slot)
    IF item is not a weapon THEN CONTINUE
    append weapon
    IF the weapon's magazine is non-empty THEN
      cartridge = the first round in the magazine
      IF no entry already has that cartridge's section THEN append cartridge
  RETURN the list
```

**Notes** — only the *first* round is dumped. A magazine holding mixed ammunition reports
one type; the rest are unverified. That is a real hole and it is a deliberate cost trade —
checking every round would multiply the dump by the magazine size.

**Notes** — the sixteen-object cap silently truncates. With the shipped slot layout it cannot
be reached, but a rebuild changing the slot set must revisit it.

## `sign_configs`

**Contract** — the second stage, and the part whose byte-exactness the whole scheme rests
on. It appends an *identity string* to the image, signs everything up to and including it,
then **rewinds over the identity string** and writes an identity *section* in its place.

```text
FUNCTION sign_configs()
  identity = quote(player display name) THEN cdkey digest THEN current timestamp
                                        # "unknown_just_connected" / "null" when unavailable

  mark = dump_result.position
  append identity as a zero-terminated string

  signature = sign(dump_result CONTENTS, from the start to here)

  dump_result.position = mark            # discard the identity string from the image
  append a configuration section holding:
      the player name, the digest, the timestamp, and the signature
```

**Invariants** — the signed bytes and the transmitted bytes are **different**, and this is
the design. The server reconstructs the signed form by taking what it received, cutting it
at the identity section, and re-appending the identity string assembled from that section's
three fields in the same order. Any disagreement about that reconstruction — field order,
quoting, the terminator — makes every dump fail verification, so all of it is frozen
protocol. A rebuild is free to sign the transmitted bytes directly instead, and should, but
then both ends change together.

**Notes** — the player name is wrapped in double quotes before being used. That is not
cosmetic: the identity string is also written into a configuration section, whose format
would otherwise break on a name containing a separator.

**Notes** — the signer is primed at construction by signing four bytes of filler. This warms
the signing context's internal randomness so the first real signature does not pay the
initialization, and is the kind of thing a rebuild's library handles itself.

## `compress_configs`

**Contract** — the third stage: grows the output buffer to at least the image's size and
runs a context-modelling compressor over it, yielding to the main thread periodically when a
yield is installed. Configuration text compresses heavily, and the dump crosses the network.

**Notes** — the yield is taken on every fifth progress report rather than every one. Yielding
on each report costs more in context switches than it returns in responsiveness; five is a
tuned value with no derivation.

**Notes** — a yield is a "give up the rest of my slice, and if the platform says there was
nobody to give it to, sleep ten milliseconds". The sleep matters: on a single core with
nothing else runnable, a pure yield returns immediately and the loop spins.

## the signing key

**Contract** — the signer is constructed from the shared domain parameters in
[`configs_common.cpp`](configs_common.cpp.md) and then has its **private key written in as
twenty literal bytes**.

**Notes** — this is the scheme's central weakness and it must be stated plainly: every client
holds the private key, so any client can sign any dump. See
[`configs_common.cpp`](configs_common.cpp.md). The signature detects corruption and casual
interference; it does not authenticate. A rebuild wanting a real guarantee needs the server
to challenge the client, not the client to sign for itself.
