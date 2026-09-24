# OpenXRay — a recipe

This is a **recipe**: the [OpenXRay](https://github.com/OpenXRay/xray-16) repository with
the code taken out and the reasoning left in. The tree is the same; every source file has
been replaced by a markdown **twin** that says what that file decides, promises and
computes, in no language in particular.

Recipe of `xray-16` at commit `a7055a4b9583d0111ea4026313e1b5a3828d7b4d`, 2026-09-22.

Someone holding only this recipe should be able to rebuild the engine in a language it
never mentions. That is the test every page here was written against.

**4,151 twins · 132 chapter pages · ~2.9 million words.**

---

## What the thing is

A first-person shooter engine that loads and runs three existing commercial games —
*S.T.A.L.K.E.R.: Shadow of Chernobyl*, *Clear Sky* and *Call of Pripyat* — from their
original, unmodified data files. It is a drop-in replacement for a proprietary engine
shipped in 2007, and that single requirement explains most of what follows: dozens of
on-disk formats are frozen, roughly two hundred script classes are frozen by name, and
the console variable set is frozen because shipped configuration files address it.

---

## How to read this

Read it as a book, in build order — dependencies before dependents. Nothing refers
forward.

1. **[`SYSTEM-REQUIREMENTS.md`](SYSTEM-REQUIREMENTS.md)** is the preface, and the place to
   start. It states what the world must provide before a line is written: the language
   **tier** and the floor constraint that pins it there, the sixteen **seams** where this
   repository stops and somebody else's world begins, the platform assumptions, the frozen
   data formats, and the conformance criteria that tell a rebuilder when they are done.
2. **[`GLOSSARY.md`](GLOSSARY.md)** defines the project's own vocabulary once — *level*,
   *sector*, *portal*, *alife*, *online/offline*, *server object* vs *client object* vs
   *game object*, *feel*, *restrictor*, *smart terrain* — so the twins can stay terse.
   Read it before chapter 6; keep it open through chapter 23.
3. Then the chapters in the order below. Each directory's `README.md` opens a chapter and
   states the ideas its twins assume; each twin is a page.

### The shape of a twin

Every twin carries the same sections in the same order, whatever its size. That sameness
is the navigational aid: a reader who has read two knows where to look in the two
hundredth.

- a one-line summary, then **Needs** / **Used by** / **Tier floor** — the navigation the
  whole recipe hangs on, with `Used by` computed by inverting every `Needs` edge in the
  tree;
- **Purpose** — the job this file does and why it is a separate file at all;
- **State** — the data it owns as abstract records, with the invariants that must hold
  between fields. Those invariants are usually enforced by scattered code and written
  down nowhere else, which makes them the most valuable lines on the page;
- one section per exported unit: a **Contract** in prose, its **Invariants**, and
  **pseudocode** where there is an actual algorithm;
- **Notes**, where an artifact of C++ is translated into the problem it solves.

Pseudocode uses one dialect across the whole recipe, fenced as `text`: control flow in
caps, a fixed abstract type set (`int`, `real`, `text`, `bytes`, `list<T>`, `map<K,V>`,
`optional<T>`, `result<T,E>`, …), and a width or precision stated only where it is
load-bearing. No C++ appears inside a pseudocode block.

---

## Build order

| # | Chapter | Role |
|---|---|---|
| 1 | [`src/Common`](src/Common/README.md) | Platform and compiler abstraction; the shared object-serialization vocabulary the entity system rests on |
| 2 | [`src/xrCommon`](src/xrCommon/README.md) | The standard library this project decided to have: containers routed through one allocator, two text types |
| 3 | [`src/utils/xrMiscMath`](src/utils/xrMiscMath/README.md) | Vectors, matrices, quaternions; the transform conventions everything above obeys |
| 4 | [`src/Include`](src/Include/README.md) | The interfaces the engine talks to the renderer, the UI and the editor through — the smallest, densest chapter here |
| 5 | [`src/Layers/xrAPI`](src/Layers/xrAPI/README.md) | The one global environment struct binding the modules at runtime; the deliberate cycle-breaker |
| 6 | [`src/xrCore`](src/xrCore/README.md) | Virtual filesystem and its four archive generations, the configuration parser, strings, memory, threading, the chunked container, the animation data model |
| 7 | [`src/xrCDB`](src/xrCDB/README.md) | Static collision database and its queries — the one seam this recipe teaches you to *build* |
| 8 | [`src/xrMaterialSystem`](src/xrMaterialSystem/README.md) | Surface materials and the pairwise interaction table physics, sound, AI and weapons all read |
| 9 | [`src/xrNetServer`](src/xrNetServer/README.md) | Transport, message framing, the bit-packed stream and its quantization |
| 10 | [`src/xrScriptEngine`](src/xrScriptEngine/README.md) | Lua VM ownership, the declarative registration pattern, script-side debugging |
| 11 | [`src/xrSound`](src/xrSound/README.md) | 3D audio: emitter lifetime, voice allocation and its hysteresis, streaming, environmental reverb, occlusion |
| 12 | [`src/xrParticles`](src/xrParticles/README.md) | Particle simulation as an action list over a pool, independent of how it is drawn |
| 13 | [`src/xrEngine`](src/xrEngine/README.md) | **The spine.** Device, frame loop, input, console, object registry, the update scheduler and its budget, level lifecycle, weather |
| 14 | [`src/xrAICore`](src/xrAICore/README.md) | Navigation graphs and A\*, and the goal/plan/action planner — with no knowledge of any particular creature |
| 15 | [`src/xrUICore`](src/xrUICore/README.md) | The widget toolkit: window tree, event and focus model, XML-driven construction, text layout |
| 16 | [`src/xrPhysics`](src/xrPhysics/README.md) | Rigid bodies, joints, ragdolls, vehicles, the character controller, and the collision bridge |
| 17 | [`src/xrGameSpy`](src/xrGameSpy/README.md) | The matchmaking integration — documented as a contract, because the service it called is gone |
| 18 | [`src/Layers/xrRender`](src/Layers/xrRender/README.md) | Renderer core: visibility, the data-driven material/pass system, model types, textures, the draw stream |
| 19 | [`src/Layers/xrRender_R2`](src/Layers/xrRender_R2/README.md) | The deferred-shading frame graph shared by both modern backends |
| 20 | [`src/Layers/xrRenderDX11`](src/Layers/xrRenderDX11/README.md) · [`xrRenderPC_R4`](src/Layers/xrRenderPC_R4/README.md) | The Direct3D 11 filling of the graphics seam, and the module that registers it |
| 21 | [`src/Layers/xrRenderGL`](src/Layers/xrRenderGL/README.md) · [`xrRenderPC_GL`](src/Layers/xrRenderPC_GL/README.md) | The OpenGL filling — the portable one, and a continuous act of translation |
| 22 | [`src/xrServerEntities`](src/xrServerEntities/README.md) | The authoritative entity records: what an entity *is* on disk and on the wire |
| 23 | [`src/xrGame`](src/xrGame/README.md) | The game itself — actors, weapons, inventory, anomalies, vehicles, the level, the alife simulation, trade, dialogue |
| 24 | [`src/xrGame/ai`](src/xrGame/ai/README.md) | Concrete creature brains built on chapter 14 |
| 25 | [`src/xrGame/ui`](src/xrGame/ui/README.md) | The game's screens, built on chapter 15 |
| 26 | [`src/xrGame/ik`](src/xrGame/ik/README.md) · [`CdkeyDecode`](src/xrGame/CdkeyDecode/README.md) · [`gamespy`](src/xrGame/gamespy/README.md) | Foot placement, key decoding, matchmaking glue |
| 27 | [`src/xr_3da`](src/xr_3da/README.md) | The executable: the composition root |
| 28 | [`src/utils`](src/utils/README.md) | Archive packer (the *writing* side of the frozen format), multiplayer tooling, the profile server |
| 29 | [`src/editors`](src/editors/README.md) | The weather editor: a second application on the same core, and the tool that authors chapter 13's data |

### Cycles

Three exist, and they are facts about the design rather than accidents. `xrEngine` ⇄
`xrPhysics`/`xrUICore`/`xrAICore`: the engine declares the interfaces and owns the frame
loop, and those modules link back to be driven — break it at the interface. `xrGame` ⇄
`xrServerEntities`: one shared data module compiled into two consumers with different
macro sets, not truly a cycle. And chapter 5's global environment struct, which is the
service locator every module reaches its peers through; a rebuild should make it explicit
dependency injection, and each use site says what is really being reached for.

---

## What this recipe is honest about

A recipe that only described the design would be a worse artifact than one that also says
where the design and the code disagree. Writing 4,151 contracts turned up a large number
of places where the shipped behaviour is not what the code appears to intend. Each is
recorded **in the twin where it lives**, marked as a defect rather than transcribed as a
contract — so a rebuilder implements the intent, or makes an informed choice to reproduce
the bug for compatibility, instead of inheriting it blindly.

A representative sample, to show the kind of thing:

- **Transform composition applies its second argument first.** Argument order is reversed
  from the underlying product, nothing asserts it, and a rebuild that reads it the other
  way is silently and consistently wrong.
- **Weapon dispersion multipliers are swapped** in both stance branches, so shipped
  non-player characters are *more* accurate when not aiming; nearby, the "too far to kill"
  test returns false unconditionally and the recoil effector returns its input.
- **The relations system captures who provoked a fight correctly, then never uses it** —
  the line is commented out, so killing a neutral you provoked scores as killing an enemy.
- **The smart-cover containment test accepts a point inside any one of six face planes**
  rather than all six.
- **Several complete features never run**: a monster behaviour registered in a dispatch
  table and selected by no file; an entire creature state-tree that would not compile if
  anything instantiated it; a colour picker that is never opened; a tree filter never bound
  to its own filtering algorithm.
- **The weather editor prints numbers locale-independently and parses them
  locale-dependently**, so on a comma-decimal machine it refuses its own output.
- **Shipped test scaffolding is live behaviour**: one creature's "search for enemy" state
  *is* a developer cover-test harness, so it walks to an assigned cell and stands there.

Two further things a reader should know before planning work:

- **Multiplayer does not ship.** The build lists the *null* networking filling as the
  default on every platform; the real client and server are commented out. Conformance
  criteria 13 and 14 cannot be met by the original either, without a build edit. A rebuild
  that wants multiplayer is finishing a feature, not restoring one.
- **There is no automated test suite.** Correctness here was established by running the
  original game data and comparing behaviour, which is why §6 of the preface states the
  acceptance criteria behaviourally.

Each chapter README ends with its own honesty section listing what could not be recovered
from the source: magic constants with no derivation, dead fields, abandoned designs, and
decisions whose intent is simply not written down anywhere. Those lists are deliberately
specific — "this value is unexplained" is more useful to a rebuilder than a confident
invention.

---

## What is not here

- **`Externals/`** — vendored third-party code (the Lua runtime and its binding layer, the
  rigid-body and collision libraries, the debug-overlay toolkit, compression, image and
  profiling libraries, the matchmaking SDK). Each is described as a **seam** in the
  preface: what the engine demands of it, what the engine actually calls, and which
  substitutes satisfy the same contract. Naming the capability is the useful work;
  respecifying somebody else's library is not.
- **`res/gamedata/`** — the shipped shaders, configuration, UI layouts and assets. Their
  *formats* are specified in §5 of the preface and in the chapters that read them; the
  content belongs to the games.
- **`sdk/`, build files, CI configuration, editor settings, binary assets** — skipped as
  incidental.
- **`src/utils/mp_gpprof_server/libraries/`** — vendored third-party code inside a source
  directory; its own chapter page says so.

---

## Provenance

Generated with [Claude Code](https://claude.com/claude-code) by reading the source at the
commit above. The source repository is not modified by anything here, and this recipe
carries no code from it — only the reasoning recovered from it.

OpenXRay is a fan project, not affiliated with the original engine's authors; this recipe
is a description of that project's source and inherits no rights to the games' data.
