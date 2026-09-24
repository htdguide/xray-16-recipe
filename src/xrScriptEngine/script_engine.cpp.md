# src/xrScriptEngine/script_engine.cpp

> Owns the single Lua virtual machine: brings it up in a fixed order, maps a script file onto a
> namespace, routes every script error, and keeps the VM stack balanced across every crossing.

**Needs** — [`script_engine.hpp`](script_engine.hpp.md) · [`script_process.hpp`](script_process.hpp.md) · [`script_thread.hpp`](script_thread.hpp.md) · [`script_profiler.hpp`](script_profiler.hpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [`BindingsDumper.hpp`](BindingsDumper.hpp.md) · [`ScriptExporter.hpp`](ScriptExporter.hpp.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [`Include/xrAPI/xrAPI.h`](../Include/xrAPI/xrAPI.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator) · [Seam: Profiler and GPU debugging](../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)

**Used by** — [`script_engine.hpp`](script_engine.hpp.md)

**Tier floor** — T2: it owns a foreign interpreter's lifetime, hands it the engine allocator and
walks its value stack by index. Nothing here touches a device or a frozen byte layout, so the
only thing keeping it off T3 is that the interpreter it drives is a C-API guest with a manual
stack discipline and a collector the engine steps by hand.

## Purpose

This is the whole of criterion 10's machinery. Three existing games ship several hundred Lua
files that were written against a specific interpreter version, a specific way of turning a file
into a namespace, and a specific set of globals. This file reproduces that environment exactly:
it creates the interpreter, decides what standard library is visible, installs the compatibility
switches the shipped scripts need, invites every engine module to register its exported classes,
and then defines the two ways a script is reached — by name from C++, and by an undefined global
read from another script.

It is a separate file because it is the only place that may touch the interpreter's own
lifetime. Everything else in this module talks to the VM through the object it publishes.

## State

```text
RECORD ScriptEngine
  vm                : handle            # the interpreter; exactly one per engine instance
  current_thread    : optional<ScriptThread>   # the coroutine being resumed right now, or none
  processes         : map<ProcessorId, ScriptProcess>   # at most two: Level, Game
  baseline_stack    : int               # value-stack depth recorded once initialisation ends
  reload_modules    : bool              # one-shot: next file load re-runs even if already loaded
  last_missing_name : text              # single-entry negative cache of a script that is absent
  source_scratch    : bytes             # grown-only buffer holding prelude + file source
  transcript        : bytes             # every logged script line, flushed to a file on exit
  is_editor         : bool              # editor host: no profiler, different failure policy
  profiler          : optional<ScriptProfiler>
  debugger          : optional<ScriptDebugger>       # developer builds only
```

Invariants:

- `baseline_stack` is the depth the stack must be at whenever no engine→script call is in
  progress. `unload` restores it; every load and call path restores its own entry depth.
- `current_thread` is set only while a coroutine is being resumed, and is cleared on every exit
  path from the resume, including a thrown failure. A script that calls yield outside a thread
  is a hard error precisely because this field is empty.
- `last_missing_name` holds *one* name. It is not a cache in the general sense; it exists
  because the auto-load path is hit repeatedly with the same missing name inside a loop, and one
  slot removes the whole cost of that case without the bookkeeping of a real cache.

A process-wide registry maps an interpreter handle back to the engine that owns it:

```text
RECORD StateRegistry               # global, guarded by a lock, sized for 32 entries
  by_handle : map<handle, ScriptEngine>
```

This exists because the interpreter hands a raw state handle to every callback it invokes —
error handlers, hooks, the auto-load metamethod — and because each coroutine is its own handle.
Without the registry a callback cannot find its engine. A rebuild that can attach an owner
reference to the interpreter instance (most bindings can) deletes this registry entirely.

## `init`

**Contract** — Brings the VM from nothing to "ready to run shipped scripts". Takes the export
entry point (see [`ScriptExporter.cpp`](ScriptExporter.cpp.md)) and a flag saying whether to
load the global prelude script. Allocates; does not block; must run on the thread that will
subsequently drive the VM. Called again on every script-engine restart, which discards the old
VM completely.

**Invariants** — The order below is load-bearing in three places, marked `!`. On return the
value stack is empty apart from whatever the prelude left, and that depth becomes
`baseline_stack`.

```text
FUNCTION init(export_all, load_prelude)
  reinit()                                  # fresh VM, engine allocator, registry entry
  binding_layer.open(vm)                    # ! before anything registers a class

  # Compatibility switches read from the engine's own settings file, not the game's
  binding_layer.allow_nil_conversion(setting "lua_scripting/allow_nil_conversion", default true)
  binding_layer.disable_super_deprecation()
  vm.allow_escape_sequences(setting "lua_scripting/allow_escape_sequences", default false)

  binding_layer.enable_class_introspection(vm)
  install_error_callbacks()                 # ! before any script or registration can fail
  export_all(vm)                            # every module's registration, dependency-ordered
  IF launched with "-dump_bindings" AND not already dumped
    write the whole exported surface to a text file under the writable app root

  open_standard_library(vm)                 # the subset below, in this order
  open_engine_lua_patches(vm)               # interpreter-level fixes shipped with the engine
  open_profiler_zone_bindings(vm)           # optional external profiler, compiled out by default

  # Shipped scripts call random without ever seeding it
  run "math.randomseed(os.time())"
  run "math.random()" three times           # discard the first draws of a freshly seeded stream

  install_module_loader(vm)                 # require() resolves through the virtual filesystem
  IF jit is available AND not launched with "-nojit"
    open_jit_library(vm)

  install_auto_load_metatable(vm)           # ! after the globals table is fully populated
  IF developer build AND no external debugger attached
    install_line_and_call_hook(vm)

  IF load_prelude
    force reload; load the global prelude script into the global namespace
  baseline_stack = stack_depth(vm)
  redirect the interpreter's error stream into a fixed buffer the frame loop drains
```

**Notes**

- *The standard library subset.* Base, package, table, io, os, math and string are always
  opened. The bit and foreign-function libraries are opened only when the interpreter is the
  JIT variant. The debug library is opened only in non-shipping builds — shipped game scripts do
  not use it, and leaving it out of a release build removes the cheapest way to break the
  sandbox from a downloaded mod. There is no other sandboxing: a script has real file and
  process access through io and os, by design, because the shipped scripts use them.
- *Why the exports come before the libraries.* Registration writes into the globals table;
  opening the base library afterwards fills in missing names without replacing the table, so
  both survive. Reversing them is also survivable, but the auto-load metatable must come after
  both, or it fires during registration and tries to load a script for every name being defined.
- *The three randomised draws.* Discarding the first values of a freshly seeded stream is
  folklore rather than a property of any particular generator, but it is cheap and the shipped
  scripts do not compensate for it themselves.
- *The JIT is opt-out, not opt-in.* When it is disabled, the profiler's hook mode must also be
  disabled: the shipped prelude script installs its own hook when it detects no JIT, and the
  interaction between that hook and the binding layer loses the `super` global the shipped
  scripts use throughout.

## `reinit`

**Contract** — Destroys the current VM if any and creates a new one on the engine allocator.
Notifies the profiler on both edges so that a sampling profiler attached to the dying VM is
detached before the handle becomes invalid. Chooses which of the two namespace preludes is in
force for the rest of the run, from a command-line switch. Allocates a one-megabyte source
scratch buffer.

**Notes** — Every allocation the interpreter and the binding layer make is routed to the engine
allocator, so that script memory shows up in the engine's own accounting rather than the
system heap's.

## The namespace model

This is the part criterion 10 is most sensitive to, because every shipped script depends on it
and none of it is visible in the scripts themselves.

A script file named `foo` becomes a table named `foo` in the globals table. Its own top-level
declarations go into that table; its reads of undeclared names fall through to the globals
table. This is achieved by prefixing a *prelude* to the file's source before compiling it:

```text
FUNCTION build_prelude(namespace) -> text
  # namespace "a.b.c" becomes opener "a={b={c=" and closer "}}"
  opener, closer = split_on_dots(namespace)
  RETURN concat_without_newlines(
    'local function script_name() return "', namespace, '" end',
    'local this = {}',
    opener, ' this ', closer,              # publish the table at its dotted path, globally
    'setmetatable(this, {__index = _G})',  # unresolved reads fall through to the globals
    'setfenv(1, this)')                    # from here on the chunk's environment is `this`
```

**Invariants**

- The prelude contains no line break. The first line of the script file is therefore also the
  first line of the compiled chunk, and every error message and breakpoint line number matches
  the file as the modder sees it. A rebuild that injects a prelude on its own line shifts every
  reported line by one and breaks every debugger integration.
- `setfenv` comes last, so the table is published into the *globals* while the script body runs
  against `this`. Reversing those two statements publishes the table into itself.
- `script_name()` is a function, not a constant, and is the only reliable way a script learns
  its own namespace. Shipped scripts call it.

An alternate prelude, selected by a command-line switch, replaces the fall-through metatable
with an explicit `this._G = _G` binding. A script loaded under it must qualify global reads.
This exists to find accidental global reads in new script code; the shipped scripts require the
default.

The rebuild consequence is blunt: this needs a language-level "run this chunk with that table as
its global environment" facility. The shipped scripts assume the 5.1 semantics of exactly that,
which is why [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
fixes the language version rather than the implementation.

## `load_buffer`

**Contract** — Compiles a source buffer into the VM as a named chunk, optionally wrapped in the
namespace prelude. The chunk name is the file's logical path with a marker that tells the
interpreter to report it as a file rather than as a literal string. On a compile failure it
logs, runs the error path, and reports failure with nothing left on the stack. Grows the shared
source scratch buffer when prelude plus source exceed it; the buffer is never shrunk.

## `do_file` / `load_file_into_namespace`

**Contract** — Opens the script through the virtual filesystem (so a script inside an archive
loads exactly like a loose one), compiles it with the namespace prelude, and executes the
resulting chunk under a protected call. Returns whether it succeeded. On any failure path the
value stack is restored to the depth it had on entry.

```text
FUNCTION load_file_into_namespace(path, namespace) -> bool
  start = stack_depth(vm)
  reader = vfs.open(path)
  IF reader IS none
    log error "cannot open file"; RETURN false
  ok = load_buffer(vm, reader.bytes, "@" + path, namespace)
  close(reader)
  IF NOT ok
    stack_depth := start; RETURN false
  handler = debugger_present ? debugger.push_error_handler(vm) : none
  err = protected_call(vm, args: 0, results: 0, handler)
  IF debugger_present THEN debugger.pop_error_handler(vm, handler)
  IF err
    report(err); stack_depth := start; RETURN false
  ASSERT stack_depth(vm) == start
  RETURN true
```

**Notes** — The main chunk is run exactly once and is never called again, so compiling it to
native code is wasted work; the original leaves a note to that effect where it used to toggle
the JIT around this call. A rebuild that controls its own compilation tier should treat a
file's top level as interpret-once.

## `process_file_if_exists`

**Contract** — The single entry point for "make namespace `name` exist". Returns true if the
namespace is loaded (or already was), false if the script does not exist or failed. The
`warn_if_not_exist` flag distinguishes an explicit request (a missing file is an error worth
reporting) from a speculative one (an unknown global, where absence is normal).

```text
FUNCTION process_file_if_exists(name, warn_if_missing) -> bool
  IF NOT warn_if_missing AND name == last_missing_name
    RETURN false                        # the tight-loop case, answered without touching the vfs
  IF NOT reload_modules AND name is non-empty AND namespace_loaded(name)
    RETURN true
  path = vfs.resolve("$game_scripts$", name + ".script")
  IF NOT warn_if_missing AND NOT vfs.exists(path)
    last_missing_name = name
    RETURN false
  reload_modules = false                # the force-reload flag is one shot
  RETURN load_file_into_namespace(path, name IS empty ? GLOBAL : name)
```

**Notes** — The empty name means the global namespace, which is how the prelude script is
loaded at startup: it is the one script whose declarations land directly in the globals table
rather than in a table of its own.

## `auto_load` — reading an undefined global loads a script

**Contract** — Installed as the globals table's index metamethod. When a script reads a name
that is not present, the engine tries to load a script file of that name, then re-reads the
name without the metamethod. Returns nil when the argument shape is wrong or the script does
not exist.

```text
FUNCTION auto_load(table, key) -> value
  IF key is not a string OR table is not a table
    RETURN none
  engine = owner_of(current_vm)
  engine.process_file_if_exists(key, warn_if_missing: false)
  RETURN raw_read(table, key)          # raw: no second metamethod dispatch, no recursion
```

**Notes** — This is why the shipped scripts contain almost no explicit imports: writing
`xr_logic.something` anywhere loads `xr_logic.script` on first use. It also means a typo in a
global name costs a filesystem probe, which is what the single-entry negative cache is for.
The raw read is what terminates the recursion. A rebuild must preserve this behaviour verbatim;
it is observable from script in ways the shipped code relies on, including the deliberate
practice of naming a script after the table it is expected to provide.

## The module loader — `require` through the virtual filesystem

**Contract** — A loader inserted as the *second* searcher, after the preload table and before
the path-based one. It turns a dotted module name into a path under the game data root with a
script extension, opens it through the virtual filesystem, and returns the compiled chunk. When
the file is absent it returns a *message string* rather than failing, so that the remaining
searchers still get their turn. The game data root is also appended to the interpreter's own
search path as a fallback for loose files.

**Notes** — Ordering it before the path searcher is the whole point: a module packed inside a
game archive must win over nothing, and a loose file must win over an archived one — which it
does, because the virtual filesystem already resolves loose data ahead of archives. This is a
modern addition; the shipped scripts use the auto-load mechanism instead.

## Error routing — `install_error_callbacks`, and what happens at each call site

**Contract** — Installs four interception points into the binding layer and the interpreter.
Which of them exist depends on the build configuration, and the difference is load-bearing
because it changes whether a script error is recoverable.

```text
FUNCTION install_error_callbacks()
  IF an external debugger is attached
    hand the debugger its own hooks and stop          # it owns the failure path
  IF exceptions are disabled in the binding layer
    binding_layer.on_error      = report_and_die      # logs, dumps stack, terminates
    binding_layer.on_cast_failed = report_and_die     # wrong argument type at a bound call
  binding_layer.pcall_handler   = protected_call_failed
  vm.on_panic                   = report_and_die
```

- **Protected-call failure** (`protected_call_failed`) is the message handler pushed below every
  protected call the binding layer makes. It logs the message, prints the script call stack, and
  then — *only in a build that shows error dialogs* — asks the operator. "Retry" or "ignore"
  makes the call report success, and execution continues from the call site as if the script had
  returned normally. In a build that does not show dialogs the failure stands. This is the
  difference between a developer build, where a broken script is a nuisance, and a shipping
  build, where it is an error the player sees; a rebuild must decide it deliberately rather than
  inherit it.
- **Binding-layer error** is reached when the layer is compiled without exceptions, which is the
  shipping configuration. There is no recovery: the process reports and dies. With exceptions
  enabled the same condition becomes a throw that call sites catch — see
  [`script_callback_ex.h`](script_callback_ex.h.md).
- **Cast failure** — a script passed a value the bound signature cannot accept — is fatal in the
  no-exception build and a catchable failure otherwise.
- **Panic** — the interpreter's own unrecoverable state — is always fatal.

A special case: the message "cannot resume dead coroutine" is recognised and reported as
"do not return any values from main", because that is the only way a shipped script produces it.

## `print_stack` and `log_value` — the call-stack dump

**Contract** — Walks the VM's call stack from the innermost frame outward and logs one line per
frame: native frames as their name, script frames as source, current line, and either the
called name with its definition line or an anonymous `function <source:line>` form. When the
dump-depth console variable is above zero, each frame is followed by its locals, expanded
recursively to that depth.

```text
FUNCTION log_value(name, depth)
  value = top of stack
  SWITCH type of value
    nil, function, coroutine -> log type and name only
    number, boolean, string  -> log type, name and the formatted value (strings truncated)
    table                    -> IF depth <= dump_depth THEN expand ELSE log "[...]"
    userdata                 -> IF it carries a bound class, log the class name;
                                expand its environment table under the same depth rule
    otherwise                -> log "[not available]"
  IF expanding
    FOR EACH key, value IN the table
      SKIP native functions          # every bound method would otherwise flood the dump
      log_value(key, depth + 1)
```

**Invariants** — Every push made while walking is popped; the dump must not disturb the stack of
the failing call it is describing. The error-logging path is re-entrancy guarded, because
logging an error triggers a stack dump which logs errors.

**Notes** — Naming a userdata by its *bound* class rather than by the interpreter's generic type
is what makes the dump readable at all; it is the only place outside the bindings dumper that
reaches into the binding layer's own representation.

## `script_log` / `error_log` / `flush_log`

**Contract** — Every script-originated message is tagged with a kind (info, error, plain
message, and one per debug-hook event) and goes to two places: the engine log, and an in-memory
transcript that is written to `$logs$/<application>_<user>_lua.log` when the engine shuts down.
Only errors are logged unconditionally; everything else is gated on a console-settable flag.
Logging an error also dumps the call stack, once — the second, nested attempt is suppressed.

**Notes** — Each kind carries two distinct prefixes, one for the engine log and one for the
transcript, so that the transcript can be read on its own. The prefixes are part of what tools
built around the original parse, which is the only reason the exact strings matter.

## `namespace_loaded` and `object` — does this name exist, and is it the right kind

**Contract** — `namespace_loaded` walks a dotted name from the globals table down, following
only raw reads so that the auto-load metamethod does not fire, and reports whether every segment
resolved to a table. It optionally leaves the resolved table on the stack for a caller that is
about to look inside it. Finding a *non-table* at an intermediate segment is fatal: a namespace
name that collides with an ordinary value is a modding mistake that would otherwise produce
silent nonsense far away.

`object` scans a namespace table for a member of a given name and interpreter type, and reports
whether it exists. Both restore the stack depth they found.

## `function_object` / `functor` — resolving a script function from C++

**Contract** — Turns a dotted name such as `bind_stalker.on_death` into a callable object.
Splits at the *last* dot into namespace and function name; if the namespace is not the global
one, speculatively auto-loads it (using only its first segment, so that `a.b.f` loads `a`);
then verifies a value of function type exists there and returns it. Returns failure rather than
an invalid object when the name does not resolve — so that "is this callback defined" is a
question the engine can ask cheaply, which it does for every optional script callback.

## Process and thread ownership

**Contract** — The engine holds at most two named script processes, identified as *Level* and
*Game*, each created and destroyed with the thing it belongs to: the level process at level
load, the game process at match start. Registering a process twice under the same identifier is
an error. See [`script_process.cpp`](script_process.cpp.md) for what a process does with its
frame.

Coroutine creation goes through the engine so that the new coroutine's handle is registered
against this engine before anything can call back from it; destruction unregisters it and
releases the reference that kept it alive. A coroutine whose creation failed is destroyed
immediately rather than returned.

## `collect_all_garbage`

**Contract** — Runs a full collection twice. Once is not enough: the first pass runs finalizers,
which drop the last references to further objects, and only the second pass reclaims those.
Called at level transitions, where the cost is affordable and the reclaimed amount is large.

**Notes** — Per-frame collection is *stepped*, not full — see
[`script_process.cpp`](script_process.cpp.md). The engine choosing when the collector runs,
rather than letting it run on allocation pressure, is what keeps script memory out of the frame
budget's tail, and is why
[Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) demands
a manually steppable collector.

## `unload`

**Contract** — Truncates the value stack back to the recorded baseline and clears the
missing-name cache. This is the recovery used between levels: whatever a failed script left on
the stack is discarded without destroying the VM.

## `lua_hook_call`

**Contract** — The single per-line/per-call/per-return hook the engine installs. It finds the
owning engine from the state handle, forwards to the current coroutine's stack tracker in
developer builds, and forwards to the profiler when one is active. Installed only when no
external debugger has claimed the hook — the interpreter permits exactly one.

## `is_editor`

**Contract** — Reports whether this VM belongs to an editor host rather than the game. The
editor never gets a profiler, and exposes the answer to script so that shared scripts can branch.
