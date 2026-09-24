# src/xrEngine/XR_IOConsole_script.cpp

> The console as scripts see it: run a line, read a setting, show or hide, and defer execution to a safe moment.

**Needs** — [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) · [`EventAPI.h`](EventAPI.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a binding declaration

## Purpose

The shipped game's scripts drive the console: they set graphics options from the options
menu, read the current value back to populate a control, and execute settings files. **This
surface is frozen** — the exported names and signatures are part of the criterion that every
shipped script runs unmodified (see the system requirements, §6, criterion 10).

It is a separate file because binding declarations are compiled against the script binding
seam and are worth isolating from the console's own code.

## State

`Stateless.`

## The exported surface

**Contract** — a free function returns the single console instance; the console type exports
nine methods. Reads discard the bounds, because script has no use for them. Execution from
script is immediate except for the deferred form.

```text
FREE FUNCTION get_console()               -> Console
FREE FUNCTION renderer_allow_override()   -> bool

CLASS CConsole
  execute(line)                 # runs now, without recording in the history
  execute_script(filename)      # runs the configuration-load command on that file
  execute_deferred(line)        # queues the line as an event; runs between frames
  show() / hide()
  get_string(name)  -> text
  get_integer(name) -> int
  get_bool(name)    -> bool
  get_float(name)   -> real
  get_token(name)   -> text
```

**Notes** — `execute_deferred` is the one that carries a real decision. A script runs from
inside the frame's update phase, and many console commands reset the graphics device, unload
the level or shut the engine down — none of which may happen while the code that asked for
them is still on the stack. The deferred form copies the line onto the heap, posts it as an
event, and the console executes and frees it when the event queue drains. A rebuild must
keep the distinction even if its ownership handling differs: the two forms are not
interchangeable, and script authors choose between them.

`renderer_allow_override` is exported here rather than with the renderer because the options
menu asks it, and the options menu is a script talking to the console. It says whether the
renderer selection may be changed from what the startup negotiation settled on.

The bound reads drop the minimum and maximum the underlying command declared. Script that
wants to present a slider therefore has to know the range itself, which is why those ranges
are duplicated in the shipped configuration files — a duplication a rebuild could remove by
exporting the bounds.
