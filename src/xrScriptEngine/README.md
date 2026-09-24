# src/xrScriptEngine — ownership of the Lua virtual machine

Chapter 10 of [the build order](../../SYSTEM-REQUIREMENTS.md#7-build-order).

This module owns the one script interpreter the process runs, and everything that surrounds it:
how it is created and what it can see, how a script file becomes a namespace, how two hundred
and fifty engine classes end up visible to script in the right order, what happens to a script
error at each of the places the engine calls in, and the instruments — a stack dump, a line
profiler, an external debugger, a dump of the entire exported surface — that let a modder find
out why a script did what it did. It does **not** contain the exported classes themselves; those
live with the classes, in the game and server-entity modules, and reach the VM through the
declaration form defined here.

**Conformance criterion 10 is won or lost in this directory.** *Every Lua script that ships with
the game must load and run unmodified* is the strictest line in
[the requirements](../../SYSTEM-REQUIREMENTS.md#6-conformance): it fixes the language version,
the whole exported surface with exact names and signatures, the callback set, and the order
callbacks fire in. Three of those four are decided here, and the fourth — the surface — is
*collected* here. Read this chapter more carefully than its size suggests.

## Where it sits

It rests on [`xrCore`](../xrCore/README.md) — the virtual filesystem, strings, containers, the
allocator, logging — and on nothing else in the engine. It reaches the rest of the engine only
through the global environment struct of [chapter 5](../Layers/xrAPI/README.md), and only from
three files, each of which is reaching for the *current* script engine rather than for a peer
module. Two external seams are load-bearing and are the reason the module is small: the
interpreter itself
([Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)) and
the layer that turns engine classes into script classes
([Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)). It is
also the first consumer of [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator) that
hands its allocator to a guest.

Everything above it in the build order — the engine loop, the AI layer, the game, the UI — calls
*into* script through this module and is called *back* from script through it. Nothing below it
knows script exists.

## The ideas, named once

These recur in every twin in the directory. They are set out here so the twins can be terse.

**One VM, many handles.** There is exactly one interpreter per engine instance, but every
coroutine is its own handle, and the guest hands a raw handle to every callback it invokes. A
process-wide registry maps handle back to owning engine. A rebuild whose binding can attach an
owner reference to the interpreter deletes the registry.

**A script file is a namespace, made by a prelude.** Loading `foo.script` does not merely run it:
its source is prefixed with a newline-free prelude that creates a fresh table, publishes it
globally at the script's dotted name, gives it a fall-through to the globals, and switches the
chunk's environment to it. Top-level declarations land in the table; undeclared reads fall
through. The prelude carries no line break so that reported line numbers match the file. The
shipped scripts depend on all of this, and on being able to ask for their own name. This is why
the seam fixes the *language version* — the 5.1 environment model is the mechanism.

**An undefined global loads a script.** The globals table carries an index metamethod that, on a
miss, tries to load a script of that name and then re-reads raw. This is why shipped scripts
contain almost no imports: naming `xr_logic.something` loads `xr_logic.script`. A one-entry
negative cache absorbs the repeated-miss-in-a-loop case. A modern `require` searcher resolving
through the virtual filesystem sits alongside it, inserted ahead of the path searcher so that a
script inside a game archive is found.

**Registration is declarative and the engine collects it.** A class that wants to be visible to
script says so *in its own body*: here is my registration procedure, here are the classes that
must be registered before me. Each declaration enrols a node in a process-wide chain before the
program starts; at VM initialisation the chain is topologically sorted and walked. Ordering is
not cosmetic — the binding layer must know a base before it is told something derives from it,
and getting it wrong produces a class whose inherited methods are silently missing. Cycles are
fatal. Roughly two hundred and fifty classes carry the declaration.

**Initialisation order is load-bearing in three places**: the binding layer opens before anything
registers; the error callbacks install before anything can fail; the auto-load metamethod
installs after the globals table is fully populated, or it fires during registration.

**Build configuration decides what a script error costs.** With exceptions in the binding layer,
a script error is a catchable failure a call site absorbs. Without them — the shipping
configuration — the layer's error path logs and terminates. Between those sits the protected-call
message handler, which in a build that shows error dialogs lets the operator *convert a failure
into a success* and continue. With an external debugger attached, a failure instead stops in the
editor and then terminates. Four policies; a rebuild must pick deliberately, because the choice
is visible to modders.

**The stack is balanced across every crossing.** Every entry point records the value-stack depth
on entry and restores it on every exit, success or failure; initialisation records a baseline
that recovery truncates to; the debugger's hooks assert balance on every line of a live game.
[The requirements list this among the invariants asserted at runtime](../../SYSTEM-REQUIREMENTS.md#6-conformance);
a rebuild whose guest has no exposed value stack gets it for free and should still keep the
equivalent — that a failed script leaves no residue behind.

**Script runs as coroutines, one per frame.** Long-lived script logic is a coroutine running
`<namespace>.main()`. A *process* owns a set of them — one process per loaded level, one per
multiplayer match — and resumes exactly one per frame, round-robin. A script's effective tick
rate is therefore the frame rate divided by the number of live coroutines in its process, and
the shipped scripts are tuned to that. The engine also steps the collector itself rather than
letting it run on allocation pressure, and reads the VM's live byte count to decide when.

**Failures at a callback do not reach the frame loop.** The type the engine's optional script
handlers are stored in reports a script error and returns a zero-valued result; a failure it
cannot name unbinds the handler so a broken callback fails once rather than every frame. The
same swallowing applies to script overrides of engine methods. This is why a modded game
misbehaves rather than crashes — and why a rebuild should return "no answer" instead of a
zero-valued result, which is strictly better and changes no shipped script.

**The exported surface is dumpable, and that is how criterion 10 is checked.** One command-line
switch writes every namespace, class, base, constant, method and property to a readable listing.
Diffing the original's listing against a rebuild's is the only mechanical check that exists for
the strictest criterion in the recipe. Build it early.

## What this module demands of the binding layer

Everything below is asked of
[Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) and is not
described here beyond the demand:

- class registration with bases, constructors, methods, properties, operators and named integer
  constants, declared where the class lives;
- overload resolution by argument type at call time;
- automatic conversion of engine types in both directions, with **nil accepted where a function
  is expected** (shipped scripts clear handlers with nil, and a conversion failure is fatal in
  the shipping build);
- calling a script function, and calling a method on a script object, from the engine;
- a wrapper mechanism so a script table can override selected engine methods, with the native
  implementation still reachable;
- four policy annotations visible in shipped script — adopt, return-reference-to, out-value,
  iterator;
- an error path compiled either to exceptions or to a callback, chosen at build time;
- and introspection: enumerate a namespace, a class's bases, its constants, its members, and
  render a function's signature. Without this last one there is no bindings dump and no way to
  identify a bound object in a stack trace.

Two compatibility switches are set at startup and matter: nil conversion is permitted, and the
layer's deprecation of the `super` name is disabled — the shipped scripts use `super` throughout.

## The twins

| File | Role |
|---|---|
| [`script_engine.cpp`](script_engine.cpp.md) | VM lifecycle, the namespace prelude, file loading, auto-load, error routing, the stack dump. The chapter's centre. |
| [`script_engine.hpp`](script_engine.hpp.md) | Its declaration surface, plus the message kinds and the two console-settable debug variables. |
| [`ScriptExporter.cpp`](ScriptExporter.cpp.md) | Collects every module's registration and topologically sorts it. |
| [`ScriptExporter.hpp`](ScriptExporter.hpp.md) | The declaration a class writes to export itself — the declarative pattern itself. |
| [`ScriptExportMacros.hpp`](ScriptExportMacros.hpp.md) | The declaration form for engine methods a script subclass may override, and the native fallback. |
| [`ScriptEngineScript.cpp`](ScriptEngineScript.cpp.md) | This module's own exports: logging, prefetch, bit operations, the profiler surface, the script stopwatch. |
| [`ScriptEngineScript.hpp`](ScriptEngineScript.hpp.md) | Its declaration, plus the two clock slots the game module fills. |
| [`script_process.cpp`](script_process.cpp.md) | A named group of coroutines, one resumed per frame; the per-frame collector step. |
| [`script_process.hpp`](script_process.hpp.md) | Its declaration surface. |
| [`script_thread.cpp`](script_thread.cpp.md) | One coroutine: created from a namespace or console text, resumed, retired. |
| [`script_thread.hpp`](script_thread.hpp.md) | Its declaration surface. |
| [`script_stack_tracker.cpp`](script_stack_tracker.cpp.md) | A shadow call stack per coroutine, so a failed coroutine can still be printed. |
| [`script_stack_tracker.hpp`](script_stack_tracker.hpp.md) | Its declaration surface; fixes the 256-frame ceiling. |
| [`script_callback_ex.h`](script_callback_ex.h.md) | The type an optional script handler is stored in, and what a failure inside one costs. |
| [`Functor.hpp`](Functor.hpp.md) | A typed handle to a script function, and the nil-accepting conversion rule. |
| [`script_profiler.cpp`](script_profiler.cpp.md) | Hook-mode call-edge accounting and JIT-sampling capture, with reports to log and file. |
| [`script_profiler.hpp`](script_profiler.hpp.md) | Its declaration surface and the constants that bound it. |
| [`script_profiler_portions.hpp`](script_profiler_portions.hpp.md) | The two measurement records, and the recursion rule that keeps durations honest. |
| [`BindingsDumper.cpp`](BindingsDumper.cpp.md) | Writes the entire script-visible surface as a readable, diffable listing. |
| [`BindingsDumper.hpp`](BindingsDumper.hpp.md) | Its declaration surface and options. |
| [`script_debugger.cpp`](script_debugger.cpp.md) | The engine side of an external source-level debugger: breakpoints, stepping, the stop protocol. |
| [`script_debugger.hpp`](script_debugger.hpp.md) | Its declaration surface and the step modes. |
| [`script_debugger_messages.hpp`](script_debugger_messages.hpp.md) | The debugger wire vocabulary: the message set and three fixed-size records. |
| [`script_debugger_threads.cpp`](script_debugger_threads.cpp.md) | Snapshots the live coroutines into the editor's thread list. |
| [`script_debugger_threads.hpp`](script_debugger_threads.hpp.md) | Its declaration surface. |
| [`script_lua_helper.cpp`](script_lua_helper.cpp.md) | Everything the debugger does that touches the VM: hooks, traces, variable panes, watch evaluation. |
| [`script_lua_helper.hpp`](script_lua_helper.hpp.md) | Its declaration surface. |
| [`script_callStack.cpp`](script_callStack.cpp.md) | The debugger's frame selection and source navigation. |
| [`script_callStack.hpp`](script_callStack.hpp.md) | Its declaration surface. |
| [`mslotutils.h`](mslotutils.h.md) | The local datagram channel the debugger speaks over, and its fixed message buffer. |
| [`script_space.hpp`](script_space.hpp.md) | The single admission point for the interpreter and the binding layer — and the list of what is demanded of them. |
| [`script_space_forward.hpp`](script_space_forward.hpp.md) | The three binding-layer names an engine header may mention for free. |
| [`xrScriptEngine.hpp`](xrScriptEngine.hpp.md) | Module boundary marker and the two opaque interpreter types. |
| [`xrScriptEngine.cpp`](xrScriptEngine.cpp.md) | The element count of a script table, exported so consumers need not instantiate the iteration. |
| [`pch.hpp`](pch.hpp.md) | The module's common prelude — and the record that it rests on the core library and the script seams alone. |
| [`pch.cpp`](pch.cpp.md) | Build scaffolding; nothing. |

## Where a rebuilder should start

Bring up, in order: the VM with its allocator and the standard-library subset; the namespace
prelude and file loading through the virtual filesystem; the auto-load metamethod; the export
declaration and its sort; then the bindings dump, because from that point on every further class
you export is checkable against the original's listing. Coroutines, the profiler and the
debugger can all wait — none of them is on the path to criterion 10.

## What could not be recovered

- The engine ships a set of interpreter-level patches as a separate dependency, opened into the
  VM immediately after the standard library. That dependency is not present in the tree, so
  **what those patches change about the language is unknown** — and since they are opened before
  any game script runs, they are potentially part of what "the shipped scripts run unmodified"
  means. Two of their effects are visible from the call sites: a runtime switch controlling
  whether string escape sequences are honoured (off by default, settable in the engine's own
  settings file), and whatever the patch library's own open routine installs. A rebuilder must
  read that dependency before trusting this chapter to be complete.
- The relative order in which two *independent* export nodes run is not defined — the sort
  iterates an unordered container. Whether the original ever depended on a particular order
  between unrelated modules cannot be determined from the source; nothing enforces one.
- The per-frame collector step is guarded to developer builds. Whether that was a decision or an
  oversight is not recoverable; the reading here is that it is an oversight, because the
  shipping build is where an unscheduled collection is most expensive.
- The coroutine registry reference is *not* released on destruction in the shipped
  configuration, behind a switch recording that the binding layer of the era had defects with
  coroutines. Which defect, and whether it still exists in the project's fork of that layer, is
  not discoverable here.
- Run-to-cursor in the debugger compares the file for inequality, and the editor message that
  would enable the mode does not set it. Whether the feature ever worked is unknown.
- The debugger's global-variable pane iterates the globals and sends nothing; the emit is
  commented out. No reason is recorded.
- The exact log prefix strings for each script message kind are reproduced because external
  tooling parses them, but no such tool is in the tree and the claim is inferred from their
  shape.
- Three discarded draws after seeding the random generator, a one-megabyte initial source
  buffer, a 2048-byte debugger message buffer, a 256-frame shadow stack, a 12-and-10 frame
  traceback elision, and a 128-entry profiler report limit are all round numbers with no
  recorded derivation.
