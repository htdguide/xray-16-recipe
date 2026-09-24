# src/xrGame/ai_debug_variables.h

> Declares the AI's named-value debug scratch table.

**Needs** — [`ai_debug_variables.cpp`](ai_debug_variables.cpp.md)
**Used by** — [`ai_debug_variables.cpp`](ai_debug_variables.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`ai_debug_variables.cpp`](ai_debug_variables.cpp.md): `set_var` to publish a real under a
name, `get_var` in real, integer and truth-value forms to read one back, and `show_var` to
print one to the log. Grouped under their own namespace so the very generic names do not
collide.
