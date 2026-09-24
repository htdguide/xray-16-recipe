# src/editors/xrSdkControls/Properties/AssemblyInfo.cs

> The control library's identity as a loadable module: name, vendor, version, and a declaration that it is not for foreign callers.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: build metadata, not code.

## Purpose

The editor is assembled from separately built modules loaded at run time, so each needs an identity the loader can match against. This declares the control library's.

## State

```text
RECORD ModuleIdentity
  name      : "SdkControls"
  vendor    : the original studio
  version   : 1.0.0.0            # fixed; never bumped in this repository's history
  foreign_callable : false       # not exposed to the platform's legacy object system
```

## Notes

Nothing here is load-bearing for behaviour, and the version is frozen at its initial value — the module is loaded by name from a known directory, not resolved by version, so nothing reads it. A rebuild supplies whatever identity its own module system requires and may discard the rest.

The declaration that the module's types are *not* reachable from the platform's legacy cross-language object system is the one line with a consequence: the editor host reaches this library only through the two exported entry points described in [`interfaces.hpp`](../../../Include/editor/interfaces.hpp.md), and nothing else is meant to bind to it.
