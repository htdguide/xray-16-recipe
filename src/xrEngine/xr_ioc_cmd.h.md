# src/xrEngine/xr_ioc_cmd.h

> The typed console-variable vocabulary: what a command is, and the seven kinds of bounded variable the engine binds names to.

**Needs** — [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md)
**Used by** — [`xrRender_console.cpp`](../Layers/xrRender/xrRender_console.cpp.md) · [`Engine.cpp`](Engine.cpp.md) · [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`XR_IOConsole_callback.cpp`](XR_IOConsole_callback.cpp.md) · [`XR_IOConsole_get.cpp`](XR_IOConsole_get.cpp.md) · [`XR_IOConsole_script.cpp`](XR_IOConsole_script.cpp.md) · [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md) · [`xr_level_controller.cpp`](xr_level_controller.cpp.md) · [`account_manager_console.h`](../xrGame/account_manager_console.h.md) · [`console_commands.cpp`](../xrGame/console_commands.cpp.md) · [`UIOptConCom.cpp`](../xrGame/ui/UIOptConCom.cpp.md)
**Tier floor** — T2: parsing, bounds and formatting; a variable binds to storage the rest of the engine reads directly.

## Purpose

The console is the engine's only configuration surface, and this file defines the *kinds* of
thing a console name can be. Each kind knows four things about itself: how to parse an
argument, how to print its current value, how to describe its accepted syntax, and how to
serialize itself back into the settings file. That fourth one is what makes the console a
persistence layer and not just a debug prompt.

Every variable **binds to storage owned elsewhere**. A command holds a reference to a
number, a flag bit or a text buffer that the rest of the engine reads directly, with no
indirection at the read site. The cost of that decision is that a variable's value can
change without anyone being told; the benefit is that reading a setting in an inner loop is
free. The engine relies on the second and has no mechanism for the first — there is no
change notification anywhere in this file, and the two commands that need one (video mode,
renderer) implement it by overriding the setter.

This file is substantive, not a declaration: the kinds are defined here in full and have no
implementation file. Only the registration of the engine's own command set lives in
[`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md).

## `Command` — what every console name is

**Contract** — a name, four behaviour flags, a recent-argument list, and five operations. A
command registers itself with the console by name and unregisters in its own destruction,
so a command's lifetime *is* its registration.

```text
RECORD Command
  name                  : text        # the frozen identifier; see note
  enabled               : bool        # a disabled command reports and does nothing
  lowercases_arguments  : bool        # fold the argument before executing
  handles_empty_argument: bool        # else an empty argument prints the value instead
  recent               : list<text>   # at most 10, oldest dropped, no duplicates

OPERATIONS
  execute(arguments)                  # REQUIRED of every kind
  status(out text)                    # the current value, as the settings file spells it
  info(out text)                      # what arguments are accepted, for the human
  save(writer)                        # write "<name> <status>" unless status is empty
  suggestions(out list, mode)         # completion candidates; default = the recent list
  remember_argument(argument)         # push onto the recent list
  invalid_syntax()                    # log the name and the accepted syntax
```

**Invariants** — the set of names is **frozen** (system requirements §6, criterion 3): the
shipped configuration files, the user's settings and every modification's scripts address
these by name and by argument syntax. A rebuild may change how a command is implemented and
may not rename one.

**Notes** — `save` writing nothing when the status is empty is how an *action* (quit, help,
restart the sound device) distinguishes itself from a *variable*: it simply has no status.
There is no separate action type; a command with nothing to print is one.

`handles_empty_argument` opts out of the console's "name with no argument prints the value"
rule. Actions set it, because an action given no argument should act.

The recent-argument list feeds completion. Duplicates are rejected and empty arguments are
never recorded — and neither is anything belonging to a command that handles empty
arguments, since an action has no arguments worth recalling.

## The kinds

Each kind is the same four answers about a different storage type. Bounds are declared at
registration and violating one logs the accepted range rather than clamping — **the value is
left unchanged**, which is the decision: a typo in a settings file leaves the previous value
in force rather than silently moving the setting to a limit.

```text
Mask(flags, bit)          # one bit of a shared flag word
    accepts "on"/"off"/"1"/"0"; status is "on"/"off"
    suggests the current value and the two choices

ToggleMask(flags, bit)    # the same bit, but the command flips it and logs the result
    takes no argument; handles the empty argument

Token(value, table)       # a named choice from a (name, id) table
    accepts a name, case-insensitively; status is the name of the current id, or "?"
    info is every name joined by "/", TRUNCATED WITH "..." when it would overflow
    suggests the current value first, then every name
    the table is obtained through an overridable accessor, so it may be built at run time

Float(value, min, max)    # a real in a closed range, with a tolerance at each end
    status is five decimal places with trailing zeros stripped — see note

Integer(value, min, max)  # an integer in a closed range, default [0, 999]

Vector3(value, min, max)  # three reals, per-component bounds
    accepts "x,y,z" or "(x,y,z)"

Vector4(value, min, max)  # four reals, otherwise identical

String(value, capacity)   # text, truncated to capacity; never lowercased
```

**Notes** — the range test on a real admits a small epsilon past each bound. Without it a
value written as five decimal places and read back as a binary float can fail its own
bounds check, which would make a settings file the engine wrote unloadable by the engine
that wrote it. **This is the load-bearing line in the whole file**: the console's write and
read paths must round-trip, and they only do because both the formatting (five places,
trailing zeros stripped) and the parsing (epsilon-tolerant) are chosen to make them.

The token table is fetched through an overridable accessor rather than held directly,
because three of the engine's token variables — the video mode list, the monitor list, the
audio device list — are not known until the corresponding hardware has been enumerated. The
accessor is how a table can be empty at registration and populated later.

Stripping trailing zeros from a real's status is cosmetic in the console and load-bearing in
the settings file: it is what keeps the file diffable across runs.

## `LoadCFG`

**Contract** — executes a file as a sequence of console lines. Declared here, implemented in
[`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md). Exposes one extension point: a predicate that decides
whether a given line may run.

## `LoadCFG_custom`

**Contract** — a `LoadCFG` whose predicate admits only lines that **begin with the command's
own name**. This is the mechanism by which a restricted configuration file can be executed:
a `LoadCFG_custom` registered as, say, the bindings loader will run only the binding lines
of a file and ignore everything else in it. It is how one file safely serves several
purposes, and how a file from an untrusted source can be executed for one narrow effect.

## The registration shorthand

**Contract** — registering a command means constructing one with a process-lifetime storage
duration and handing it to the console. The source does this through a family of macros
differing only in argument count; a rebuild writes one call.

**Notes** — the storage duration is the point: a command must outlive every execution of
it, and its destruction is what unregisters it. Constructing commands into a list the
console owns achieves the same thing and is what a rebuild should do — the macro family
exists only because C++ has no varargs constructor call.
