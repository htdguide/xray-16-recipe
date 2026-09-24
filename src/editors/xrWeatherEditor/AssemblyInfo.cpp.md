# src/editors/xrWeatherEditor/AssemblyInfo.cpp

> The editor library's identity as a loadable module.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: build metadata, not code.

## Purpose

Declares the name, vendor, copyright and version of the module the engine host loads at run time. The counterpart of [the control library's](../xrSdkControls/Properties/AssemblyInfo.cs.md).

## State

```text
RECORD ModuleIdentity
  name             : "xrWeatherEditor"
  vendor           : the original studio
  version          : 1.0.* — the build number is stamped at build time
  foreign_callable : false
  strictly_typed   : true        # the module promises a cross-language-safe public surface
```

## Notes

Two of these have a consequence and the rest do not.

The version is *not* frozen: it carries a wildcard, so every build stamps a different one. Nothing resolves the module by version — the host loads it by path — so this matters only in that two builds of the same source produce modules with different identities, which some module loaders treat as incompatible. A rebuild should pin it.

The declaration that the public surface is cross-language-safe is the one line that constrains the code: it forbids the library from exposing types whose spelling cannot be reproduced in another language of the same runtime. In practice it is satisfied because the library's public surface is [two functions](../../Include/editor/interfaces.hpp.md).
