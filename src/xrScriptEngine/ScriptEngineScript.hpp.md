# src/xrScriptEngine/ScriptEngineScript.hpp

> Declares the two clock hooks the game module fills in so scripts can ask the engine what time it is.

**Needs** — [`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md) · [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md)

**Used by** — [`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md)

**Tier floor** — T3: two function slots.

## Purpose

Declares the surface implemented in [`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md), plus
two mutable function slots that belong here rather than there.

## Exported units

- `script_time_global` — a slot holding "the game's global time in milliseconds", filled in by
  the game module at startup.
- `script_time_global_async` — the same reading taken without waiting for the simulation to
  reach a consistent point.

## Notes

These are slots, not functions, because the script engine is built before the game module and
must not depend on it; the game fills them when it loads. This is the same cycle-breaking
pattern as the global environment struct in chapter 5, at a much smaller scale, and a rebuild
with explicit dependency injection hands the engine a clock instead. **Nothing in this module
reads them** — the only readers are in the game module's own exports, which suggests they were
placed here so that both the game and the editor host could fill the same slot.
