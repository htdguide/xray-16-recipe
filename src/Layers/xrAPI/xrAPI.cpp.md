# src/Layers/xrAPI/xrAPI.cpp

> Defines the one and only storage for the engine's global service environment — the record
> through which every module reaches every other module it was not linked against.

**Needs** — [`Include/xrAPI/xrAPI.h`](../../Include/xrAPI/xrAPI.h.md) · [`stdafx.h`](stdafx.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: holds nothing but references and one flag; what stops it going higher is
that all separately-loaded modules must resolve to the *same* instance, so the target's
module system has to support a process-wide mutable singleton shared across dynamic modules.

## Purpose

This is the definition site of the service environment. Everything else about the
environment — which slots exist, what they mean — is declared in the header; this file
exists only so that exactly one instance of the record is created, in exactly one module,
and every other module borrows it.

The file is a single declaration because that is the whole decision: *where does the one
copy live*. It lives in a module that depends on nothing (see
[`README.md`](README.md)), so every other module can depend on it without creating a cycle.
Splitting it out costs one build target and buys the removal of every link-time edge
between peer modules.

## State

The `*Port` names below are the interfaces defined in the chapters that fill each slot;
here they are opaque — the record knows nothing about them but their identity.

```text
# The process holds exactly one of these, zero-filled before any code runs.
RECORD EngineGlobalEnvironment
  Render            : optional<RendererPort>        # the frame-graph renderer
  DRender           : optional<DebugRendererPort>   # immediate debug primitives, debug builds only
  DU                : optional<DrawUtilityPort>     # shape/text helpers over the debug renderer
  UIRender          : optional<UIRendererPort>      # 2D quad/texture submission for the widget layer
  RenderFactory     : optional<RenderFactoryPort>   # creates renderer-owned companions for engine objects
  ScriptEngine      : optional<ScriptEnginePort>    # the Lua virtual machine and its bindings
  AISpace           : optional<AISpacePort>         # navigation graphs and path search
  Sound             : optional<SoundManagerPort>    # the 3D audio mixer
  UI                : optional<UICorePort>          # widget toolkit singletons: fonts, cursor, focus, scissor stack
  isDedicatedServer : bool                          # true when the process renders nothing and only simulates
```

Invariants that hold across the whole process, none of them checked by the record itself:

- Every slot reads as `none` (and `isDedicatedServer` as false) from the first instruction
  of the process until its filler runs. The zero-fill is the only initialization; there is
  no constructor and no initialization-order hazard, which is exactly why the record is a
  plain aggregate rather than an object with behaviour.
- A slot is written by exactly one filler and read by everyone. No reader ever writes.
- The pointed-to services outlive the slot: a slot is cleared before, or in the same step
  as, the destruction of the service it names. Where that ordering is violated the reader
  sees a dangling reference, not `none` — see the teardown notes in
  [`README.md`](README.md).

## `GEnv`

**Contract** — the process-wide service environment. Readable from any module and any
thread at any time; reading a slot before its filler has run yields `none`, and almost no
reader checks, so the engine's actual behaviour on a too-early read is an immediate crash
at the call site rather than a diagnostic. Writing is reserved to the ten filler sites
enumerated in [`README.md`](README.md). No locking: publication of a slot filled on a
worker thread is carried by the join with that worker, never by the record.

**Invariants** — one instance per process, shared by every dynamically loaded module.
Zero-valued before startup, and partially cleared during shutdown in a fixed order.

**Notes** — the record is exported from its module and imported by every other, which in a
rebuild is the ordinary question "how do dynamically loaded plug-ins see the host's
registry"; the answer must guarantee a single instance, because two copies of this record
is the failure mode that produces a renderer that is present in one module and absent in
the next.

The declaring header names four types that no slot uses — a material library, a concrete
renderer class, a UI-render token type. They are leftovers from slots that were removed;
nothing reads them and a rebuild should not reproduce them.
