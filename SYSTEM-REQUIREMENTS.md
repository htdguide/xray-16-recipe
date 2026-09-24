# System requirements

Recipe of **OpenXRay** (`xray-16`) at `a7055a4b9583d0111ea4026313e1b5a3828d7b4d`, 2026-09-22.

This is the preface. It states what the world must provide before a single line of the
rebuild is written: what the thing is, what language class it demands, which boundaries
belong to somebody else, what the machine must guarantee, which byte layouts are frozen,
and how a rebuilder knows they are finished.

---

## 1. What this builds

A first-person shooter engine that loads and runs three existing commercial games —
*S.T.A.L.K.E.R.: Shadow of Chernobyl*, *Clear Sky* and *Call of Pripyat* — from their
original, unmodified data files. It is a **drop-in replacement** for a proprietary engine
shipped in 2007: the player installs a retail copy of the game, drops this executable in
beside it, and the same levels, models, sounds, scripts, saves and configuration files
load and behave the same way.

The engine is a single process. It mounts the game's virtual filesystem from a chain of
compressed archives, brings up a window and a graphics device, loads a level's geometry
and collision database, spawns the entities recorded in that level's spawn file, and then
runs a fixed-rate simulation loop forever: read input, advance the server-side world by a
fixed timestep, run physics, run the AI scheduler, run Lua scripts, render a frame,
present. The same executable also runs as a dedicated multiplayer server with rendering
disabled, and as a client connected to one.

The people who run it are players (who see only a game), modders (who write Lua and
edit text configuration, and for whom stability of the script surface is the product),
and engine contributors.

---

## 2. Tier

A recipe is read without the tool that produced it, so the scale travels with the file.
The ladder describes what a repository *demands of its language*:

| Tier | Name | The repo demands | Typically |
|---|---|---|---|
| **T0** | Metal | No runtime, no allocator you did not write; bytes at fixed addresses; interrupts, MMIO, boot | asm, C, Zig, Rust `no_std` |
| **T1** | Manual | Deterministic destruction, explicit layout, hard latency or memory budgets, FFI as a first-class concern | C, C++, Rust, Zig |
| **T2** | Managed native | Compiled or JIT with a GC; throughput and data layout still matter; concurrency is explicit | Go, Java, C#, Swift, Kotlin |
| **T3** | Dynamic | Iteration speed over throughput; the library ecosystem carries most of the weight | Python, TypeScript, Ruby, Elixir |
| **T4** | Glue | Orchestration, config and text; spawning processes is the primary abstraction | Shell, Make, Nix, small Python |

T0 is the most demanding tier and T4 the most abstract. A repo belongs as close to T4 as
its hard requirements allow.

> **Tier: T1 (Manual).** Floor constraint: three requirements each independently forbid
> T2, and they compound.
>
> 1. **Frame budget with a hard tail.** The loop must present at 60 Hz while walking a
>    visibility tree, skinning several hundred animated meshes, stepping a rigid-body
>    world and running a Lua scheduler. Thousands of short-lived vectors, matrices and
>    collision results are produced and dropped every frame. A tier with unpredictable
>    collection pauses drops frames visibly; a tier that boxes small aggregates by default
>    turns a 64-byte matrix multiply into a pointer chase.
> 2. **Explicit layout across three foreign boundaries.** Vertex, index and constant
>    buffers are described to the graphics driver as raw byte strides and offsets; audio
>    buffers are handed to the audio device as raw PCM; the rigid-body library is handed
>    structs it will write back into. None of these boundaries accepts a layout the
>    language chose for itself.
> 3. **Frozen on-disk formats that are memory images.** Level geometry, collision trees
>    and animation data are loaded by pointing a struct at a mapped file region. A rebuild
>    may parse them field by field instead — see §5 — but must still reproduce the exact
>    byte layout, so the language must be able to describe it.
>
> Deterministic destruction is also load-bearing, but it is the *least* of the three:
> graphics, audio and script handles must be released in a defined order at a defined
> time, which several T2 languages can express with explicit scopes.
>
> T2 becomes viable if the renderer's inner loops and the collision database are pushed
> behind a native boundary and the frame's scratch allocations are pooled — which is
> most of the interesting code, so the honest reading is that the *game and AI layers*
> are comfortably T2 and the *device-facing layers* are not. The original is C++, and
> unlike many engines of its vintage, that was a requirement rather than a habit.

---

## 3. Seams

A seam is a boundary where this repository stops and somebody else's world starts.
Each is marked `given` (shop for it), `buildable` (the recipe can teach you) or
`pluggable` (the repo owns the interface and ships one filling).

### Seam: Windowing and input

**Verdict** — given

**Must provide**
- A window, fullscreen-capable and resizable, on Windows, Linux, macOS and the BSDs
- A graphics context attachable to that window: an OpenGL 4.1 core context, and on
  Windows also a Direct3D 11 swap chain bound to the same window handle. 4.1 is the floor
  the engine asks for and the version its shader prelude declares; everything above it is
  probed per-extension and used only where present, which is what lets one backend serve
  every supported platform
- Keyboard events with press and release distinguished and scancodes stable across
  keyboard layouts, since key bindings are stored by scancode
- Mouse motion delivered as relative deltas with the pointer captured, plus absolute
  position when the pointer is free for menus
- Gamepad axes and buttons with hot-plug notification
- A text-input mode that yields composed Unicode characters, separate from key events
- Clipboard read and write, message boxes before a window exists, and a display-mode
  enumeration (resolution, refresh rate, monitor index)
- Millisecond-or-better sleep, and the ability to raise the process timer resolution

**Surface used** — window lifecycle, event pump, GL context creation and swap, display
enumeration, clipboard, cursor capture, gamepad. Roughly forty entry points.

**Known-good substitutes** — SDL2 (what the original uses, and the widest platform reach);
SDL3; GLFW plus a separate gamepad and clipboard source. A platform-native path per OS is
possible but multiplies the port cost by the number of supported systems, of which there
are seven.

**If you build it** — not worth it. Nothing in the engine depends on the internals of the
windowing layer, and the per-OS work is measured in months.

### Seam: Graphics device

**Verdict** — pluggable

The repository owns this interface and ships two fillings: an OpenGL renderer used on
every platform, and a Direct3D 11 renderer used only on Windows. A rebuild must preserve
the seam itself — a renderer is selected at startup by name, and a failed device creation
falls back to the next candidate.

**Must provide** (what the interface demands of a filling)
- Immutable and dynamic buffers for vertices, indices and shader constants, with explicit
  byte strides, plus map/unmap of dynamic buffers with a discard hint
- Textures in 2D, 3D, cube and array forms; block-compressed formats (DXT1/3/5 and the
  BC4/BC5 pair) uploaded without decoding; render targets including depth and
  multiple-target binding; mip generation
- Vertex, pixel, geometry and compute shaders compiled from source at load time, with a
  compiled-blob cache keyed by source hash and macro set
- Explicit rasterizer, blend, depth-stencil and sampler state objects, set as units
- Occlusion queries, timestamp queries and a scissor rectangle
- Present with and without vertical sync, and a device-lost path that rebuilds every
  resource

**Surface used** — the renderer interface is wide because it is a *frame graph* boundary,
not a draw-call boundary: the engine hands it a scene and a camera and receives a
presented frame. See [`src/Include/xrRender/README.md`](src/Include/xrRender/README.md).

**Known-good substitutes** — Vulkan, Direct3D 12, Metal, WebGPU. Any of these satisfies
the capability list; the shader language differs, which matters because shader *sources*
ship with the game data (see §5).

### Seam: Audio device

**Verdict** — given

**Must provide**
- A 3D positional mixer: sources with position, velocity, direction, cone, distance
  attenuation and Doppler, against a movable listener
- At least 32 simultaneously audible sources, with the engine free to allocate and free
  them per frame
- Streaming playback: a source fed from a queue of PCM buffers the engine refills on a
  worker thread
- Enumeration of output devices, and selection of one by name
- Environmental reverb applied per-source with a wet/dry mix, driven by named presets
  (cave, hall, sewer and so on) attached to level regions
- Playback rate (pitch) and gain adjustable per source while playing

**Surface used** — device open and close, context, source allocate/free/play/stop/queue,
buffer create and fill, listener transform, per-source property set, reverb effect slot.

**Known-good substitutes** — OpenAL Soft (what the original uses, and the only common
implementation with the reverb extension the engine relies on); FMOD; Wwise; a
hand-rolled mixer feeding a raw audio output — but then reverb becomes yours to write.

### Seam: Audio and video codecs

**Verdict** — given

**Must provide**
- Vorbis decode from a seekable stream to interleaved PCM, with the ability to start
  decoding at an arbitrary sample offset (used for looped ambient beds and for streaming
  music) and to read the stream's sample rate, channel count and total length without
  decoding it
- Ogg container demuxing
- Theora video decode to a YUV plane set with presentation timestamps, for the intro and
  in-game screens

**Surface used** — open from callbacks, read PCM, seek, info query, close; the same for
video frames.

**Known-good substitutes** — libogg/libvorbis/libtheora (the original); ffmpeg's decoders;
any decoder with sample-accurate seek. Vorbis is not optional in practice: the game's
entire sound bank is Vorbis-encoded and must be read as shipped.

### Seam: Script virtual machine

**Verdict** — given

**Must provide**
- A Lua 5.1 interpreter — the *language version* matters, not the implementation: the
  game's shipped scripts use 5.1 semantics, including the 5.1 `setfenv`/`getfenv`
  environment model, integer-free number handling and the 5.1 coroutine rules
- The C API surface: stack manipulation, table get/set with and without metamethods,
  function registration, `pcall` with a custom error handler, userdata with metatables,
  registry, and the debug hook used for the line-level profiler
- Deterministic-enough garbage collection that the engine can step it manually per frame
  and inspect its byte count
- Optionally a JIT, with the ability to disable it per function (the original disables it
  for a few known-bad cases)

**Surface used** — a few dozen C API calls, concentrated behind the script engine module.

**Known-good substitutes** — LuaJIT (the default, selected by a build option); PUC Lua
5.1. Lua 5.4 will *not* load the shipped scripts unmodified.

### Seam: Script binding layer

**Verdict** — given

**Must provide**
- Registration of C++ classes into Lua with inheritance, constructors, properties,
  methods, operator overloads and enumerations, declared where the class lives rather
  than in one central table
- Automatic conversion of engine types (vectors, strings, handles) in both directions
- Function overloading resolved by argument type at call time
- An error path that can be compiled either to exceptions or to a callback, because the
  shipping build disables exceptions in this layer

**Surface used** — the declarative registration DSL, and the object/iterator wrappers.

**Known-good substitutes** — luabind (the original, with a project-local fork); sol2;
a hand-written binding generator. This seam is enormous in reach — roughly two hundred
classes are exported — so the cost of replacing it is in the call sites, not the library.

### Seam: Rigid-body dynamics

**Verdict** — given

**Must provide**
- Bodies with mass, inertia tensor, linear and angular velocity, damping and sleeping
- Joints: ball, hinge, hinge-2, slider, universal and fixed, each with low and high stops,
  spring and damping constants, and motors
- Collision shapes: sphere, box, capsule, cylinder and a user-supplied triangle-mesh
  collider that queries the engine's own collision database
- A stepping model that separates collision detection from the constraint solve, because
  the engine injects its own contacts between the two
- Contact parameters per contact point: friction (including anisotropic), bounce, soft
  constraint force mixing and error reduction
- A deterministic step: the same inputs must produce the same outputs, since the
  multiplayer server and client both step and compare

**Surface used** — world/space creation, body and joint lifecycle, per-step callbacks,
contact injection, and the custom-geometry hook.

**Known-good substitutes** — ODE (the original, vendored with local patches); Bullet;
Jolt; PhysX. The joint-stop and motor semantics are the part that transfers least
cleanly: vehicle and ragdoll tuning values in the game data are expressed in the
original library's units.

### Seam: Static collision database

**Verdict** — buildable

**Must provide**
- Build a tree over a triangle soup of a few hundred thousand triangles, once per level
  at load time, and serialize it
- Ray-versus-mesh, box-versus-mesh and frustum-versus-mesh queries returning all hits or
  the nearest, with a caller-supplied result budget
- Thread-safe concurrent queries against an immutable tree

**If you build it** — an AABB tree over triangle indices, built top-down by splitting on
the longest axis at the median, with leaves of one triangle; queries are the obvious
recursive descent with a slab test. This is a weekend's work and the recipe describes the
engine's wrapper around it in
[`src/xrCDB/README.md`](src/xrCDB/README.md). The original vendors a 2001-vintage library
for the tree itself and writes its own everything-else.

### Seam: Debug overlay UI

**Verdict** — given

**Must provide**
- An immediate-mode UI toolkit that renders through the engine's own graphics device:
  the engine supplies the vertex buffers and textures, the toolkit supplies the widget
  logic and layout
- Windows, docking, tables, plots, text input, and a font atlas the engine uploads

**Surface used** — per-frame new-frame/render bracket, the widget calls in debug tooling,
and the two backend halves (input translation, draw-list submission) which the engine
implements itself.

**Known-good substitutes** — Dear ImGui (the original). Any immediate-mode toolkit with a
renderer-agnostic draw-list output. This seam is debug-only and can be omitted entirely
in a shipping rebuild; the player-facing UI is the engine's own (see the UI chapters).

### Seam: Compression

**Verdict** — given

**Must provide**
- LZO1X decompression at high speed for the virtual filesystem's compressed archives,
  and its matching compressor for building them
- Raw DEFLATE decompression for a second archive variant and for network packet payloads
- A CRC32 that matches the standard polynomial, since stored checksums must verify

**Surface used** — one-shot decompress into a preallocated buffer, one-shot compress, and
the streaming inflate used by the network layer.

**Known-good substitutes** — LZO and zlib (the original). LZO's format is frozen by the
shipped archives and cannot be swapped; the DEFLATE side is standard.

### Seam: Image codecs

**Verdict** — given

**Must provide**
- DDS parsing to (format, dimensions, mip chain, raw blocks) without decoding block
  formats, since they are uploaded to the device as-is
- On the OpenGL path, the same for the KTX container, plus block-format conversion when
  the device lacks a format
- JPEG and PNG encode for screenshots; JPEG decode for a few loading screens

**Known-good substitutes** — gli (the original, for DDS/KTX); DirectXTex; a
hand-written DDS header parser, which is fifty lines and is the pragmatic choice.
libjpeg and stb_image_write cover the screenshot side.

### Seam: Multiplayer matchmaking and accounts

**Verdict** — given, and dead

**Must provide** — account login, buddy lists, server list retrieval and NAT negotiation,
statistics and persistent player profiles.

**Surface used** — the original integrates a vendor SDK across roughly thirty call sites,
plus a standalone profile server.

**Known-good substitutes** — none that are compatible: the vendor service was shut down in
2014, so the code paths exist but reach nothing. A rebuild should treat this as **an
optional module with a null implementation**, and if it wants working multiplayer,
design a fresh master-server protocol. The recipe documents what the game asks for, not
how the dead service answered.

### Seam: Networking transport

**Verdict** — given

**Must provide**
- A connection-oriented transport over UDP with optional per-channel reliability and
  ordering, connection establishment with a handshake, and disconnect notification
- Bandwidth throttling and a per-connection statistics readout (ping, loss, in/out rate)
- Support for a client count in the dozens on one server

**Surface used** — create host, connect, disconnect, send on channel with flags, poll
events, query statistics.

**Known-good substitutes** — ENet; GameNetworkingSockets; a raw UDP socket plus your own
sequencing. The original uses a Windows-only vendor library, which is precisely why
multiplayer is the least portable part of the engine; treat the transport as replaceable
and see [`src/xrNetServer/README.md`](src/xrNetServer/README.md) for the message
semantics that must survive the swap.

**Read this before planning multiplayer work.** The build ships the **null** filling as
the default on every platform — the real client and server are commented out of the build
description, and turning multiplayer on means editing it. So the shipped executable has
no working transport at all, and the conformance criteria below that exercise multiplayer
(13 and 14) cannot be met by the original either, without that edit. A rebuild that wants
multiplayer is not restoring a working feature; it is finishing one.

### Seam: Threads, atomics and process services

**Verdict** — given

**Must provide**
- OS threads with names and affinity, mutexes, condition variables, and a thread-local
  store
- Atomic integer and pointer operations with acquire/release ordering, and a
  compare-and-swap
- A monotonic high-resolution clock, and the CPU's cycle counter where available
- Process introspection: core count and topology, physical memory size, current working
  set, a backtrace at a crash, and a way to install a crash handler that runs before the
  process dies
- Dynamic library load, symbol lookup and unload — the renderer and the game module are
  loaded this way in non-static builds

**Known-good substitutes** — the C++ standard library plus a small per-OS shim; pthreads
directly; any modern language's own threading, which usually makes this seam vanish.

### Seam: Allocator

**Verdict** — pluggable

**Must provide** — sized allocation and free with 16-byte alignment, a realloc, and a
per-allocation size query. The engine routes every container and object through it and
can be built against three fillings: its own pooled allocator, the system allocator, or a
third-party one.

**Known-good substitutes** — mimalloc (a build option); the system allocator; the engine's
own small-block pool, which exists mainly to serve a 32-bit address space and matters far
less at 64 bits.

### Seam: Profiler and GPU debugging

**Verdict** — given, optional

**Must provide** — scoped CPU zone markers, frame markers, GPU timestamp zones, and a
capture trigger; plus GPU frame capture through a vendor tool.

**Known-good substitutes** — Tracy (a build option); Optick; nothing at all. A rebuild may
drop this seam entirely; it is compiled out by default.

---

## 4. Platform assumptions

**Operating systems** — Windows, Linux (including Android's kernel surface), macOS, and
the four BSDs, plus Haiku. The abstraction is drawn at compile time: one header per
platform family fills a common set of names. A rebuild in a language with a uniform
standard library will delete most of this layer.

**Architectures** — 64-bit is the supported configuration; 32-bit builds exist and change
the allocator's importance. x86-64, ARM32, ARM64, RISC-V, PowerPC 32/64 and E2K are all
reached. Consequences that matter:

- **Little-endian is assumed throughout.** On-disk structures are read as memory images,
  and no byte-swapping exists anywhere. A big-endian port must add it at every load site.
- **SSE2 intrinsics are the baseline** for the math layer; non-x86 targets get them
  through a header-only translation shim. A rebuild should express the math in terms of
  4-wide float operations and let the target choose.
- **Unaligned loads are performed** in the archive and network readers.

**Integer and float model** — 32-bit `int` and `float` are the working types; the vertex
and network formats depend on exact 32-bit float bit patterns. A few hash and random
routines rely on 32-bit unsigned wraparound. Doubles appear only in the physics solver.

**Filesystem** — paths are case-insensitively *matched* (the game data was authored on
Windows and references files with inconsistent case) but stored case-sensitively; the
engine lowercases and normalizes separators on every lookup. The path separator is
normalized to the platform's. Atomic rename is assumed for save files. A file can be
mapped into memory or read whole into a buffer; both paths exist and the recipe says
which each caller needs. Paths can exceed 260 bytes on non-Windows and must not.

**Clock** — a monotonic clock that does not step backwards is required by the simulation
loop; the loop uses a *fixed* timestep and accumulates real elapsed time against it, so a
clock that jumps forward causes a bounded catch-up burst rather than a divergence.

**Memory** — a level's working set is on the order of one to two gigabytes; the engine
assumes it can allocate a few hundred megabytes in one contiguous request for the level
geometry.

**Locale and encoding** — game text is stored in XML files in one of several single-byte
codepages depending on the localization, not UTF-8, and the engine translates to its
internal representation on load. String comparison in the resource system is ASCII
case-folding only; do not use a locale-aware fold, because it changes which files match.

**Network reachability** — nothing is required for single-player. Multiplayer assumes
UDP reachability and NAT traversal; see the matchmaking seam, which no longer resolves.

---

## 5. Data and persistence

The whole point of this engine is that it reads data it did not write. Everything in this
section is **frozen** unless marked otherwise.

**Virtual filesystem** — the engine mounts a list of archives plus loose directories into
one namespace, defined by a text file that maps logical roots (`$game_data$`,
`$level$`, `$app_data_root$` and about twenty more) to physical paths with flags for
recursion and writability. Archives are a header, a directory of (path, offset, real
size, compressed size) and a blob; entries are either stored, LZO-compressed or
DEFLATE-compressed, and four generations of the archive format are supported because four
generations of the game shipped. **Frozen.**

**Configuration** — an INI-like text format with sections, key/value lines, section
inheritance by a parent list on the section header, and include directives. It configures
literally everything: weapon damage, AI parameters, UI layout, graphics presets. Comments
and formatting need not round-trip, but parsing must accept everything the shipped files
contain, including duplicated keys and values with embedded commas that are parsed as
tuples. **Frozen (read side).**

**UI layout and text** — XML. Window trees with per-element geometry, textures, fonts and
handler names; and separate string tables keyed by identifier, per language. **Frozen.**

**Level data** — a directory of chunked binary files per level: static geometry with its
material and shader assignment, the collision model, the AI navigation mesh with its
node graph and coverage values, light maps, sector and portal topology for visibility,
particle and sound environment placement, and a *spawn* file listing every entity with
its class identifier and a class-specific payload. The chunked container is a recursive
(id, size, payload) format. **Frozen.**

**Models and animation** — a skinned-mesh format with per-bone hierarchy, skinning
weights, level-of-detail meshes, collision proxies and embedded material references, plus
a separate container for shared animation banks. Vertex layouts are enumerated by
identifier and each has a fixed byte layout. **Frozen.**

**Textures** — DDS with the game's own mip and format conventions, including a
detail-texture pairing and a bump-map channel packing that is specific to this engine and
must be reproduced exactly for the shipped art to look right. **Frozen.**

**Sounds** — Vorbis in Ogg, with an engine-specific sidecar carrying attenuation
distances, the AI-perception attributes of the sound, and loop points. **Frozen.**

**Shaders** — shader *source* ships with the game data, in two dialects (an assembly-like
pair of pixel/vertex programs for the oldest renderer, and high-level source for the
newer ones), together with a text file per material that names the passes, the blend and
depth state, the texture bindings and the sampler settings. A rebuild targeting a
different graphics API must either translate these at load time or ship replacements —
this is the single largest compatibility hazard in a port. **Frozen in form; a
translation layer is legitimate.**

**Save games** — a compressed chunked snapshot of every entity's state plus the script
layer's own serialized tables. Version-tagged; the engine refuses mismatched versions
rather than guessing. **Frozen only against itself** — a rebuild may define its own save
format, at the cost of not loading existing saves.

**Network protocol** — a bit-packed message format with quantized floats and angles,
sequenced per entity, with a client-side prediction and server reconciliation scheme.
**Frozen only against itself**, since both ends are this codebase; a rebuild is free to
redesign it and must then version-gate it.

**User settings** — the console's variable set, written back as the same INI-like text
format. **Internal, choose freely.**

---

## 6. Conformance

There is no automated test suite in this repository. That is a fact about the project and
a warning to the rebuilder: correctness here is established by *running the original game
data and comparing behaviour*. The acceptance criteria are therefore behavioural, and
they are ordered so that each one is reachable once its predecessors pass.

**Bring-up**

1. The virtual filesystem mounts a retail installation of *Call of Pripyat* and can list
   and read any file by logical path, including from all four archive generations.
2. The configuration parser reads the full shipped configuration set — several thousand
   sections across hundreds of files with multiple inheritance — and resolves every
   section's inherited keys identically to the original.
3. The console starts, accepts every shipped variable and command name, and writes back a
   settings file the original engine also accepts.

**Rendering**

4. A level's static geometry loads and renders with the shipped shaders and textures, with
   sector/portal visibility culling active; the frame is recognizably the same image as
   the original's.
5. Dynamic objects render with skeletal animation driven by the shipped animation banks,
   including level-of-detail switching.
6. The sky, weather cycle, sun position and post-processing produce the same
   time-of-day appearance as the original for a given clock value.

**Simulation**

7. A saved game from the original engine loads, and the world it restores matches:
   the same entities exist at the same positions with the same inventories.
8. Physics is *deterministic* — the same level, the same input sequence and the same seed
   produce the same trajectories across runs on one machine.
9. Ragdolls, vehicles and destructible objects behave without explosion, tunnelling or
   jitter under the shipped mass and joint parameters.

**AI and scripts**

10. Every Lua script that ships with the game loads and runs unmodified. This is the
    strictest criterion in the list: it fixes the language version, the entire exported
    class surface with exact names and signatures, the callback set, and the order in
    which callbacks fire.
11. The off-screen simulation (entities outside the loaded level, advanced at a coarse
    rate) runs, and entities cross the boundary into and out of the detailed simulation
    without losing state.
12. Non-player characters path through the navigation mesh, take cover, use the
    goal-and-plan layer, and hold conversations from the shipped dialogue tables.

**Multiplayer**

13. A dedicated server starts with rendering disabled, accepts clients, and runs a full
    match of each shipped game mode with consistent scores.
14. Client-side prediction and reconciliation keep a remote player's motion smooth at
    100 ms of simulated latency and 2% loss.

**Invariants asserted at runtime** — the code checks these continuously and a rebuild
should too: no entity is registered twice; a destroyed entity is unreferenced by the
scheduler, the render graph and the physics world before its memory is released; every
mounted archive's checksum verifies; the fixed-timestep accumulator never advances more
than a fixed number of steps in one frame; the script engine's Lua stack is balanced
across every call in and out.

**Performance** — the shipped engine targets 60 frames per second at 1080p on hardware of
the era with several hundred visible objects and a few dozen active non-player characters.
A rebuild that is correct but ten times slower has failed the only benchmark that exists.

---

## 7. Build order

Chapters in dependency order. Read them in this sequence; nothing here refers forward.

| # | Chapter | Role |
|---|---|---|
| 1 | [`src/Common`](src/Common/README.md) | Platform detection, compiler abstraction, the shared object-serialization vocabulary |
| 2 | [`src/xrCommon`](src/xrCommon/README.md) | Container and string aliases, small math helpers — the standard library this project decided to have |
| 3 | [`src/utils/xrMiscMath`](src/utils/xrMiscMath/README.md) | Vectors, matrices, quaternions, the 4-wide float layer |
| 4 | [`src/Include`](src/Include/README.md) | The interfaces the engine talks to the renderer, the UI and the editor through |
| 5 | [`src/Layers/xrAPI`](src/Layers/xrAPI/README.md) | The single global environment struct that binds the modules together at runtime |
| 6 | [`src/xrCore`](src/xrCore/README.md) | Virtual filesystem, archives, the configuration parser, strings, memory, threading, logging, the animation data model |
| 7 | [`src/xrCDB`](src/xrCDB/README.md) | Static collision database and its queries |
| 8 | [`src/xrMaterialSystem`](src/xrMaterialSystem/README.md) | Surface materials and the pairwise interaction table |
| 9 | [`src/xrNetServer`](src/xrNetServer/README.md) | Client/server transport, message framing, bit-packed streams |
| 10 | [`src/xrScriptEngine`](src/xrScriptEngine/README.md) | Lua virtual machine ownership, binding registration, script-side debugging |
| 11 | [`src/xrSound`](src/xrSound/README.md) | 3D audio: emitters, streaming, environment reverb, the AI-audible side of sound |
| 12 | [`src/xrParticles`](src/xrParticles/README.md) | Particle system simulation, independent of how it is drawn |
| 13 | [`src/xrEngine`](src/xrEngine/README.md) | The device, the frame loop, input, console, the object registry, the level lifecycle |
| 14 | [`src/xrAICore`](src/xrAICore/README.md) | Navigation graphs, path search, the goal/plan/action machinery |
| 15 | [`src/xrUICore`](src/xrUICore/README.md) | Widget toolkit: windows, layout, the XML-driven construction of screens |
| 16 | [`src/xrPhysics`](src/xrPhysics/README.md) | Rigid bodies, joints, ragdolls, vehicles, the collision bridge |
| 17 | [`src/xrGameSpy`](src/xrGameSpy/README.md) | The dead matchmaking integration, isolated behind its own module |
| 18 | [`src/Layers/xrRender`](src/Layers/xrRender/README.md) | Renderer core shared by every backend: scene graph, visibility, materials, models |
| 19 | [`src/Layers/xrRender_R2`](src/Layers/xrRender_R2/README.md) | The deferred-shading path shared by the modern backends |
| 20 | [`src/Layers/xrRenderDX11`](src/Layers/xrRenderDX11/README.md) · [`xrRenderPC_R4`](src/Layers/xrRenderPC_R4/README.md) | Direct3D 11 backend and its entry point |
| 21 | [`src/Layers/xrRenderGL`](src/Layers/xrRenderGL/README.md) · [`xrRenderPC_GL`](src/Layers/xrRenderPC_GL/README.md) | OpenGL backend and its entry point |
| 22 | [`src/xrServerEntities`](src/xrServerEntities/README.md) | The authoritative entity records — shared by the game and by the tools |
| 23 | [`src/xrGame`](src/xrGame/README.md) | The game itself: actors, weapons, inventory, the level, the alife simulation |
| 24 | [`src/xrGame/ai`](src/xrGame/ai/README.md) | Concrete creature brains built on chapter 14 |
| 25 | [`src/xrGame/ui`](src/xrGame/ui/README.md) | Game screens built on chapter 15 |
| 26 | [`src/xrGame/ik`](src/xrGame/ik/README.md) · [`CdkeyDecode`](src/xrGame/CdkeyDecode/README.md) · [`gamespy`](src/xrGame/gamespy/README.md) | Foot placement, key decoding, matchmaking glue |
| 27 | [`src/xr_3da`](src/xr_3da/README.md) | The executable: argument parsing, module selection, crash filter |
| 28 | [`src/utils`](src/utils/README.md) | Archive packer, multiplayer balance tooling, the profile server |
| 29 | [`src/editors`](src/editors/README.md) | The weather editor and its host engine — a separate application on the same core |

**Cycles.** Three exist and are facts about the design, not accidents:

- `xrEngine` ⇄ `xrPhysics`, `xrUICore`, `xrAICore`: the engine declares the interfaces and
  owns the frame loop; those modules link back against it to be driven. The edge is broken
  at the interface — in a rebuild, the engine depends on abstract ports and the modules
  provide adapters.
- `xrGame` ⇄ `xrServerEntities`: entity records are compiled into both the game and the
  tools with a different macro set. In a rebuild this is one shared data module with two
  consumers, not a cycle.
- The global environment struct in chapter 5 is a deliberate cycle-breaker: every module
  reaches its peers through one mutable global filled at startup. It is the engine's
  service locator. A rebuild should make it explicit dependency injection, and the recipe
  notes at each use site what is actually being reached for.
