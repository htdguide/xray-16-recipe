# src/xrServerEntities/object_item_script.cpp

> Constructs entities whose classes are defined in script, and contains the failure so that a broken mod class does not take the process down.

**Needs** — [`object_item_script.h`](object_item_script.h.md) · [`object_factory.h`](object_factory.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

A script-declared entity class is constructed by calling into the script layer, which can
fail in ways an engine class cannot. This file is where that failure is bounded: a mod's
broken class must produce a diagnosable nothing, not a dead process — while a record that
constructs and then fails to configure itself is still fatal, because it is unusable.

## State

```text
RECORD ScriptRegistryEntry
  identifier     : ClassIdentifier
  script_name    : text
  make_client    : script callable -> ClientObject     # ownership transfers to the engine
  make_server    : script callable(section) -> ServerRecord
```

**Invariants** — both callables are resolved once, at registration; if a script reset
invalidates them the whole registry is destroyed rather than re-resolved (see
[`object_factory_inline.h`](object_factory_inline.h.md)).

## `client_object`

**Contract** — calls the client constructor. A script-side error is **caught and turned
into a failure to create**, answering nothing, because a mod class that throws must not
abort the engine. On success the object runs its post-construction step and is handed over.

## `server_object`

**Contract** — calls the server constructor with the configuration section name. Errors are
caught and logged with the section that provoked them — the section name is the only clue a
modder has about *which* spawn went wrong, so it is always in the message. On success the
record runs its post-construction step, and a record that fails that step is fatal (unlike a
script error, a failed initialization means the record is half-built and unusable).

```text
FUNCTION server_object(section : text) -> optional<ServerRecord>
  TRY
    r = make_server(section)
  ON script error e
    log "creating server object from section " + section + " failed: " + e
    RETURN none
  r = r.after_construction()
  IF r is none THEN FAIL WITH "script server record failed to initialize"
  RETURN r
```

**Notes** — the asymmetry is deliberate: *the script's* fault is recoverable and reported,
*the engine's* fault is not. The shipping build compiles the script binding layer without
exceptions (see the binding seam), so a rebuild must decide up front how a script error
crosses back — a returned error works as well as a thrown one, and the contract above is
written in those terms.

**Ownership.** The object the script constructs is adopted by the engine at the boundary:
the script's garbage collector must stop considering it reachable, or a live entity gets
collected out from under the world. Whatever a rebuild's script layer calls this, the
transfer has to be explicit and it has to happen at exactly this call.
