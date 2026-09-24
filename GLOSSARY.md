# Glossary

The vocabulary this engine uses about itself. Defined once here so the twins can be
terse, and inherited from the original's own type and file names rather than invented —
a reader holding this recipe alongside the source should find the same words in both.

Terms are grouped by the part of the system they belong to, and within a group in the
order a reader meets them.

---

## The world

**Level** — one loadable map: a directory of geometry, collision, navigation, lighting and
spawn data. Exactly one level is loaded in detail at a time. The games ship a few dozen.

**Sector** and **portal** — the visibility topology authored into a level. A sector is an
enclosed volume; a portal is a polygon joining two sectors. Visibility is resolved by
walking from the camera's sector through portals whose clipped projection is still
non-empty, which bounds what the renderer ever considers. A rebuild that substitutes a
different culling scheme will not reproduce the original's performance profile on the
shipped data, because the data was authored for this one.

**Spawn** — the authored record an entity is created from, stored in the level's spawn
file: a class identifier plus a class-specific payload. The same record format is what a
save game writes back, which is why the entity serialization contract is frozen.

**Alife** — the simulation of the *whole* world, including the parts nobody is looking at.
Creatures, items and events outside the loaded level are advanced at a coarse rate on the
cross-level graph; when one crosses into the loaded level it is promoted to a fully
simulated object, and demoted again on the way out. This is the series' signature
mechanic and the source of most of its state-management complexity. Also written A-Life.

**Online** and **offline** — the two states of an alife entity. *Online* means promoted to
a live, updating, renderable object in the loaded level; *offline* means a record advanced
coarsely by the alife simulation. The transition in both directions must preserve state
exactly (conformance criterion 11).

**Game graph** — the coarse, cross-level navigation graph the alife simulation moves
offline entities on. Its vertices are places; its edges carry travel time.

**Level graph** — the fine navigation mesh inside one level: a grid of walkable vertices
with per-vertex height, cover values and neighbour links. Both graphs ship prebuilt.

**Node** / **vertex** — a position in one of those graphs. Most AI positions in the engine
are stored as a graph vertex plus an offset, not as a free-floating coordinate, because
the vertex is what the pathfinder and the cover system can reason about.

**Smart terrain** — an authored place that hands out jobs. Creatures register with it and
it assigns each one a task (guard this point, patrol this path, sleep here), which is how
the world appears populated with purposeful behaviour without every creature carrying its
own script.

**Restrictor** — a volume that constrains where an entity may go, either as a permitted
region (an *out*-restrictor) or a forbidden one (an *in*-restrictor). An entity carries a
list of each, and the algebra is easy to get backwards: **within a list the volumes union;
the two lists then subtract.** The effective space is (union of the permitted volumes)
minus (union of the forbidden ones) — so naming two permitted regions widens the space,
it does not narrow it to their overlap. The pathfinder consults the result as part of its
cost model. A restriction that names a restrictor which has not spawned stays inert rather
than forbidding everything.

**Anomaly** / **zone** — a volume with an effect on entities inside it. Zones are
first-class entities with their own alife presence, not level decoration.

---

## Entities

**Server object** — the authoritative record of an entity: its identity, its position, its
class-specific state, and everything that must survive a save or reach a network client.
It exists whether or not the entity is currently simulated.

**Client object** — the live, local instance: renderable, animated, physically simulated,
updated every frame. In single player both live in one process, which is the single most
confusing thing about this codebase for a newcomer; the names come from the multiplayer
architecture and are used everywhere regardless.

**Game object** — the facade the scripts see. One handle exposing the parts of both the
server and the client sides that the script layer is allowed to touch. Its exported
surface is frozen by conformance criterion 10.

**Entity identifier** — a 16-bit handle that names an entity across the save file, the
network protocol and every internal registry. Its width is load-bearing: it caps the
world's entity count and it is packed into network messages at that width.

**Class identifier** — the fixed-width tag in a spawn record that selects which class to
instantiate. The values ship in the game data and are therefore frozen; the factory that
maps tag to constructor is a registration table filled at startup.

**Section** — the configuration section, named by string, that supplies an entity's tuned
parameters. An entity is (class identifier, section): the class supplies the behaviour and
the section supplies every number.

---

## Behaviour

**Brain** — the per-creature decision loop that runs the planner, re-evaluating its goal
and its plan as the world changes.

**Evaluator** and **operator** — the planner's two halves. An evaluator answers one
question about the current world state; an operator is an action with preconditions
expressed over those answers and effects that change them. Planning is a search over
operators from the current world state to a goal state.

**Motivation** / **goal** — the target world state the planner is searching toward. The
brain selects among competing motivations by weight each cycle.

**Scheduler** — the engine's per-frame update budget manager. Registered objects get an
update call at a rate that degrades with distance and load, so that a thousand entities
can exist without a thousand full updates per frame. The degradation rules are visible in
gameplay and are load-bearing.

**Feel** — the senses. Three of them, each a separate opt-in interface an object can
implement: *touch* (which objects are near me), *vision* (which objects can I see, with
a frustum, a range and a raycast against the collision database) and *sound* (which sound
events reached me, with the emitter's AI-perception attributes deciding how loud and what
kind). Perception is *event-driven*, not polled, which is why the sound sidecar carries AI
attributes at all.

**Cover** — a precomputed per-vertex measure of how exposed a navigation position is from
each direction, used to answer "where can I stand where he cannot see me". Shipped with
the level.

**Danger** — a recorded threat with a source, a type and a decay, feeding the brain's
motivation weighting.

**Dialog** / **phrase** — the conversation system: authored phrase graphs with
preconditions and script callbacks, driving both the talk screen and reputation effects.

---

## Animation and models

**Bone** and **skeleton** — the rigid hierarchy a skinned model deforms with. Bones carry
physics shapes and hit-zone identity as well as transforms, so a bone is simultaneously an
animation node, a collision proxy and a damage target.

**Motion** — one animation clip. Motions live in shared banks, referenced by name, so that
many models can draw on one library.

**Blend** — an active playing motion with a weight, a speed and a time; the pose is the
weighted accumulation of blends, partitioned by bone group so that an upper body can aim
while the legs walk.

**Part** / **bone group** — the named subset of the skeleton a blend applies to.

**Callback** (animation) — a marker at a time within a motion that fires into the game
layer: the frame a footstep sounds, a casing ejects, a hit lands.

**Kinematics** — the interface the game layer drives an animated model through; the
renderer implements it. Deliberately a renderer interface rather than a game one, because
the pose is needed to skin the mesh.

---

## Rendering

**Visual** — a renderable object's geometry plus its material bindings. Visuals are loaded
from the model format and shared between instances.

**Render backend** — one filling of the graphics-device seam. Two ship: OpenGL everywhere,
Direct3D 11 on Windows. Selected at startup by name, with fallback.

**Render level** — the shipped renderers are numbered by generation (a fixed-function-era
forward path, a deferred path, and a deferred path with more modern lighting). The number
appears in console variables, in shader directory names and in the data, so it is part of
the frozen vocabulary.

**Shader** — in this codebase the word usually means a *material pass description* loaded
from a data file: which GPU programs, which textures, which blend and depth state, in how
many passes. The GPU programs themselves are called *programs* or by their stage.

**Detail objects** — the grass-and-debris layer: many instances of a few small meshes,
placed procedurally from a per-level density map and animated by wind.

**Wallmark** — a decal projected onto geometry: bullet holes, blood, scorch.

**Environment** — the weather and time-of-day system: a day's worth of keyframed
atmospheric parameters, interpolated by clock and blended between named weather sets.

---

## Data and configuration

**LTX** — the engine's configuration format: an INI-like text file with sections,
key/value lines, section inheritance declared on the section header, and includes. Nearly
every tunable number in the game lives in one.

**Chunk** — the unit of the engine's binary container format: a recursive (identifier,
size, payload) record. Levels, models, saves and most other binary assets are trees of
chunks.

**Archive** / **virtual filesystem** — the mounted chain of compressed archives plus loose
directories that the engine reads all game data through, addressed by logical roots such
as `$game_data$`.

**Console variable** — a named, typed runtime setting. The name set is frozen because
shipped configuration files and user settings address them by name.

**String table** — the localization lookup: identifier to display string, per language,
loaded from XML. UI text is never a literal.
