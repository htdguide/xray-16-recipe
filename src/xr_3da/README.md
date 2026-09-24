# src/xr_3da — the executable

Chapter 27 of [the build order](../../SYSTEM-REQUIREMENTS.md#7-build-order). Four source
files, about a hundred lines. It is the smallest chapter in the recipe and one of the two
or three most worth reading carefully, because it is the only place where the whole system
is visible at once.

## What this module is responsible for

Everything above this chapter is a *module*: it declares what it needs, provides what it
promises, and deliberately does not know who else exists. Somebody has to know. This
directory is that somebody — the **composition root**. It answers four questions and has
no other purpose:

1. **Which graphics backends this binary offers, and in what order.** The engine owns the
   interface and the selection algorithm; it does not own the candidate list. That list
   lives here, so that producing a variant of the product — a GL-only build, a headless
   server, a backend under test — is an edit to one line in one file rather than a change
   inside the engine.
2. **Whether the game layer is attached at all.** A command-line switch detaches it. An
   engine with no game is a supported configuration, not a degraded one: it is how the
   core, the filesystem, the console and the renderer are exercised in isolation.
3. **What the command line is.** The host hands arguments over in a platform-specific
   shape; this chapter flattens them into the single string every consumer in the engine
   searches by substring.
4. **What happens when the process cannot be saved.** Specifically stack exhaustion, which
   the engine's own crash reporter cannot report because reporting needs stack.

It also carries the content that must exist before the virtual filesystem does: the
application icons, the splash image, and the fields of the failure dialog.

## Where it sits

It rests on everything. It links the core, the module-binding environment, the engine, at
least one renderer backend and the game, and it is the final target in the build. It is
depended upon by nothing.

It refers only to three headers — the platform vocabulary, the core, and the engine — plus
the two *module* interfaces: the renderer-module interface and the game-module interface,
each a handful of methods. That narrowness is the point. If the composition root can see
inside a module, it will start deciding things on that module's behalf and will stop being
readable.

## The load-bearing ideas

**A module interface, not a library.** The renderer and the game are each reached through a
tiny abstract surface — for the renderer: *what display modes do you support*, *are your
requirements met*, *install yourself into the global environment*, *remove yourself*; for
the game: *initialize and hand me your object factory*, *finalize*, *create the persistent
game state*, *destroy it*. Nothing else crosses. In the original these can be satisfied by
a statically linked object or by a dynamically loaded library, chosen at build time, and
the calling code cannot tell which. A rebuild should keep the interface and may drop the
dynamic-loading option entirely; nothing depends on it.

**Slot order is preference order, and slot zero carries an obligation.** The candidate list
is a fixed-length list of two with the platform-native backend first. An empty slot is
legal and skipped. A dedicated server, which renders nothing, still refuses to start unless
*slot zero* loads — because the renderer module is where the material and shader vocabulary
is registered, and the server must resolve the same material names its clients do. That is
the least obvious fact in the chapter and the easiest to break in a rebuild.

**The command line is a flat, unvalidated, unquoted string.** Switches are found by
substring search; a switch's value is read by scanning to the next space. Unknown arguments
are ignored rather than rejected, because shipped launchers and mod loaders append
arguments the engine has never heard of. A rebuild may parse properly, and must keep the
tolerance. It should also know what the flat-string design costs: a path containing a space
is indistinguishable from two arguments.

**Two things are addressed to somebody else's code entirely.** Two integers are exported
from the executable image under names chosen by two graphics-driver vendors; a laptop with
both an integrated and a discrete adapter reads them out of the binary before any of this
code runs and binds the process to the discrete adapter. No line in the repository reads
them. The mechanism is pure platform incident; the decision — *this application always
wants the high-performance adapter and never asks* — is load-bearing, and a rebuild whose
platform offers no equivalent silently halves its frame rate on hybrid machines.

**Some content cannot live in the game data.** Three icons (one per supported retail
title), the splash bitmap, and the failure dialog are embedded in the executable, because
they are needed either before the virtual filesystem is mounted or in order to report that
mounting it failed.

## The twins

| File | Role |
|---|---|
| [`entry_point.cpp`](entry_point.cpp.md) | The whole chapter: candidate list, game attachment, command-line flattening, the stack-exhaustion filter, the process entry |
| [`resource.h`](resource.h.md) | Identifiers for the icons, splash and pre-window failure dialog embedded in the binary |
| [`stdafx.h`](stdafx.h.md) | The statement of scope: the executable sees the platform vocabulary, the core, and the engine — nothing lower |
| [`stdafx.cpp`](stdafx.cpp.md) | A build-tool artifact with no content; delete it in a rebuild |

Four more files in the directory are not twinned because they are content rather than
source: the three icons and the splash bitmap. Two more are build description — a resource
script that binds the identifiers above to those files and stamps the version block, and a
manifest asserting that the process runs with the invoking user's privileges and declares
itself display-scaling aware. Both are platform packaging, not decisions.
