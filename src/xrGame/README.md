# src/xrGame

This is the game. Everything a player recognizes is here: the actor, the weapons they carry,
the creatures that shoot back, the anomalies, the inventory, the conversations, the quests,
the multiplayer modes, and the off-screen simulation that keeps the rest of the world moving
while they are not looking at it. Sixteen hundred files, which is more than a third of the
repository, and by far the largest chapter.

Read it as the answer to five questions, because almost every twin in it presupposes one of
them and none of them restates it:

1. **What is an entity?** There are three objects for every one thing in the world, they have
   confusing names, and in single player they all live in the same process.
2. **How does one come to exist, and how does it stop?** The spawn path is the most
   order-sensitive code in the repository and the teardown order is asserted at runtime.
3. **What holds them together?** One inheritance chain carries every entity in the game, with
   a fixed set of behaviours mixed into it.
4. **What happens outside the loaded level?** The off-screen simulation, and the promotion and
   demotion that move an entity across the boundary without losing state.
5. **What does a frame do?** The level's per-frame order, which is the order every subsystem
   in this chapter is driven in.

A sixth question — *what may scripts touch* — is answered by roughly two hundred exported
classes whose names and signatures are frozen, and which are therefore the hardest thing in
the chapter to change.

## Where it sits

Chapter 23 of the build order, and the last chapter with substantial new ideas in it. It rests
on everything: the platform and container layers ([`src/Common`](../Common/README.md),
[`src/xrCommon`](../xrCommon/README.md),
[`src/utils/xrMiscMath`](../utils/xrMiscMath/README.md)), the virtual filesystem and the
configuration parser ([`src/xrCore`](../xrCore/README.md)), the collision database
([`src/xrCDB`](../xrCDB/README.md)), the material table
([`src/xrMaterialSystem`](../xrMaterialSystem/README.md)), the transport
([`src/xrNetServer`](../xrNetServer/README.md)), the script virtual machine
([`src/xrScriptEngine`](../xrScriptEngine/README.md)), audio
([`src/xrSound`](../xrSound/README.md)), the frame loop and the object registry
([`src/xrEngine`](../xrEngine/README.md)), the navigation and planning machinery
([`src/xrAICore`](../xrAICore/README.md)), the widget toolkit
([`src/xrUICore`](../xrUICore/README.md)), the dynamics
([`src/xrPhysics`](../xrPhysics/README.md)) and the authoritative entity records
([`src/xrServerEntities`](../xrServerEntities/README.md)).

It is **loaded as a module**, not linked: the engine knows only the four operations in
[`xrGame.h`](xrGame.h.md) and two raw function pointers for creating and destroying an object
by class identifier. That boundary is worth keeping even in a rebuild that fuses the two
programs, because it is the one place guaranteed to run after the factory's registration table
is filled and before the first spawn.

Five sub-chapters have their own directories and their own openers:
[`ai`](ai/README.md) (concrete creature brains), `ui` (the game's screens, built on the widget
toolkit), [`ik`](ik/README.md) (foot placement),
[`CdkeyDecode`](CdkeyDecode/README.md) and [`gamespy`](gamespy/README.md) (the dead
matchmaking glue). Everything else is in this directory's own listing below.

The cycle the system requirements name — this chapter against
[`xrServerEntities`](../xrServerEntities/README.md) — is not an accident and not really a
cycle: the entity records are one shared data module compiled into both the game and the
offline tools with a different macro set. A rebuild makes it one module with two consumers.

---

## Part 1 — The object model

Three objects exist for every one thing in the world. Their names come from the multiplayer
architecture and are used everywhere regardless, which is the single most confusing thing in
this codebase for a newcomer.

**The server object** is the authoritative record: an entity identifier, a class identifier, a
position, a configuration section, and whatever class-specific state must survive a save or
reach a network client. It exists whether or not the entity is being simulated in detail, and
it is what the save file contains. Its types are declared in
[`xrServerEntities`](../xrServerEntities/README.md) and its registries are in this chapter's
alife files.

**The client object** is the live, local instance: it has a model, a skeleton, a rigid body, a
place in the space partition, a slot in the update scheduler and a per-frame update. It is
what the renderer draws and what the physics world steps. Its base is
[`GameObject.cpp`](GameObject.cpp.md).

**The game object** is the facade scripts see —
[`script_game_object.cpp`](script_game_object.cpp.md) and its dozen siblings. It is created
lazily, on the first script access, and it exposes a curated subset of *both* the client and
the server sides through one handle.

The relationships are not symmetric, and getting them backwards is the commonest way to
misread a twin:

- **The server object owns the truth.** A client object is *derived* from it and may be
  destroyed and rebuilt from it at any time — that is exactly what promotion and demotion do.
- **The client object owns the presentation and the simulation.** Nothing on the server side
  has a pose, an animation or a physics body.
- **The game object owns neither.** It holds an entity identifier and reaches through it. A
  script holding a game object whose entity has been destroyed is holding a dead handle, which
  is why almost every script-facing method begins by checking that the object still exists.
- **A client object may exist with no server object behind it** — purely presentational things
  do. The reverse is the normal state of most of the world.

**Why single player runs both sides.** There is exactly one session path in this engine. A
single-player game constructs a server and a client in the same process, connected by a loopback
transport, and every entity therefore has both halves living a few hundred bytes apart. This is
not a vestige and it is not overhead worth removing: the alife simulation *is* the server, saves
are server-side snapshots ([`Level_SLS_Save.cpp`](Level_SLS_Save.cpp.md) writes the whole
server world and [`Level_SLS_Load.cpp`](Level_SLS_Load.cpp.md) is empty because the restore
happens entirely on the other side), and promotion goes through the same spawn-message path a
network client would receive. One construction path for live objects, used by both modes, is
what keeps single player and multiplayer from diverging — and it is why a client may never
mutate a server record directly. It emits an event and asks.

A rebuild is free to collapse the two in a single-player-only build. It should expect to
reimplement saving, the offline simulation and every script spawn call when it does.

---

## Part 2 — The spawn and destroy path

### From a record to a live object

A spawn record is `(class identifier, entity identifier, position, configuration section,
class-specific payload)`, read either from the level's spawn file
([`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md)), from a save
([`alife_storage_manager.cpp`](alife_storage_manager.cpp.md)), or from a script asking for a
new entity. It reaches a live object through one path and only one
([`Level_network_spawn.cpp`](Level_network_spawn.cpp.md)):

```text
1. the record arrives as a spawn message and is decoded into a server object
2. the class identifier selects a constructor from the factory registration table
3. the new client object is constructed, and the identifier is stamped onto it
4. the object's spawn hook runs against the record — the order-sensitive part, below
5. ownership and player control are wired up
```

The **class-identifier factory** is a registration table filled at startup, mapping a
fixed-width tag that ships in the game data to a constructor. The tags are frozen; the table
is not. The engine holds the two function pointers ([`xrGame.cpp`](xrGame.cpp.md)) and knows
nothing else about what it is creating. An unregistered identifier is a *data* error, caught
during development and fatal in a release build — deliberately, because a level whose spawn
file names a class this build does not have is not a level that can be played.

**The spawn hook is the most order-sensitive function in the chapter.** Its full sequence is in
[`GameObject.cpp`](GameObject.cpp.md); the shape a rebuilder needs is:

```text
identity and visual        # before anything can ask what this is or how it looks
claim the entity identifier — refuse a duplicate
transform from the record  # before any registry is told a position
per-instance configuration overrides carried in the record itself
ownership, then registration in the level's object registry
class resolution: reload(section) then reinit()
  — reload picks the class's tuned values, reinit clears per-life state, in that order
saved per-object state, if this spawn came from a save game
navigation position: level-graph vertex and game-graph vertex
  — taken from the record when valid, re-derived from the position when not;
    a contained object inherits its container's place rather than re-deriving
visual and collision model fall back to the configuration section
the script binder's own spawn runs last
```

Each registry is entered once the thing it indexes is valid, and never before. The space
partition is told a position only after the transform is set; the scheduler is told the object
exists only after it is in the object registry; the script binder runs last because a script's
spawn callback is entitled to ask the object anything, including where it is on the navigation
mesh. Reordering these produces failures that present far from their cause — a script seeing an
entity at graph vertex "invalid", or an update delivered to an object the space partition has
never heard of.

**Registration with the scheduler** deserves its own note, because it is where the chapter's
scale becomes affordable. A level has thousands of objects wanting periodic work. Objects
needing unconditional per-frame work are driven by the engine's object registry; everything
else declares how much it currently matters and is advanced by the time-budgeted scheduler at
a rate that degrades with distance and load. Most creatures in this chapter therefore have two
update rates and the twins say so —
[`CustomMonster.cpp`](CustomMonster.cpp.md) splits a slow thinking tick from a fast
presentation tick and interpolates the visible pose between them, which is what a reduced
update rate looks like from the player's side.

**The render graph and the physics world** are entered by the two mix-ins every drawable and
every collidable object carries. Both are *conditional*: an invisible object is not registered
as renderable and a disabled one is not registered as collidable, and the flags that say so are
consulted every frame rather than at spawn.

### Destruction

The teardown order is the spawn order reversed, and it is **asserted at runtime**: no entity is
registered twice, and a destroyed entity must be unreferenced by the scheduler, the render graph
and the physics world before its memory is released.

```text
the object is marked for destruction — it is never destroyed where it dies
  (a particle effect cannot be deleted inside its own callback; an item cannot be
   deleted inside the inventory routine that dropped it)
at the end of the frame, the queued objects are torn down:
  the script binder detaches first, and a failure in it is swallowed rather than propagated
  every holder of a reference is told: memories, senses, the sight and enemy managers,
    the movement manager's obstacle records, the inventory, the physics world
  the object leaves the scheduler, the space partition, the render graph
  the physics assembly is released through the dynamics seam, which unregisters
    every body before freeing it
  the storage is released
```

Two conventions make this work and both are load-bearing. **Deferred destruction** — nothing is
destroyed at the point it is decided — is why the destroy flag exists on every object. And
**reference removal is a broadcast**: every subsystem that can hold a reference to an entity has
a hook, and the hooks are called in a fixed order before anything is freed. A rebuild with
reference counting still wants the broadcast, because the subsystems are not merely holding
references — they are holding *derived state about* the entity that must be recomputed.

---

## Part 3 — The inheritance spine

A rebuilder needs this map before any individual twin makes sense. Almost every file in this
chapter is a point on this chain or a behaviour mixed into it, and the twins name their position
without redrawing it.

```text
IGameObject                       # the engine's contract for a client object (chapter 13)
  └── CGameObject                 # GameObject.cpp — identity, registries, navigation
      │                             position, script binding, callback table
      │  mixes in: factory object · spatial · scheduled · renderable · collidable
      │
      ├── CSpaceRestrictor        # an invisible volume assembled from spheres and boxes
      │   └── CCustomZone         # + Feel::Touch — every anomaly
      │
      └── CPhysicsShellHolder     # owns a rigid-body assembly; the adapter the dynamics
          │                         module asks the game layer questions through
          │  mixes in: particles player · physics-collision view
          │
          ├── CPhysicItem         # a thing with a body that is not alive
          │   └── CInventoryItemObject   # + CInventoryItem — everything you can pick up
          │       └── CHudItemObject     # + CHudItem — everything you can hold
          │           └── CWeapon        # + CShootingObject
          │
          └── CEntity             # + CDamageManager — can be damaged, can die
              └── CEntityAlive    # condition, wounds, blood, burning, faction relations
                  │
                  ├── CCustomMonster      # every thinking creature
                  │   │  mixes in: script entity · Feel::Vision · Feel::Sound · Feel::Touch
                  │   └── CAI_Stalker     # + object handler · phrase dialogue · step manager
                  │
                  └── CActor              # the player's entity
                         mixes in: input receiver · Feel::Touch · Feel::Sound ·
                                   inventory owner · phrase dialogue · step manager
```

The behaviours mixed into that chain are the chapter's real vocabulary. Each is an opt-in
interface with its own twin, and an entity's capabilities are the list of the ones it carries:

| Behaviour | What carrying it means |
|---|---|
| **Feel: touch** | "Which entities are inside my radius right now", with enter and leave edges. Carried by creatures, by the actor, and by every anomaly. |
| **Feel: vision** | A frustum query, a set difference against last frame, and one cached ray per candidate. Creatures only. |
| **Feel: sound** | Told when a sound it could hear was emitted, with the emitter's AI-perception attributes. Event-driven, never polled. |
| **Damageable** | A per-bone damage table ([`damage_manager.cpp`](damage_manager.cpp.md)) and a per-damage-type immunity table ([`hit_immunity.cpp`](hit_immunity.cpp.md)). A bone is simultaneously an animation node, a collision proxy and a damage target. |
| **Condition** | Health, stamina, radiation, psychic health, morale and open wounds, advanced against in-world time ([`EntityCondition.cpp`](EntityCondition.cpp.md)). |
| **Inventory owner** | Three storage areas and a slot that is in the hands ([`Inventory.cpp`](Inventory.cpp.md)), plus the character's known information, reputation and trade relationships. |
| **Inventory item** | Name, weight, cost, condition, grid footprint, upgrade list ([`inventory_item.cpp`](inventory_item.cpp.md)). |
| **Attachable item** | May hang visibly on a bone of whoever carries it ([`attachable_item.cpp`](attachable_item.cpp.md)); an attachment owner is the other end. |
| **Held item** | A state machine driven by *animation ends* rather than by a clock ([`HudItem.cpp`](HudItem.cpp.md)). |
| **Holder** | Something the actor can climb into, which takes over their input, camera and movement ([`holder_custom.h`](holder_custom.h.md)) — a vehicle, a mounted gun, a fixed camera. |
| **Physics shell holder** | Owns a rigid-body assembly and answers the dynamics module's questions ([`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md)). |
| **Script entity** | Accepts a script-authored behaviour attached from configuration, and forwards its whole lifecycle to it ([`script_binder.cpp`](script_binder.cpp.md)). |
| **Restricted object** | Carries permitted and forbidden volumes that constrain where it may go. Within a list the volumes union; the two lists then subtract. |
| **Phrase dialogue manager** | Can hold a conversation from the authored phrase graphs. |
| **Object handler** | Can be told "equip this and do that with it" as a planning goal rather than as a call. |

The two rules that make this chain readable rather than a thicket:

**Multiple inheritance is used as capability composition, not as type hierarchy.** The chain
down the left is the single spine; everything else joined to a class is a behaviour with no
state overlap. A rebuild with components, traits or interfaces reproduces the chain as
composition directly, and the only thing it must preserve is the *ordering* between behaviours
at each lifecycle hook — which the twins of the joined classes state case by case (see
[`inventory_item_object.cpp`](inventory_item_object.cpp.md) for the pattern in its clearest
form).

**Capability questions are a closed set of cheap downcasts.** Rather than a class-identifier
switch, every game object answers a fixed list of questions about itself — *are you a weapon,
food, a missile, a held item, ammunition, an inventory item, an attachable, a physics holder* —
with the base answering no and each subclass overriding the ones it becomes. The set is closed:
adding a kind means adding a question to the common base. Callers rely on the answers being
total and free.

---

## Part 4 — The alife simulation

The series' signature mechanic, and the source of most of this chapter's state-management
complexity. The whole world is simulated, including the parts nobody is looking at.

**Online and offline are the two states of an alife entity.** *Online* means promoted to a live
client object in the loaded level. *Offline* means a record advanced coarsely on the cross-level
game graph. Most of the world is offline at any moment; the loaded level is a detailed window
onto a coarse simulation that never stops.

**What advances offline.** Not much, deliberately, and that is the design rather than a
limitation. An offline entity has a position on the game graph and a schedule
([`alife_update_manager.cpp`](alife_update_manager.cpp.md) advances it under a time budget, the
same way the scheduler advances client objects). Offline, an entity:

- travels along game-graph edges toward wherever its decision loop is sending it, at the edge's
  authored travel time;
- runs that decision loop at squad granularity, not individually — a squad asks its smart
  terrain what it should be doing and walks there
  ([`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md));
- accumulates the passage of in-world time against its condition and its inventory;
- can be created, destroyed, moved and edited by script, which is how the story happens.

It does **not** path on the navigation mesh, perceive anything, animate, collide or fight in any
detail. Offline combat is resolved by the coarse rules of the squad layer. The two simulations
are deliberately different in kind, not merely in resolution.

**Promotion and demotion** are the boundary, and conformance criterion 11 is about exactly this
file: [`alife_switch_manager.cpp`](alife_switch_manager.cpp.md). The rules a rebuild must
reproduce:

- **Two radii, not one.** Promotion uses a smaller radius and demotion a larger one, both derived
  from an authored nominal distance and a hysteresis fraction. An entity hovering at the boundary
  must not flip every frame; with one radius it will, visibly.
- **Promotion goes through the spawn-message path**, exactly as a network client would receive
  it, rather than handing the record to the client side directly. The spawn path is the only code
  that knows how to build a client object, and routing through it is what makes single player and
  multiplayer construct live objects identically.
- **Demotion writes back before it destroys.** The live object's state is serialized into the
  record — position, condition, inventory, every class-specific field — and only then is the
  client object torn down. The serialization is the *same* one the save file uses, which is why
  the entity serialization contract is frozen: a save and a demotion produce the same bytes.
- **Containment is preserved as a tree.** An entity's children — the items in a creature's
  inventory — are promoted and demoted with it, parent first, and the record registry saves and
  loads them as a parent-first tree of spawn-plus-update packets
  ([`alife_object_registry.cpp`](alife_object_registry.cpp.md)).
- **The registries must agree.** An entity is in exactly one arrangement: either a live object in
  the object registry, or a record indexed by game-graph vertex
  ([`alife_graph_registry.cpp`](alife_graph_registry.cpp.md)). Being in both, or in neither, is
  the failure this whole file exists to prevent, and it is what the runtime invariant "no entity
  is registered twice" is checking.

**Level change and new game** are the two world-scale operations, and both are in
[`alife_update_manager.cpp`](alife_update_manager.cpp.md). They rewrite everything else, which is
why they are deferred events rather than calls.

---

## Part 5 — The level

The level is the game layer's object that exists for as long as a match does
([`Level.cpp`](Level.cpp.md) and a dozen siblings). It owns, for the duration:

- the client half of the session and the server half in single player;
- every live client object, through the engine's object registry;
- the bullet manager — every projectile in flight, simulated centrally
  ([`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md));
- the space-restriction registry, the map and task managers, the ambient sound manager;
- the level's own script process and its deferred physics command queues;
- the input dispatch chain from the window to the controlled entity
  ([`Level_input.cpp`](Level_input.cpp.md)).

**The client/server split inside one level.** The server side owns the entity table, the client
table, and the switch that decides what a received message is allowed to do
([`xrServer.cpp`](xrServer.cpp.md)). The client side owns the live objects, the prediction and
the presentation ([`game_cl_base.cpp`](game_cl_base.cpp.md)). Between them sits an **ordered
event queue drained on a deliberate lag** — server time minus a fixed latency allowance — which
is what lets events arriving out of order still be applied in order. In single player the two
sides are microseconds apart and the lag is invisible; the discipline is what keeps one code path
serving both.

**The per-frame order** is the load-bearing content of [`Level.cpp`](Level.cpp.md), and it is the
order every subsystem in this chapter is driven in:

```text
 1  commit last frame's bullet hits          # before anything reads its own health
 2  drain the transport into the event queue, or tear down if disconnected
 3  drain the event queue up to (server time − latency)
 4  correction prediction, if an update arrived needing it
 5  the presentation-side managers: map, and in single player the task manager
 6  the engine's own frame — every registered object's update, then the scheduler
 7  hand the clock to the weather system
 8  the level's script process
 9  deferred physics work: the engine's queue, then the scripts'
10  build this frame's bullet render set        # after everything moved
11  ambient sound, then one incremental step of script garbage collection
```

Three things about this shape are decisions:

**The bullet manager is touched twice and the two halves straddle the object update.** Committing
hits first applies last frame's damage before anybody reads their own condition; building the
render set last captures where the tracers actually ended up. Merging them changes what the
player sees.

**Script work is incremental, every frame, forever.** One garbage-collection step per frame rather
than letting a collection run to completion, because the script heap is large enough that a full
collection is a visible hitch.

**Parallelism is four items long.** The map manager, the ambient sound manager, script garbage
collection and (disabled) the task manager can each be pushed onto a worker queue behind its own
flag. They are the four that touch no shared mutable state. Everything else in the frame is
serial by construction, and that is this engine's defining performance characteristic.

Above the level sits [`GamePersistent.cpp`](GamePersistent.cpp.md), the object that outlives any
one level: it brings the game layer up and down around the engine's startup, owns the main menu
and the loading screen, and drives the intro chain.

---

## Part 6 — The script surface

Roughly two hundred classes are exported to the script virtual machine, and **conformance
criterion 10 freezes all of them** — names, signatures, the callback set and the order callbacks
fire in. It is the strictest criterion in the system requirements and the hardest constraint in
this chapter. Modifications are the product for a large part of this engine's audience, and every
shipped script is written against this exact surface.

What that means in practice:

- **A rebuild may reimplement any exported method and may not rename one.** The same rule the
  console commands live under, applied to a far larger surface.
- **The callback set is a numbered catalogue** ([`game_object_space.h`](game_object_space.h.md)) —
  the event vocabulary of the whole script layer. Adding to it is safe; renumbering is not.
- **The facade is assembled from partial registrations.** The game object's exported surface is
  split across a dozen files, each registering a slice, and one file assembles the chain
  ([`script_game_object_script.cpp`](script_game_object_script.cpp.md)). The split is by topic and
  is arbitrary; the *union* is the contract.
- **Script behaviours attach from configuration.** An entity's section may name a script class,
  which is instantiated and receives the object's whole lifecycle
  ([`script_binder.cpp`](script_binder.cpp.md)). A failure in a script binder is swallowed rather
  than propagated — a broken modification degrades rather than crashing the game.
- **Scripts spawn, destroy, move and edit alife records directly**, which is how the story is
  told. Those calls go through the same paths the engine's own spawns do.

The binding files are listed as their own subsystem below. They are mechanical, they are the
largest single group in the chapter, and they are the last thing a rebuild should write and the
first thing it should test.

---

## Ideas to hold before reading the twins

**An entity is (class identifier, configuration section).** The class supplies the behaviour and
the section supplies every number. Nearly every constant in the game is in an
[`ltx`](../xrCore/README.md) file, and a twin that says "read from configuration" means the
rebuilder must not invent the value.

**Three games, one executable.** *Shadow of Chernobyl*, *Clear Sky* and *Call of Pripyat* ship
different data and expect different behaviour. One of three process-wide flags is set at startup
and the game layer branches on it at every divergence — different upgrade prerequisite rules,
different script refusal codes, different inventory slot defaults, different weather handling.
Each branch is noted in the twin that carries it. There are dozens of them and none can be
removed without dropping support for a game.

**Managers, not systems.** A creature owns a memory manager, a movement manager, a sight manager,
an inventory, a sound player, an animation manager. Each is a component with its own lifecycle
hooks, constructed by its owner and fanned out to by it. The fan-out *order* is the content of
most of those owners' twins.

**Derived state is rebuilt, not maintained.** The clearest example is the creature's memory:
three senses record what happened, and three judgements — who is my enemy, what is worth picking
up, what is dangerous — are cleared and rebuilt from those records every frame
([`memory_manager.cpp`](memory_manager.cpp.md)). Nothing caches a pointer into a judgement across
frames, and the judgements are not saved because they are derived.

**Nothing is destroyed where it dies.** Objects are queued and torn down at a frame boundary;
events are queued and delivered at the top of the next frame; physics changes are queued into
command lists. A twin that says "cannot be deleted here" is naming this rule.

**The navigation graph is the AI's coordinate system.** Most AI positions are a graph vertex plus
an offset, not a free coordinate, because the vertex is what the pathfinder and the cover system
can reason about. Two graphs: a coarse cross-level one the offline simulation moves on, and a fine
per-level one creatures path on. Both ship prebuilt.

**The restrictor algebra is easy to get backwards.** Within a list the volumes union; the two lists
then subtract. Naming two permitted regions *widens* the space. A restriction naming a restrictor
that has not spawned stays inert rather than forbidding everything.

---

## What could not be recovered

Written honestly, because a rebuilder who believes the recipe is complete will waste time looking
for reasons that are not there.

**Feel values with no derivation.** A very large number of the constants in this chapter are
hand-tuned and have no discoverable justification: the weapon-inertia catch-up speeds and swing
magnitudes, the ammunition-to-cost weighting in the quick-switch ranking, the two-second
quick-switch cycle reset, the ten-times clamp on animation-derived body velocities, the
0.35-metre "too close to pass" width in obstacle avoidance, the fifty-millisecond pose-sampling
interval, the 4096-vertex ceiling on trial path searches, the five-metre obstacle-forgetting
radius. Each twin says so where it appears. A rebuild should treat them as data to be re-tuned
against the shipped content, not as formulas to be re-derived.

**Wire fields nobody reads.** The account message carries an achievement list that is parsed and
discarded, and the second value of each achievement pair has no discoverable meaning. The
information-transfer event carries a sender identifier that no code consumes. Both must still be
present, byte for byte, to stay compatible with the original executable.

**Disabled code whose reason is unknown.** Several mechanisms are written, complete and switched
off: the detail-path restrictor re-verification, four of the five timestamps on every memory
record, the upgrade tooltip hook, the precondition checks on every smart-cover planner target.
In each case the *what* is recoverable from the source and the *why* is not — whether the feature
was cut for cost, for behaviour, or because it was wrong.

**Bugs that are now contracts.** The merged memory record sets only the winning sense's flag
rather than all contributing ones; a script asking "did I both see and hear him" gets the newest
sense only. Shipped scripts were written against that behaviour. A rebuild that "fixes" it changes
the game. Similar cases are noted individually.

**Formats frozen against a program, not a specification.** The network protocol, the save format
and the upgrade forest's shape are all frozen only against this codebase — but *are* frozen
against it, because existing saves and existing multiplayer clients depend on them. The upgrade
forest in particular is part of the save format: moving an upgrade between groups invalidates
existing saves, and nothing in the source says so.

**Fifteen multiplayer game-mode files had no twin at the time this chapter was written** — the
deathmatch, artefact-hunt and capture-the-artefact server and client rules. They are listed below
with roles derived from their names and their siblings rather than from a full reading, and are
the one place in this chapter where the listing is less trustworthy than the twins.

**What the tools say about the data.** [`player_hud_tune.cpp`](player_hud_tune.cpp.md) is worth
reading for what it implies rather than for what it does: the eleven numbers that place a weapon
in the player's hands were arrived at by eye, per weapon, per aspect ratio, by a developer moving
sliders. None of them is derivable. That is true of more of this chapter's data than the code
admits.

---

## Twins

Grouped by subsystem, and within a subsystem alphabetically. Every file in this directory is
listed; the five subdirectories have their own chapters. Build files
(`CMakeLists.txt`, `packages.config`, `xrGame.vcxproj`, `xrGame.vcxproj.filters`) describe how
the module is compiled and have no twins.

A note on reading the listing: files ending `_inline.h` exist because C++ must define a member
after its class is complete, and files ending `_script.cpp` exist because the binding registration
is kept out of the implementation. Neither split carries a decision, and a rebuild folds both back
into the file they belong to.

### Build and module entry

| File | Role |
|---|---|
| [`StdAfx.cpp`](StdAfx.cpp.md) | The one source file that exists to make the precompiled prelude compile. Nothing else. |
| [`StdAfx.h`](StdAfx.h.md) | The game module's precompiled prelude: a build artifact, not a design decision. |
| [`xrGame.cpp`](xrGame.cpp.md) | The game module's entry point: what the engine calls to bring the game layer up, and the two functions through which every entity in the world is created and destroyed. |
| [`xrGame.h`](xrGame.h.md) | Declares the four-operation boundary between the engine and the game layer, and the two raw entity-factory entry points that cross it. |
| [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md) | Fills and empties the game layer's shared, process-wide tables — the character, dialogue and reputation data every entity reads but none owns. |

### The entity spine and holders

| File | Role |
|---|---|
| [`attachable_item.cpp`](attachable_item.cpp.md) | The half of an inventory item that lets it hang visibly on a bone of whoever is carrying it. |
| [`attachable_item.h`](attachable_item.h.md) | Declares the attachable-item mix-in implemented in `attachable_item.cpp` and `attachable_item_inline.h`. |
| [`attachable_item_inline.h`](attachable_item_inline.h.md) | The trivial accessors and the initial state of the attachable-item mix-in. |
| [`attachment_owner.cpp`](attachment_owner.cpp.md) | The half of a carrier that holds a list of items hanging on its skeleton and re-places them every time its bones are evaluated. |
| [`attachment_owner.h`](attachment_owner.h.md) | Declares the attachment-carrier mix-in implemented in `attachment_owner.cpp`. |
| [`Entity.cpp`](Entity.cpp.md) | The thing that can be damaged and can die: health, team membership, the damage path, and the once-only bookkeeping of who killed it and when. |
| [`Entity.h`](Entity.h.md) | Declares the damageable, killable, team-affiliated entity implemented in `Entity.cpp`. |
| [`entity_alive.cpp`](entity_alive.cpp.md) | The base of everything that can be hurt: condition and wounds, blood and burning, faction relations, death, and the surface a bullet is allowed to land on. |
| [`entity_alive.h`](entity_alive.h.md) | Declares the base of every creature that can be hurt, implemented in `entity_alive.cpp`. |
| [`entity_alive_inline.h`](entity_alive_inline.h.md) | Field access for the living-entity base: the condition model, and two behaviour flags. |
| [`EntityCondition.cpp`](EntityCondition.cpp.md) | Everything a living creature can be in the middle of: health, stamina, radiation, psychic health, morale and a set of open wounds, all advanced against in-world time and all changed through one accumulator per … |
| [`EntityCondition.h`](EntityCondition.h.md) | Declares what a living creature's condition is made of — the value set, the wound list, the seventeen boost kinds and the consumable record — implemented in `EntityCondition.cpp`. |
| [`GameObject.cpp`](GameObject.cpp.md) | The base class every entity in the game is: the client object's lifecycle from spawn record to destruction, its place in the spatial and scheduling registries, its navigation-graph position, its script binding … |
| [`GameObject.h`](GameObject.h.md) | Declares the base class every client object in the game derives from, implemented in `GameObject.cpp`. |
| [`holder_custom.cpp`](holder_custom.cpp.md) | Records who is currently riding a holder. |
| [`holder_custom.h`](holder_custom.h.md) | The interface anything a player can climb into must satisfy: a vehicle, a mounted gun, a fixed camera — anything that takes over the actor's input, camera and movement. |
| [`holder_custom_script.cpp`](holder_custom_script.cpp.md) | Exports the rideable-thing interface to script under the name `holder`. |
| [`HolderEntityObject.cpp`](HolderEntityObject.cpp.md) | A generic mountable object: a static prop the player can occupy, which takes over the camera and the input while occupied. The minimum viable holder, with the weapon and driving behaviour deliberately absent. |
| [`HolderEntityObject.h`](HolderEntityObject.h.md) | Declares the generic mountable prop, implemented in `HolderEntityObject.cpp`. |
| [`inventory_owner_info.cpp`](inventory_owner_info.cpp.md) | What a character *knows*: the story-information portions they have received, how one is granted or withdrawn, and the script actions a grant fires. |
| [`inventory_owner_inline.h`](inventory_owner_inline.h.md) | One accessor: an inventory owner's trade parameters. |
| [`InventoryOwner.cpp`](InventoryOwner.cpp.md) | The character half of an entity: an inventory, a personality record, money, trade and conversation state, and the knowledge it has received. |
| [`InventoryOwner.h`](InventoryOwner.h.md) | Declares the character mixin — inventory, personality, money, trade, conversation and knowledge — implemented across `InventoryOwner.cpp` and its siblings. |
| [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md) | The base of every game object that can own a rigid-body assembly, and the adapter through which the physics module asks the game layer questions it is not allowed to know the answers to. |
| [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) | Declares the physics-owning game object base and the adapter interface it presents to the dynamics module, implemented in `PhysicsShellHolder.cpp`. |

### The script surface

| File | Role |
|---|---|
| [`account_manager_script.cpp`](account_manager_script.cpp.md) | Exports the account manager and its four callback shapes to the script virtual machine, so the account screens can be written in Lua. |
| [`action_base_script.cpp`](action_base_script.cpp.md) | Exports the planner operator to the script virtual machine, so that a creature's actions can be written in Lua and planned alongside the engine's own. |
| [`action_planner_action_script.cpp`](action_planner_action_script.cpp.md) | Exports the composite planner-action to the script virtual machine, so that a nested brain can be written in Lua. |
| [`action_planner_action_script.h`](action_planner_action_script.h.md) | Declares the bridge that lets an engine-written composite action live in a script-facing planner. Behaviour is in `action_planner_action_script_inline.h`. |
| [`action_planner_action_script_inline.h`](action_planner_action_script_inline.h.md) | Recovers the concrete creature from the script-visible facade when a composite engine action is installed into a script-facing planner. |
| [`action_planner_script.cpp`](action_planner_script.cpp.md) | Exports the brain to the script virtual machine: build a planner in Lua, give it actions and evaluators, aim it at a goal, and drive it. |
| [`action_planner_script.h`](action_planner_script.h.md) | Declares the bridge that lets an engine-written brain be a script-facing planner. Behaviour is in `action_planner_script_inline.h`. |
| [`action_planner_script_inline.h`](action_planner_script_inline.h.md) | Binds a script-facing brain to a concrete creature by deriving the script facade from it. |
| [`action_script_base.h`](action_script_base.h.md) | Declares the bridge type that lets an engine-written action be installed in a planner that speaks to scripts. Behaviour is in `action_script_base_inline.h`. |
| [`action_script_base_inline.h`](action_script_base_inline.h.md) | Resolves the script-visible game object handed to an engine action back into the concrete client object the action actually needs. |
| [`actor_script.cpp`](actor_script.cpp.md) | Exports the player character and the level-transition trigger to the script virtual machine. |
| [`ActorCondition_script.cpp`](ActorCondition_script.cpp.md) | Exports the condition model — wounds, boosts, and every health and stamina accessor — to the script virtual machine. |
| [`ActorEffector_script.cpp`](ActorEffector_script.cpp.md) | The one piece of the effector system that has to reach into the script virtual machine: telling a script when its camera animation has finished. |
| [`ai_crow_script.cpp`](ai_crow_script.cpp.md) | Exports the crow to the script layer. |
| [`alife_human_brain_script.cpp`](alife_human_brain_script.cpp.md) | Exports the offline human brain to the script layer as a named type with no members. |
| [`alife_monster_brain_script.cpp`](alife_monster_brain_script.cpp.md) | Exports the offline creature brain to scripts: read its movement manager, force an update, and toggle whether it may accept simulation-assigned tasks. |
| [`alife_monster_detail_path_manager_script.cpp`](alife_monster_detail_path_manager_script.cpp.md) | Exports the offline detail-path mover to the script layer. |
| [`alife_monster_movement_manager_script.cpp`](alife_monster_movement_manager_script.cpp.md) | Exports the offline movement arbiter to the script layer. |
| [`alife_monster_patrol_path_manager_script.cpp`](alife_monster_patrol_path_manager_script.cpp.md) | Exports the offline patrol cursor to the script layer. |
| [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md) | The script layer's window onto the whole alife simulation — and the only place that knows how to spawn an entity into a *live* parent rather than an offline one. |
| [`alife_smart_terrain_task_script.cpp`](alife_smart_terrain_task_script.cpp.md) | Exports the job destination to the script layer. |
| [`animation_script_callback.cpp`](animation_script_callback.cpp.md) | Plays a named animation on a game object and fires one script callback when it reaches its end. |
| [`animation_script_callback.h`](animation_script_callback.h.md) | Declares the one-slot animation-end mailbox implemented in `animation_script_callback.cpp`. |
| [`artefact_script.cpp`](artefact_script.cpp.md) | Exports the artefact base class and every shipped artefact type to the script virtual machine. |
| [`base_client_classes_script.cpp`](base_client_classes_script.cpp.md) | Exports the root of the client-object hierarchy to the script virtual machine, so that a script can derive its own game object. |
| [`base_client_classes_wrappers.h`](base_client_classes_wrappers.h.md) | The shim types that let a script class inherit from an engine client class and have the engine call back into it. |
| [`client_spawn_manager_script.cpp`](client_spawn_manager_script.cpp.md) | Exports the spawn-notification registry to the script virtual machine. |
| [`cover_point_script.cpp`](cover_point_script.cpp.md) | Exports the cover point to the script virtual machine. |
| [`CustomOutfit_script.cpp`](CustomOutfit_script.cpp.md) | Exports the body outfit and the helmet to the script virtual machine, with their protection and condition-restore parameters as directly writable fields. |
| [`ef_storage_script.cpp`](ef_storage_script.cpp.md) | Exports the evaluation-function catalogue to Lua as a single overloaded `evaluate` call taking a function name and up to four objects. |
| [`fs_registrator_script.cpp`](fs_registrator_script.cpp.md) | Exports the virtual filesystem to the script virtual machine: path resolution, directory listing with sorting, file existence and metadata, delete/rename/copy, and raw readers and writers. |
| [`game_base_script.cpp`](game_base_script.cpp.md) | Exports the per-player scoreboard record and the game-state base to Lua, including the mode enumeration under both its old and its current names. |
| [`game_cl_base_script.cpp`](game_cl_base_script.cpp.md) | Exports the minimap marker and the respawn point to Lua, so a script-written game mode can draw its own map and place its own spawns. |
| [`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md) | Makes the multiplayer client subclassable from Lua: a script overrides the rules hooks, and the engine drives it exactly as it drives a compiled mode. |
| [`game_cl_mp_script.h`](game_cl_mp_script.h.md) | Declares the script-derivable multiplayer client: the base from which a whole game mode can be written in Lua. Implemented in `game_cl_mp_script.cpp`. |
| [`game_object_space.h`](game_object_space.h.md) | The numbered catalogue of every callback a game object can fire into script — the event vocabulary of the whole script layer. |
| [`game_sv_base_script.cpp`](game_sv_base_script.cpp.md) | Exports the server-side rules to Lua, plus the three numeric vocabularies a script mode needs: player flags, match phases and game events. |
| [`game_sv_deathmatch_script.cpp`](game_sv_deathmatch_script.cpp.md) | Exports the server-side deathmatch rules to Lua as a subclassable type. |
| [`game_sv_mp_script.cpp`](game_sv_mp_script.cpp.md) | Makes the multiplayer server rules subclassable from Lua, and hands scripts the one thing they cannot otherwise reach: the damage numbers inside a hit packet. |
| [`game_sv_mp_script.h`](game_sv_mp_script.h.md) | A multiplayer game mode whose rules are supplied entirely from script: the engine provides the session plumbing and declines to decide anything. |
| [`GameTask_script.cpp`](GameTask_script.cpp.md) | Exports the quest classes to the script layer, and supplies the two tree-building calls that only exist for scripts. |
| [`HairsZone_script.cpp`](HairsZone_script.cpp.md) | Exports three anomaly classes to the script virtual machine. |
| [`helicopter_script.cpp`](helicopter_script.cpp.md) | Exports the helicopter to the script virtual machine: four state enumerations, the flight and targeting commands, and the tuning fields a mission script writes directly. |
| [`level_script.cpp`](level_script.cpp.md) | The `level`, `game`, `relation_registry` and `actor_stats` script namespaces: the general-purpose surface a mod uses to ask about and manipulate the running world. |
| [`login_manager_script.cpp`](login_manager_script.cpp.md) | Exports the multiplayer account session to the script virtual machine, so the main menu can be written in Lua. |
| [`map_script.cpp`](map_script.cpp.md) | Exports the map manager and the map marker to the script virtual machine. |
| [`memory_space_script.cpp`](memory_space_script.cpp.md) | The creature-memory vocabulary as scripts see it: the eight memory record types and the danger record, with their two enumerations. |
| [`mincer_script.cpp`](mincer_script.cpp.md) | Exports two anomaly classes to the script virtual machine. |
| [`MosquitoBald_script.cpp`](MosquitoBald_script.cpp.md) | Exports three anomaly classes to the script virtual machine. |
| [`particle_params_script.cpp`](particle_params_script.cpp.md) | Exports the particle-placement bundle to the script virtual machine, as four constructors and nothing else. |
| [`PhraseDialog_script.cpp`](PhraseDialog_script.cpp.md) | Exports the dialog, phrase and phrase-script records to the script layer, so that a mod can build a conversation at run time instead of authoring it in data. |
| [`PhysicObject_script.cpp`](PhysicObject_script.cpp.md) | Exports the physics prop and its destructible variant to the script virtual machine. |
| [`profile_data_types_script.cpp`](profile_data_types_script.cpp.md) | Exports the profile record shapes to the script layer. |
| [`profile_data_types_script.h`](profile_data_types_script.h.md) | Declares the profile registrator and the callback type an asynchronous profile operation reports through. |
| [`profile_store_script.cpp`](profile_store_script.cpp.md) | Exports the profile store and the two achievement enumerations to the script layer. |
| [`property_evaluator_script.cpp`](property_evaluator_script.cpp.md) | Exports the evaluator base class to the script layer, so planner conditions can be written in Lua. |
| [`property_storage_script.cpp`](property_storage_script.cpp.md) | Exports the planner's answer board to the script layer. |
| [`saved_game_wrapper_script.cpp`](saved_game_wrapper_script.cpp.md) | Exports the save-file peek to the script virtual machine, so the load screen can be written in Lua. |
| [`script_abstract_action.h`](script_abstract_action.h.md) | The one thing every script-issued action has: whether it is finished. |
| [`script_action_condition.h`](script_action_condition.h.md) | Declares the rule for when a script-issued action ends: a set of parts that must all finish, and optionally a time limit. |
| [`script_action_condition_inline.h`](script_action_condition_inline.h.md) | Construction and the start-of-action stamp. |
| [`script_action_condition_script.cpp`](script_action_condition_script.cpp.md) | Exports the action-end condition to the script virtual machine under the name scripts spell it: `cond`. |
| [`script_action_planner_action_wrapper.cpp`](script_action_planner_action_wrapper.cpp.md) | Routes the five overridable points of a nested planner into a script class. |
| [`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md) | Declares the adapter for a script-defined operator that is itself a planner — one step of a plan that expands into a whole sub-plan. |
| [`script_action_planner_action_wrapper_inline.h`](script_action_planner_action_wrapper_inline.h.md) | The nested-planner adapter's constructor. |
| [`script_action_planner_wrapper.cpp`](script_action_planner_wrapper.cpp.md) | Routes a script-defined brain's two points into a script class, and keeps the planner's trace switch in step with the console flag. |
| [`script_action_planner_wrapper.h`](script_action_planner_wrapper.h.md) | Declares the adapter that lets a script define a creature's whole brain. |
| [`script_action_planner_wrapper_inline.h`](script_action_planner_wrapper_inline.h.md) | Empty. |
| [`script_action_wrapper.cpp`](script_action_wrapper.cpp.md) | Routes a planner operator's five overridable points into a script class, and refuses to let a script quote a cost that would break the search's admissibility. |
| [`script_action_wrapper.h`](script_action_wrapper.h.md) | Declares the adapter that lets a script define a planner operator — an action with preconditions, effects and a cost. |
| [`script_action_wrapper_inline.h`](script_action_wrapper_inline.h.md) | The script operator adapter's constructor. |
| [`script_animation_action.h`](script_animation_action.h.md) | Declares the animation part of a script-issued action: play this clip, or just adopt this mental posture. |
| [`script_animation_action_inline.h`](script_animation_action_inline.h.md) | The setters, and the one decision they encode: naming a clip makes the action wait, naming a posture does not. |
| [`script_animation_action_script.cpp`](script_animation_action_script.cpp.md) | Exports the animation part to the script virtual machine under the name scripts spell it: `anim`. |
| [`script_bind_macroses.h`](script_bind_macroses.h.md) | The generator for the game object's script facade: every exported method is a downcast to a concrete class, an error if the cast fails, and a forwarded call. |
| [`script_binder.cpp`](script_binder.cpp.md) | Attaches a script-authored behaviour to a game object from its configuration, forwards the object's whole lifecycle to it, and detaches it rather than propagating any failure. |
| [`script_binder.h`](script_binder.h.md) | Declares the engine-side half of the script binding: the mixin every game object carries so a script can attach behaviour to it. |
| [`script_binder_inline.h`](script_binder_inline.h.md) | Reads the binder attached to a game object. |
| [`script_binder_object.cpp`](script_binder_object.cpp.md) | The base class a script author subclasses to attach their own state and lifecycle to an existing game object. |
| [`script_binder_object.h`](script_binder_object.h.md) | Declares the script-side binder base class. |
| [`script_binder_object_script.cpp`](script_binder_object_script.cpp.md) | Exports the binder base class to the script layer under the name `object_binder`. |
| [`script_binder_object_wrapper.cpp`](script_binder_object_wrapper.cpp.md) | Routes each binder lifecycle hook into the script object's method of the same name, and back out to the base implementation. |
| [`script_binder_object_wrapper.h`](script_binder_object_wrapper.h.md) | Declares the script-override adapter for the binder base class. |
| [`script_effector.cpp`](script_effector.cpp.md) | A post-processing effector whose per-frame parameter evaluation is written in script. |
| [`script_effector.h`](script_effector.h.md) | Declares the script-authored post-process effector. |
| [`script_effector_inline.h`](script_effector_inline.h.md) | Constructs a script effector of a given slot type and duration. |
| [`script_effector_script.cpp`](script_effector_script.cpp.md) | Exports the post-process parameter block and the script effector to the script layer. |
| [`script_effector_wrapper.cpp`](script_effector_wrapper.cpp.md) | Routes the effector's per-frame evaluation into the script object's `process` method. |
| [`script_effector_wrapper.h`](script_effector_wrapper.h.md) | Declares the script-override adapter for the post-process effector. |
| [`script_effector_wrapper_inline.h`](script_effector_wrapper_inline.h.md) | Forwards the effector wrapper's construction to the effector it wraps. |
| [`script_entity.cpp`](script_entity.cpp.md) | The action queue that lets a script drive a creature directly, bypassing its brain. |
| [`script_entity.h`](script_entity.h.md) | Declares the script-driven action queue mixin. |
| [`script_entity_action.cpp`](script_entity_action.cpp.md) | Nothing: the scripted-action type is entirely inline. |
| [`script_entity_action.h`](script_entity_action.h.md) | One scripted action: eight simultaneous demands on a creature plus the rule that says when they are jointly done. |
| [`script_entity_action_inline.h`](script_entity_action_inline.h.md) | The bodies of the scripted-action type: channel assignment, per-channel completion, and the joint completion rule. |
| [`script_entity_action_script.cpp`](script_entity_action_script.cpp.md) | Exports the scripted-action type to the script layer under the name `entity_action`. |
| [`script_entity_inline.h`](script_entity_inline.h.md) | Reads the entity the action-queue mixin is half of. |
| [`script_entity_space.h`](script_entity_space.h.md) | Names the six channels a scripted action can drive, plus the retirement event. |
| [`script_game_object.cpp`](script_game_object.cpp.md) | The first slice of the game object facade: transform, condition, the action queue, weapon ammunition, and the entry points that open the actor's menu. |
| [`script_game_object.h`](script_game_object.h.md) | Declares the game object — the single facade through which every script reaches every entity in the world. |
| [`script_game_object2.cpp`](script_game_object2.cpp.md) | Explosives, the item-handling goal, animation cycles, applying a script-built hit, the creature memory surface, actor placement, and the thresholds that decide which enemies a stalker bothers with. |
| [`script_game_object3.cpp`](script_game_object3.cpp.md) | Cover selection, the creature movement and sight surfaces, scripted animations, trade tuning, anomalies, artefacts, held-item state, bones and restrictors — the widest of the nine facade slices. |
| [`script_game_object4.cpp`](script_game_object4.cpp.md) | The creature sound player, the wounded and sight states, inventory boxes, bone-attached particles, the absolute health write, and the flat table of class predicates. |
| [`script_game_object_impl.h`](script_game_object_impl.h.md) | Turns "the script is holding a destroyed object" from a crash into a named script error. |
| [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) | Information portions, dialogue and the talk screen, inventory movement and money, the whole relations model, character identity, tasks, restrictors, doors, weapon attachments, item upgrades and the actor's weig … |
| [`script_game_object_script.cpp`](script_game_object_script.cpp.md) | Assembles the game object's script registration from the chain of partial registrations, and declares the two constant namespaces that go with it: the sight parameters and the callback event names. |
| [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) | The first half of the game object's script surface: the condition properties, identity, the action queue, the memory and movement vocabularies, and the smart-cover methods. |
| [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md) | The second half of the game object's script surface: sounds, sight, restrictors, information portions, tasks, news, relations, anomalies, and the explicit downcast family. |
| [`script_game_object_script_trader.cpp`](script_game_object_script_trader.cpp.md) | Adds the trader presentation methods to the game object's script registration. |
| [`script_game_object_smart_covers.cpp`](script_game_object_smart_covers.cpp.md) | The facade's smart-cover surface: choosing a cover and a loophole to occupy, selecting what to do while in it, aiming out of it, and retuning the dwell times that make the behaviour look deliberate. |
| [`script_game_object_trader.cpp`](script_game_object_trader.cpp.md) | The five facade methods that drive a trader's idle presentation: its body animation, its head animation, and the speech played over them. |
| [`script_game_object_use.cpp`](script_game_object_use.cpp.md) | The facade's construction and destruction, its identity and death, faction relations, the callback installation surface, and the two entry points that let a script push on the physics world. |
| [`script_game_object_use2.cpp`](script_game_object_use2.cpp.md) | The per-species monster controls: the levers a script pulls on a bloodsucker, a burer, a poltergeist or a zombie that no other creature has, plus the two perception snapshots every monster can hand back. |
| [`script_hit.cpp`](script_hit.cpp.md) | Nothing: the script hit is entirely inline. |
| [`script_hit.h`](script_hit.h.md) | Declares the script-authored hit: a damage event a script builds by hand and applies to an entity. |
| [`script_hit_inline.h`](script_hit_inline.h.md) | The script hit's defaults, its copy, and its bone setter. |
| [`script_hit_script.cpp`](script_hit_script.cpp.md) | Exports the script hit to the script layer as `hit`, together with the frozen table of damage kinds. |
| [`script_monster_action.cpp`](script_monster_action.cpp.md) | The one line of the monster action channel that cannot be inline: unwrapping a script facade into the client object it fronts. |
| [`script_monster_action.h`](script_monster_action.h.md) | Declares the monster-behaviour channel of a scripted action: a coarse global behaviour plus an optional target. |
| [`script_monster_action_inline.h`](script_monster_action_inline.h.md) | The two behaviour-naming constructors of the monster action channel. |
| [`script_monster_action_script.cpp`](script_monster_action_script.cpp.md) | Exports the monster behaviour channel to the script layer as `act`. |
| [`script_monster_hit_info.h`](script_monster_hit_info.h.md) | The record a monster hands to script when asked who last hit it, from where, and when. |
| [`script_monster_hit_info_script.cpp`](script_monster_hit_info_script.cpp.md) | Exports the monster hit record as `MonsterHitInfo`, and a namespace object carrying the monster-facing constant tables. |
| [`script_movement_action.cpp`](script_movement_action.cpp.md) | The movement channel's constructors that decide something: unpacking a patrol path parameter block, deriving a goal kind from a monster move action, and the pathless goal. |
| [`script_movement_action.h`](script_movement_action.h.md) | Declares the movement channel of a scripted action: where an entity should go, in what posture, by what kind of path. |
| [`script_movement_action_inline.h`](script_movement_action_inline.h.md) | The movement channel's setters and its simpler constructors — and the rule that any change reopens the channel. |
| [`script_movement_action_script.cpp`](script_movement_action_script.cpp.md) | Exports the movement channel to the script layer as `move`, with the six constant tables and the twenty constructor forms scripts address it by. |
| [`script_object.cpp`](script_object.cpp.md) | Composes the client-object lifecycle with the scripted-entity mixin's, and fixes the order in which the two halves run. |
| [`script_object.h`](script_object.h.md) | Declares the plain scripted entity: a client object whose only behaviour is whatever the action queue tells it to do. |
| [`script_object_action.cpp`](script_object_action.cpp.md) | The one method of the object channel that cannot be inline: unwrapping a script facade into the client object it fronts. |
| [`script_object_action.h`](script_object_action.h.md) | Declares the object channel of a scripted action: what an entity should do with a thing it is holding, or with one of its own bones. |
| [`script_object_action_inline.h`](script_object_action_inline.h.md) | The object channel's constructors and setters. |
| [`script_object_action_script.cpp`](script_object_action_script.cpp.md) | Exports the object channel to the script layer as `object`, with the frozen table of item-handling orders. |
| [`script_particle_action.cpp`](script_particle_action.cpp.md) | Creates the live particle effect the channel owns — and does not destroy it. |
| [`script_particle_action.h`](script_particle_action.h.md) | Declares the particle channel of a scripted action: which effect to play, attached to a bone or standing at a place. |
| [`script_particle_action_inline.h`](script_particle_action_inline.h.md) | The particle channel's constructors and setters — and the order dependence that decides whether the effect follows a bone or stands still. |
| [`script_particle_action_script.cpp`](script_particle_action_script.cpp.md) | Exports the particle channel to the script layer as `particle`. |
| [`script_particles.cpp`](script_particles.cpp.md) | A script-owned particle effect: its transform, its optional path animation, and the two-way link that lets either side die first. |
| [`script_particles.h`](script_particles.h.md) | Declares the standalone script-owned particle effect: one a script creates, positions, animates along an authored path and stops, with no entity behind it. |
| [`script_particles_inline.h`](script_particles_inline.h.md) | Nothing: an empty companion header. |
| [`script_particles_script.cpp`](script_particles_script.cpp.md) | Exports the standalone particle effect to the script layer as `particles_object`. |
| [`script_property_evaluator_wrapper.cpp`](script_property_evaluator_wrapper.cpp.md) | Routes a planner evaluator's two overridable points into a script class, and decides what a misbehaving script evaluator answers. |
| [`script_property_evaluator_wrapper.h`](script_property_evaluator_wrapper.h.md) | Declares the adapter that lets a script define a planner evaluator — one question about the world the planner can ask. |
| [`script_property_evaluator_wrapper_inline.h`](script_property_evaluator_wrapper_inline.h.md) | The script evaluator adapter's constructor. |
| [`script_sound.cpp`](script_sound.cpp.md) | Creating a script-owned emitter, the three ways to start it, and what happens when a script drops one that is still playing. |
| [`script_sound.h`](script_sound.h.md) | Declares the script-owned sound: an audio emitter a script constructs by name, plays, tunes and destroys by value. |
| [`script_sound_action.cpp`](script_sound_action.cpp.md) | Naming the sound a channel will play, and deciding on the spot that a missing one is already finished. |
| [`script_sound_action.h`](script_sound_action.h.md) | Declares the sound channel of a scripted action: a sound to play from a bone or a place, with monster and trader variants folded into the same record. |
| [`script_sound_action_inline.h`](script_sound_action_inline.h.md) | The sound channel's seven constructors and its setters — where the mode and the goal kind are decided by statement order. |
| [`script_sound_action_script.cpp`](script_sound_action_script.cpp.md) | Exports the sound channel as `sound`, and the engine's whole AI-perception sound vocabulary as `snd_type`. |
| [`script_sound_info.h`](script_sound_info.h.md) | The record a creature's sound-perception callback hands to a script: who made the noise, where, how loud, when, and whether it counts as dangerous. |
| [`script_sound_info_script.cpp`](script_sound_info_script.cpp.md) | Exports the heard-sound record to the script virtual machine as `SoundInfo`. |
| [`script_sound_inline.h`](script_sound_inline.h.md) | The pass-through half of the script-owned sound handle: every property a script can read or set on a playing sound, forwarded to the audio emitter. |
| [`script_sound_script.cpp`](script_sound_script.cpp.md) | Exports the script-owned sound handle and the emitter parameter block to the script virtual machine. |
| [`script_watch_action.cpp`](script_watch_action.cpp.md) | The one look-order setter that has to reach through the script facade: naming an object to watch. |
| [`script_watch_action.h`](script_watch_action.h.md) | Declares the look-at order a script attaches to an entity action: what to aim the head (or a searchlight) at. |
| [`script_watch_action_inline.h`](script_watch_action_inline.h.md) | The constructors and setters of a look order: each fixes which of the four goal kinds the order is, and clears its completion flag. |
| [`script_watch_action_script.cpp`](script_watch_action_script.cpp.md) | Exports the look order to the script virtual machine as `look`, together with the script-facing names of the sight types. |
| [`script_zone.cpp`](script_zone.cpp.md) | The scripted trigger volume: a restrictor shape that tracks which entities are inside it and calls into Lua when the set changes. |
| [`script_zone.h`](script_zone.h.md) | Declares the scripted trigger volume: a restrictor that fires enter and exit callbacks into Lua. |
| [`script_zone_script.cpp`](script_zone_script.cpp.md) | Exports the scripted trigger volume and the smart zone to the script virtual machine as spawnable entity classes. |
| [`ScriptXMLInit.cpp`](ScriptXMLInit.cpp.md) | The script layer's window factory: a Lua-visible object that holds one parsed UI layout document and builds a typed widget from any node in it, attaching it to a parent that then owns it. |
| [`ScriptXMLInit.h`](ScriptXMLInit.h.md) | Declares the script-visible widget factory implemented in `ScriptXMLInit.cpp`. |
| [`server_entity_wrapper.cpp`](server_entity_wrapper.cpp.md) | Writes and reads one server object as a self-contained two-chunk record, by replaying the network spawn and update messages into a file. |
| [`server_entity_wrapper.h`](server_entity_wrapper.h.md) | Declares a serializable envelope that lets one server object be written to and read from a stream on its own. |
| [`server_entity_wrapper_inline.h`](server_entity_wrapper_inline.h.md) | Construction and access for the server-object envelope. |
| [`smart_cover_object_script.cpp`](smart_cover_object_script.cpp.md) | Exports the smart cover object to the script layer as a game object subtype with no members of its own. |
| [`space_restrictor_script.cpp`](space_restrictor_script.cpp.md) | Exports the restrictor to the script virtual machine, so that scripts can recognize a restrictor game object by type. |
| [`stalker_animation_script.cpp`](stalker_animation_script.cpp.md) | The script-animation queue: how a script's request to play an animation is validated, queued, started, and advanced when one finishes. |
| [`stalker_animation_script.h`](stalker_animation_script.h.md) | One entry in a stalker's script-animation queue: a motion the script layer asked for, plus how it should be played. |
| [`stalker_animation_script_inline.h`](stalker_animation_script_inline.h.md) | Construction, copying and reading of a queued script animation — where the optional destination is actually made optional. |
| [`torch_script.cpp`](torch_script.cpp.md) | Exports the torch, the personal data assistant and the four detector variants to the script layer, as constructible classes and nothing more. |
| [`ui_export_script.cpp`](ui_export_script.cpp.md) | Exports the main menu and its multiplayer screens to the script layer — the surface the entire front end is written against. |
| [`UIGame_custom_script.cpp`](UIGame_custom_script.cpp.md) | Lets a script define its own game UI layer by subclassing the engine's. |
| [`UIGame_custom_script.h`](UIGame_custom_script.h.md) | Declares the script-derivable game UI registered in `UIGame_custom_script.cpp`. |
| [`UIGameCustom_script.cpp`](UIGameCustom_script.cpp.md) | The in-game interface as scripts see it: put a named piece of text on the screen, open or close the inventory and the handheld computer, and reach the whole thing from anywhere. |
| [`wrapper_abstract.h`](wrapper_abstract.h.md) | The adapter that lets a planner evaluator or operator written against a concrete creature be constructed from either the creature or its script facade. |
| [`wrapper_abstract_inline.h`](wrapper_abstract_inline.h.md) | The two bind paths of the evaluator wrapper: from the concrete creature down to the facade, and from the facade up to the concrete creature. |

### The alife simulation

| File | Role |
|---|---|
| [`alife_abstract_registry.h`](alife_abstract_registry.h.md) | The shape every alife registry shares: a keyed table of records that saves and loads with the game, and that asserts loudly when a key is added twice or looked up and missing. |
| [`alife_abstract_registry_inline.h`](alife_abstract_registry_inline.h.md) | The generic alife registry's five operations. |
| [`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md) | The server-side anomaly: it draws its own strength at spawn time and populates itself with artefacts by weighted lottery, so the artefacts a player eventually finds were decided before the level ever loaded. |
| [`alife_combat_manager.cpp`](alife_combat_manager.cpp.md) | Where off-screen combat was resolved, and now is not: all that survives is the routine that turns a creature killed off-screen into a lootable corpse in the right place. |
| [`alife_combat_manager.h`](alife_combat_manager.h.md) | Declares the off-screen combat layer of the simulation, almost all of it disabled. |
| [`alife_combat_manager_inline.h`](alife_combat_manager_inline.h.md) | Empty. |
| [`alife_communication_manager.cpp`](alife_communication_manager.cpp.md) | Where two off-screen characters meeting on the world graph would have traded with each other, item for item, by solving a subset-sum problem over both inventories. The whole model is disabled … |
| [`alife_communication_manager.h`](alife_communication_manager.h.md) | Declares the off-screen trading layer of the simulation; everything but the constructor is disabled. |
| [`alife_communication_manager_inline.h`](alife_communication_manager_inline.h.md) | Empty. |
| [`alife_communication_space.h`](alife_communication_space.h.md) | Empty. |
| [`alife_creature_abstract.cpp`](alife_creature_abstract.cpp.md) | What every server-side creature settles at spawn: its faction's team, its restrictor sets, and the death timestamp of a creature that was authored already dead. |
| [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md) | The online/offline boundary: what a server record does when it is promoted to a live simulated object, what it does on the way back, and how it decides which side of the boundary it should be on at all. |
| [`alife_graph_registry.cpp`](alife_graph_registry.cpp.md) | The index of the whole offline world by cross-level graph vertex, the trigger that loads a level when the player's record first appears … |
| [`alife_graph_registry.h`](alife_graph_registry.h.md) | Declares the offline world's index by game graph vertex, its terrain transpose and its level-scoped subset. |
| [`alife_graph_registry_inline.h`](alife_graph_registry_inline.h.md) | Moving an object between graph vertices, seeding a creature's travel along a graph edge, and the registry's accessors. |
| [`alife_group_abstract.cpp`](alife_group_abstract.cpp.md) | A squad that exists offline as one record and online as several creatures: this is the expansion and the collapse, plus the rule that a member who dies leaves the group and becomes an individual. |
| [`alife_group_registry.cpp`](alife_group_registry.cpp.md) | The index of every online/offline group in the world, so the simulation can find and re-initialize them after a save is loaded. |
| [`alife_group_registry.h`](alife_group_registry.h.md) | Declares the index of online/offline groups. |
| [`alife_group_registry_inline.h`](alife_group_registry_inline.h.md) | The group registry's one accessor. |
| [`alife_human_abstract.cpp`](alife_human_abstract.cpp.md) | The offline human: a server record that is simultaneously a creature and a trading party, and that forwards every behavioural question to its brain. |
| [`alife_human_brain_save.h`](alife_human_brain_save.h.md) | A disabled file kept on purpose: the offline ammunition model, the meet-or-fight decision, and how a squad divided loot. Not compiled; the header comment forbids deleting it. |
| [`alife_human_object_handler.cpp`](alife_human_object_handler.cpp.md) | The offline inventory manager, stubbed: every method answers "nothing" or does nothing, so an off-screen character never re-equips, never picks anything up and never chooses a weapon. |
| [`alife_human_object_handler.h`](alife_human_object_handler.h.md) | Declares the offline inventory manager for a human server record. |
| [`alife_human_object_handler_inline.h`](alife_human_object_handler_inline.h.md) | The offline inventory manager's binding to its record. |
| [`alife_human_object_handler_save.h`](alife_human_object_handler_save.h.md) | The real offline inventory manager, preserved and not compiled: how an off-screen character noticed items lying on the world graph, decided what to keep, merged its ammunition and weighed what it could carry. |
| [`alife_interaction_manager.cpp`](alife_interaction_manager.cpp.md) | Where two off-screen entities meeting on the world graph would have been made to fight, trade or touch a smart terrain. Disabled; only the constructor remains. |
| [`alife_interaction_manager.h`](alife_interaction_manager.h.md) | Declares the layer that joins off-screen combat and off-screen trading over one simulation state. |
| [`alife_interaction_manager_inline.h`](alife_interaction_manager_inline.h.md) | Empty. |
| [`alife_level_registry.h`](alife_level_registry.h.md) | The subset of the offline world that lies on the currently loaded level: a mutation-safe table the simulation walks every frame while its own callbacks add and remove entries. |
| [`alife_level_registry_inline.h`](alife_level_registry_inline.h.md) | The level subset's operations: a filter on the way in, a pass-through on the way out. |
| [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) | The offline creature: its registration and promotion hooks, the rule that decides when a corpse may finally be forgotten, and the population model that lets a pack breed while nobody is watching. |
| [`alife_monster_base.cpp`](alife_monster_base.cpp.md) | A creature's authored loot drop, decided once when it spawns, and the promotion hooks that route around its inheritance. |
| [`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md) | Walks an off-screen creature along the cross-level graph toward a destination, consuming game time at its travel speed and stepping it from vertex to vertex — the off-screen equivalent of walking. |
| [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) | Declares the offline mover that walks an alife creature along a game-graph path, implemented in `alife_monster_detail_path_manager.cpp`. |
| [`alife_monster_detail_path_manager_inline.h`](alife_monster_detail_path_manager_inline.h.md) | The trivial accessors of the offline detail-path mover, split out of the header purely as a C++ habit. |
| [`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md) | The offline creature's movement brain: chooses between free travel to a destination and following an authored patrol path, and feeds whichever it chose into the detail mover each alife tick. |
| [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) | Declares the offline creature's movement arbiter, implemented in `alife_monster_movement_manager.cpp`. |
| [`alife_monster_movement_manager_inline.h`](alife_monster_movement_manager_inline.h.md) | The accessors of the offline movement arbiter, split out of the header as a C++ habit. |
| [`alife_monster_patrol_path_manager.cpp`](alife_monster_patrol_path_manager.cpp.md) | Walks an offline creature around an authored patrol path: picks where to join, decides which branch to take at each point, and decides what happens at a dead end. |
| [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md) | Declares the offline patrol-path cursor, implemented in `alife_monster_patrol_path_manager.cpp`. |
| [`alife_monster_patrol_path_manager_inline.h`](alife_monster_patrol_path_manager_inline.h.md) | Accessors of the offline patrol cursor — mostly trivial, but one of them carries a real rule. |
| [`alife_object.cpp`](alife_object.cpp.md) | The game-side half of the base alife server object: turning an entity's authored "what I am carrying" text into actual spawned items. |
| [`alife_object_registry.cpp`](alife_object_registry.cpp.md) | The master table of every alife server object in the world, and the save/load of that table as a parent-first tree of spawn-plus-update packets. |
| [`alife_object_registry.h`](alife_object_registry.h.md) | Declares the master table of alife server objects, implemented in `alife_object_registry.cpp` and `alife_object_registry_inline.h`. |
| [`alife_object_registry_inline.h`](alife_object_registry_inline.h.md) | The table operations of the alife object registry: insert with a duplicate check, remove, and look up with an optional tolerance for absence. |
| [`alife_online_offline_group.cpp`](alife_online_offline_group.cpp.md) | The squad: a server object that absorbs a set of creatures so the whole group travels the world as one record while offline, and dissolves into individually simulated creatures while online. |
| [`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md) | The squad's offline decision loop: ask the smart terrain what this squad is supposed to be doing, and walk there. |
| [`alife_online_offline_group_brain.h`](alife_online_offline_group_brain.h.md) | Declares the squad's offline brain, implemented in `alife_online_offline_group_brain.cpp`. |
| [`alife_online_offline_group_brain_inline.h`](alife_online_offline_group_brain_inline.h.md) | The two accessors of the squad brain. |
| [`alife_registry_container.cpp`](alife_registry_container.cpp.md) | Saves and loads the whole bundle of per-character persistent registries as one chunk, in one fixed order. |
| [`alife_registry_container.h`](alife_registry_container.h.md) | Declares the bundle of per-character persistent registries and the type-keyed lookup that picks one out of it. |
| [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) | The authoritative list of persistent per-character registries — which stores exist, what each one keys and holds, and the order they appear in a saved game. |
| [`alife_registry_container_inline.h`](alife_registry_container_inline.h.md) | The type-keyed selector that picks one registry out of the bundle, checked at compile time. |
| [`alife_registry_container_space.h`](alife_registry_container_space.h.md) | The four macros that let the composition file be written as a flat list of registries. |
| [`alife_registry_wrapper.h`](alife_registry_wrapper.h.md) | A per-owner handle onto one of the persistent registries, with a private fallback store for the case where there is no alife simulation at all. |
| [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) | Names one wrapper type per persistent registry, so a game object can hold "my news feed" or "my relations" as a field. |
| [`alife_schedule_registry.cpp`](alife_schedule_registry.cpp.md) | Membership rules for the offline update rotation: which server objects get an alife tick at all. |
| [`alife_schedule_registry.h`](alife_schedule_registry.h.md) | The round-robin that gives every offline entity an alife tick, a bounded number of entities per pass. |
| [`alife_schedule_registry_inline.h`](alife_schedule_registry_inline.h.md) | One pass of the offline round-robin, and the budget it is bounded by. |
| [`alife_simulator.cpp`](alife_simulator.cpp.md) | The alife simulation's top object: constructing it starts a game, destroying it ends one. |
| [`alife_simulator.h`](alife_simulator.h.md) | Declares the concrete alife simulator, implemented in `alife_simulator.cpp`. |
| [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) | Owns every alife registry, and owns entity creation: how a spawn record, a configuration section or a group template becomes a live server object. |
| [`alife_simulator_base.h`](alife_simulator_base.h.md) | Declares the layer that owns the alife registries and entity creation, implemented in `alife_simulator_base.cpp` and `alife_simulator_base2.cpp`. |
| [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) | Registration and deregistration: the exact order in which an entity enters and leaves every alife registry, and what happens when one dies. |
| [`alife_simulator_base_inline.h`](alife_simulator_base_inline.h.md) | The accessors for the alife simulation's eleven sub-objects, each guarded by the same initialization check. |
| [`alife_simulator_header.cpp`](alife_simulator_header.cpp.md) | The save file's version stamp: written first, checked before anything else is read, and the reason a mismatched save is refused rather than guessed at. |
| [`alife_simulator_header.h`](alife_simulator_header.h.md) | Declares the save-format version stamp, implemented in `alife_simulator_header.cpp`. |
| [`alife_simulator_header_inline.h`](alife_simulator_header_inline.h.md) | Construction and the version accessor. |
| [`alife_smart_terrain_registry.cpp`](alife_smart_terrain_registry.cpp.md) | Membership of the smart-terrain index: which server objects are places that hand out jobs. |
| [`alife_smart_terrain_registry.h`](alife_smart_terrain_registry.h.md) | Declares the index of smart terrains, implemented in `alife_smart_terrain_registry.cpp`. |
| [`alife_smart_terrain_registry_inline.h`](alife_smart_terrain_registry_inline.h.md) | The two read accessors of the smart-terrain index. |
| [`alife_smart_terrain_task.cpp`](alife_smart_terrain_task.cpp.md) | A destination, stated either as a named patrol point or as a pair of navigation vertices, and resolved to the triple an offline mover needs. |
| [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) | Declares the job destination, implemented in `alife_smart_terrain_task.cpp`. |
| [`alife_smart_terrain_task_inline.h`](alife_smart_terrain_task_inline.h.md) | Construction of a job destination: five spellings collapsing to two forms. |
| [`alife_smart_zone.cpp`](alife_smart_zone.cpp.md) | How a smart terrain behaves when the offline simulation walks something into it: it is always awake, it never fights, and meeting it means asking it for a job. |
| [`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) | Owns the level's spawn file: loads it, verifies it matches the game graph and the save, and keeps the authored spawn records available for the whole session. |
| [`alife_spawn_registry.h`](alife_spawn_registry.h.md) | Declares the owner of the spawn file, implemented in `alife_spawn_registry.cpp` and `alife_spawn_registry_spawn.cpp`. |
| [`alife_spawn_registry_header.cpp`](alife_spawn_registry_header.cpp.md) | The spawn file's first chunk: the format version and the two identifiers that tie a spawn file, a game graph and a saved game together. |
| [`alife_spawn_registry_header.h`](alife_spawn_registry_header.h.md) | Declares the spawn file's header record, implemented in `alife_spawn_registry_header.cpp`. |
| [`alife_spawn_registry_header_inline.h`](alife_spawn_registry_header_inline.h.md) | The five read accessors of the spawn file's header. |
| [`alife_spawn_registry_inline.h`](alife_spawn_registry_inline.h.md) | Artefact placement inside an anomaly, and two small helpers the spawn walk depends on. |
| [`alife_spawn_registry_spawn.cpp`](alife_spawn_registry_spawn.cpp.md) | The spawn walk: descending the authored spawn graph, rolling its edge weights, and deciding which records become entities. |
| [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) | Writes and reads a saved game: the chunk order, the compression wrapper, and the multi-pass restore that makes thousands of entities come back consistent. |
| [`alife_storage_manager.h`](alife_storage_manager.h.md) | Declares the save/load layer of the alife simulator, implemented in `alife_storage_manager.cpp`. |
| [`alife_storage_manager_inline.h`](alife_storage_manager_inline.h.md) | Construction of the save/load layer. |
| [`alife_story_registry.cpp`](alife_story_registry.cpp.md) | The index that lets a script address an entity by the name a level designer gave it, rather than by a runtime identifier nobody can predict. |
| [`alife_story_registry.h`](alife_story_registry.h.md) | Declares the story-identifier index, implemented in `alife_story_registry.cpp` and `alife_story_registry_inline.h`. |
| [`alife_story_registry_inline.h`](alife_story_registry_inline.h.md) | Lookup and removal in the story index, and the shared shape of "absent is usually a bug". |
| [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md) | Repopulation: work out which authored spawn records have no live entity, run the spawn walk over them, and instantiate what comes back. |
| [`alife_surge_manager.h`](alife_surge_manager.h.md) | Declares the repopulation layer, implemented in `alife_surge_manager.cpp`. |
| [`alife_surge_manager_inline.h`](alife_surge_manager_inline.h.md) | Construction of the repopulation layer. |
| [`alife_switch_manager.cpp`](alife_switch_manager.cpp.md) | Promotion and demotion: turning an offline record into a live client object and back, without losing state and without leaving an entity registered twice. |
| [`alife_switch_manager.h`](alife_switch_manager.h.md) | Declares the online/offline transition layer, implemented in `alife_switch_manager.cpp` and `alife_switch_manager_inline.h`. |
| [`alife_switch_manager_inline.h`](alife_switch_manager_inline.h.md) | Where the online and offline radii come from: one authored distance and one hysteresis fraction. |
| [`alife_time_manager.cpp`](alife_time_manager.cpp.md) | The game clock: an in-world calendar that advances at a configurable multiple of real time, with the multiplier changeable mid-game without the clock jumping. |
| [`alife_time_manager.h`](alife_time_manager.h.md) | Declares the game clock, implemented in `alife_time_manager.cpp` and `alife_time_manager_inline.h`. |
| [`alife_time_manager_inline.h`](alife_time_manager_inline.h.md) | The derived clock reading and its three mutations — the file that actually defines what game time is. |
| [`alife_trader.cpp`](alife_trader.cpp.md) | The concrete trader server object: an entity that is both a character and a container, and the one place a price is asked for. |
| [`alife_trader_abstract.cpp`](alife_trader_abstract.cpp.md) | What it means, on the server side, for an entity to carry an inventory: how its contents are created, and the two hand-offs that move a container's whole inventory across the online/offline boundary. |
| [`alife_update_manager.cpp`](alife_update_manager.cpp.md) | The alife simulation's heartbeat: the one place per frame that decides which offline entities come online, advances the offline world under a time budget, and performs the two world-scale operations … |
| [`alife_update_manager.h`](alife_update_manager.h.md) | Declares the top of the alife simulator: the per-frame tick, the world-scale lifecycle operations, and the script-facing verbs, implemented in `alife_update_manager.cpp`. |

### Server objects and spawn records

| File | Role |
|---|---|
| [`xrServer.cpp`](xrServer.cpp.md) | The authoritative world's heartbeat: the per-frame update, the table of every entity, the table of every client, and the switch that decides what a received message is allowed to do. |
| [`xrServer.h`](xrServer.h.md) | Declares the authoritative side of the world: the entity table, the client table, the identifier allocator, and every message the server answers. |
| [`xrServer_balance.cpp`](xrServer_balance.cpp.md) | Chooses which client should take over simulating an entity — and, as shipped, always answers "the host". |
| [`xrServer_CL_connect.cpp`](xrServer_CL_connect.cpp.md) | Admits one client: the ordered gauntlet of checks it must pass, and then the replay of the entire world into its lap. |
| [`xrServer_CL_disconnect.cpp`](xrServer_CL_disconnect.cpp.md) | Decides what happens to everything a departing client was simulating: hand it to somebody else, or destroy the world. |
| [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md) | Brings a server up for one session — choosing the rules from the session string — and walks a newly arrived client through the gates it must pass before it is admitted. |
| [`xrServer_Disconnect.cpp`](xrServer_Disconnect.cpp.md) | Shuts one session down, in the order that leaves nothing holding a reference to something already gone. |
| [`xrServer_info.cpp`](xrServer_info.cpp.md) | Sends a joining client the server's logo and rules text over the file-transfer channel, and uses the transfer's completion as the signal that configuration is finished. |
| [`xrServer_info.h`](xrServer_info.h.md) | Declares the one-shot uploader that pushes a server's logo and rules text to a joining client and then reports done. |
| [`xrServer_perform_GameExport.cpp`](xrServer_perform_GameExport.cpp.md) | Pushes the whole game-rule state — scores, teams, round, limits — to every admitted client at once. |
| [`xrServer_perform_migration.cpp`](xrServer_perform_migration.cpp.md) | Moving an entity's simulation from one client to another — written, disabled, and worth reading for the protocol it names. |
| [`xrServer_perform_RPgen.cpp`](xrServer_perform_RPgen.cpp.md) | Where an entity's spawn position would have been chosen from the level's respawn points — abandoned, and now always a pass. |
| [`xrServer_perform_sls_default.cpp`](xrServer_perform_sls_default.cpp.md) | Populates a level from its authored spawn file when there is no save to load — and, for a developer, makes sure there is somebody to play as. |
| [`xrServer_perform_sls_load.cpp`](xrServer_perform_sls_load.cpp.md) | Reads a saved level back: every entity is spawned from its record and then updated from the next one, in the order the file holds them. |
| [`xrServer_perform_sls_save.cpp`](xrServer_perform_sls_save.cpp.md) | Writes the level's entire server-side state to a save: for each entity, the record that creates it and the record that brings it up to date. |
| [`xrServer_perform_transfer.cpp`](xrServer_perform_transfer.cpp.md) | Moves an item from one container to another, or drops it — and does it as a pair of timestamped events the clients replay in order. |
| [`xrServer_process_event.cpp`](xrServer_process_event.cpp.md) | The one gate through which every gameplay act reaches the authoritative world: a timestamped, addressed event, sorted into the handful of things the server is willing to do about it. |
| [`xrServer_process_event_activate.cpp`](xrServer_process_event_activate.cpp.md) | Lets the game rules approve an artefact being switched on, and tells everyone — provided the thing is actually in somebody's hands. |
| [`xrServer_process_event_destroy.cpp`](xrServer_process_event_destroy.cpp.md) | Destroys an entity and everything inside it, and folds the whole cascade into one packet so that clients see it happen atomically. |
| [`xrServer_process_event_ownership.cpp`](xrServer_process_event_ownership.cpp.md) | Decides whether one entity may take another — the gate every pickup, purchase and forced hand-out passes through. |
| [`xrServer_process_event_reject.cpp`](xrServer_process_event_reject.cpp.md) | Detaches an item from its owner — the inverse of taking, and the step that has to succeed before anything can be destroyed or handed on. |
| [`xrServer_process_spawn.cpp`](xrServer_process_spawn.cpp.md) | Turns a spawn record into a live entity: allocates its identifier, refuses it if the game mode will not have it, attaches it to its parent, and tells everybody — twice, differently. |
| [`xrServer_process_update.cpp`](xrServer_process_update.cpp.md) | Applies a batch of per-entity state to the server's records, and detects the moment a sender and a receiver disagree about a record's shape. |
| [`xrServer_secure_messaging.cpp`](xrServer_secure_messaging.cpp.md) | Establishes a per-client shared secret and wraps chosen messages in it, so that a message a client must not forge cannot be forged. |
| [`xrServer_sls_clear.cpp`](xrServer_sls_clear.cpp.md) | Empties the world: destroys every entity, children before parents, announcing each destruction as a backdated event. |
| [`xrServer_svclient_validation.cpp`](xrServer_svclient_validation.cpp.md) | Answers whether an entity the server knows about is actually a live, usable object on the client that simulates it. |
| [`xrServer_svclient_validation.h`](xrServer_svclient_validation.h.md) | Declares the one question the ownership path asks before touching an entity: is it actually alive on the client that simulates it. |
| [`xrServer_updates_compressor.cpp`](xrServer_updates_compressor.cpp.md) | Packs a frame's worth of per-entity updates into as few packets as possible: drop what has not changed, pile the rest into a buffer, compress the buffer, and split at the packet boundary. |
| [`xrServer_updates_compressor.h`](xrServer_updates_compressor.h.md) | Declares the three independent tricks the server uses to shrink its per-frame world broadcast, and the buffers each needs. |
| [`xrServerMapSync.cpp`](xrServerMapSync.cpp.md) | Answers a joining client's claim about which level it has, and whether its copy is the same one. |
| [`xrServerMapSync.h`](xrServerMapSync.h.md) | The three answers a server can give when a client says which level it has. |

### The level, the session and game modes

| File | Role |
|---|---|
| [`autosave_manager.cpp`](autosave_manager.cpp.md) | Periodically saves the game by itself, but only at a moment when saving is safe and the result is worth loading. |
| [`autosave_manager.h`](autosave_manager.h.md) | Declares the autosave timer and the readiness counter implemented in `autosave_manager.cpp`. |
| [`autosave_manager_inline.h`](autosave_manager_inline.h.md) | The counter and timestamp operations of the autosave manager. |
| [`client_spawn_manager.cpp`](client_spawn_manager.cpp.md) | Lets one object say "tell me when the object with this identifier comes into existence", and delivers the notification exactly once. |
| [`client_spawn_manager.h`](client_spawn_manager.h.md) | Declares the spawn-notification registry implemented in `client_spawn_manager.cpp`. |
| [`client_spawn_manager_inline.h`](client_spawn_manager_inline.h.md) | The spawn-notification registry's trivial constructor and debug accessor. |
| [`configs_common.cpp`](configs_common.cpp.md) | The signature parameters shared by the client that signs a configuration dump and the server that checks it. |
| [`configs_common.h`](configs_common.h.md) | Declares the shared signature parameters defined in `configs_common.cpp`. |
| [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) | The server's check: rebuild what a client's configuration dump *should* have been, compare, and name the first thing that differs. |
| [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) | Declares the server-side dump verifier implemented in `configs_dump_verifyer.cpp`. |
| [`configs_dumper.cpp`](configs_dumper.cpp.md) | Builds, signs and compresses an image of the configuration a multiplayer client is actually running with, on a background thread. |
| [`configs_dumper.h`](configs_dumper.h.md) | Declares the background configuration dumper implemented in `configs_dumper.cpp`. |
| [`console_commands.cpp`](console_commands.cpp.md) | Declares the game layer's entire console surface: every named runtime setting and every command a player, modder or tester can type. |
| [`console_commands_mp.cpp`](console_commands_mp.cpp.md) | The multiplayer console surface: server rules, administration, voting, demo playback and the account commands. |
| [`game_base.cpp`](game_base.cpp.md) | The per-player scoreboard record and its wire format, and the game clock: two independently rescalable in-world times derived from the server's real time. |
| [`game_base.h`](game_base.h.md) | Declares the game-rules base — the per-player scoreboard record, the team record, the game-state interface every mode implements, and the game clock — implemented in `game_base.cpp`. |
| [`game_base_kill_type.h`](game_base_kill_type.h.md) | The three vocabularies a multiplayer kill is classified by: what killed you, what was special about it, and what it was worth. |
| [`game_base_menu_events.h`](game_base_menu_events.h.md) | The three requests a player can make of the game rules from the in-match menu. |
| [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) | Artefact hunt on the client: one artefact on the map, who is carrying it, where it shows up, and the paid-respawn reinforcement window between waves. |
| [`game_cl_artefacthunt.h`](game_cl_artefacthunt.h.md) | Declares the artefact hunt client rules, implemented in `game_cl_artefacthunt.cpp`. |
| [`game_cl_artefacthunt_snd_msg.h`](game_cl_artefacthunt_snd_msg.h.md) | Artefact hunt's twenty announcement identifiers: every combination of which team, which event, and whose perspective. |
| [`game_cl_base.cpp`](game_cl_base.cpp.md) | The client's mirror of the match: it applies the server's player table and clock, reconciles the two into the local world, and announces who joined and left. |
| [`game_cl_base.h`](game_cl_base.h.md) | Declares the client half of the game rules — the player table every client mirrors, and the wide set of hooks a mode overrides — implemented in `game_cl_base.cpp`. |
| [`game_cl_base_weapon_usage_statistic.cpp`](game_cl_base_weapon_usage_statistic.cpp.md) | Match telemetry: each client tracks its own shots and provisional hits, the server confirms or denies each one, and the confirmed record is periodically pulled back to the server to be written out. |
| [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) | Declares the match telemetry: every shot fired, every hit landed, where on the body it landed, what it was worth — implemented in `game_cl_base_weapon_usage_statistic.cpp` and persisted in `..._save.cpp`. |
| [`game_cl_base_weapon_usage_statistic_save.cpp`](game_cl_base_weapon_usage_statistic_save.cpp.md) | Writes the match telemetry out: a binary report file named for the level, the mode and the clock, and a parallel text report in the configuration format. |
| [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) | Capture the artefact on the client: two artefacts, two bases, who holds which, the money and buy cycle, warm-up, voting and the time limit. |
| [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) | Declares the capture-the-artefact client mode. |
| [`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md) | The on-screen captions of a capture-the-artefact match: who has the artefact, who scored, how long is left. |
| [`game_cl_capture_the_artefact_captions_manager.h`](game_cl_capture_the_artefact_captions_manager.h.md) | Declares that captions manager. |
| [`game_cl_capture_the_artefact_messages_menu.cpp`](game_cl_capture_the_artefact_messages_menu.cpp.md) | The team-message menu of a capture-the-artefact match — the quick phrases a player can broadcast. |
| [`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md) | The buy screen a capture-the-artefact player sees between rounds. |
| [`game_cl_capturetheartefact_snd_msg.h`](game_cl_capturetheartefact_snd_msg.h.md) | Capture the artefact's two announcement identifiers, occupying the four-hundred block. |
| [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) | Deathmatch on the client: the warm-up countdown, the frag and time limits, the sequence a player walks through before he can spawn, and the vote display. |
| [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md) | Declares deathmatch on the client, and with it the buy screen, the skin screen and the preset system every mode inherits. Implemented in `game_cl_deathmatch.cpp` and `game_cl_deathmatch_buywnd.cpp`. |
| [`game_cl_deathmatch_buywnd.cpp`](game_cl_deathmatch_buywnd.cpp.md) | The buy screen's half of deathmatch: seed it from what the player is actually carrying, apply the rank's weapon upgrades, and send the purchase as a slot-and-item list with a money delta. |
| [`game_cl_deathmatch_snd_messages.h`](game_cl_deathmatch_snd_messages.h.md) | Deathmatch's eleven announcement identifiers, occupying the hundred-block reserved for it. |
| [`game_cl_mp.cpp`](game_cl_mp.cpp.md) | The multiplayer client: input routing by match phase, the kill feed, the money-bonus display, voting, the spectator policy … |
| [`game_cl_mp.h`](game_cl_mp.h.md) | Declares the client-side multiplayer rules shared by all four modes — teams, announcements, voting, bonuses, the quick-speech radio and the anti-cheat file transfer — implemented in `game_cl_mp.cpp`. |
| [`game_cl_mp_messages_menu.cpp`](game_cl_mp_messages_menu.cpp.md) | The quick-speech radio: a small menu of authored phrases, each with several recorded variants per team, heard as radio by your own side and as a voice in the world by the other. |
| [`game_cl_mp_messages_menu.h`](game_cl_mp_messages_menu.h.md) | A fragment of a class body, textually pasted into the multiplayer client's declaration: the quick-speech menu's members and methods. |
| [`game_cl_mp_snd_messages.cpp`](game_cl_mp_snd_messages.cpp.md) | The announcement mixer: one announcement plays at a time, a more important one cuts off a less important one, and equally important ones queue behind each other. |
| [`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md) | The five announcement identifiers common to every multiplayer mode, and the base of a numbering scheme the mode-specific headers extend. |
| [`game_cl_single.cpp`](game_cl_single.cpp.md) | Single-player rules: build the single-player screen, and take the clock from the alife simulation instead of from the match. |
| [`game_cl_single.h`](game_cl_single.h.md) | Declares the single-player game rules, and the four difficulty levels, implemented in `game_cl_single.cpp`. |
| [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md) | Team deathmatch on the client: the team-selection gate in front of everything else, friendly identification, the two-team scoreboard, and the lead-change announcements. |
| [`game_cl_teamdeathmatch.h`](game_cl_teamdeathmatch.h.md) | Declares the team deathmatch client rules, implemented in `game_cl_teamdeathmatch.cpp`. |
| [`game_cl_teamdeathmatch_snd_messages.h`](game_cl_teamdeathmatch_snd_messages.h.md) | Team deathmatch's fifteen announcement identifiers, occupying the two-hundred block. |
| [`game_news.cpp`](game_news.cpp.md) | Persists one news item: five fields in a fixed order, and nothing else. |
| [`game_news.h`](game_news.h.md) | Declares one news item — the record behind the message ticker that reports what the world and the plot are doing — implemented in `game_news.cpp`. |
| `game_sv_artefacthunt.cpp` | The artefact-hunt mode's server rules: one artefact in the world, carry it to your base to score. |
| `game_sv_artefacthunt.h` | Declares the artefact-hunt server mode. |
| [`game_sv_artefacthunt_process_event.cpp`](game_sv_artefacthunt_process_event.cpp.md) | Artefact hunt's two extra server events: a player entered or left a team's base. |
| [`game_sv_base.cpp`](game_sv_base.cpp.md) | The server's shared rules: load the level's respawn points, hand them out without stacking players, queue every client event for the simulation thread, and run the map rotation. |
| [`game_sv_base.h`](game_sv_base.h.md) | Declares the server half of the game rules: the interface every mode must satisfy, and the shared implementation of respawn points, map rotation and the delayed-event queue. Implemented in `game_sv_base.cpp`. |
| [`game_sv_base_console_vars.cpp`](game_sv_base_console_vars.cpp.md) | Empty. A translation unit that contains only its precompiled-header include. |
| [`game_sv_base_console_vars.h`](game_sv_base_console_vars.h.md) | Empty. A header that once declared the server's console variables and now declares nothing. |
| `game_sv_capture_the_artefact.cpp` | The capture-the-artefact mode's server rules: two artefacts, two bases, and the scoring and reset when one is taken. |
| `game_sv_capture_the_artefact.h` | Declares the capture-the-artefact server mode. |
| `game_sv_capture_the_artefact_buy_event.cpp` | The purchase event of a capture-the-artefact match, validated server-side against the player's money. |
| `game_sv_capture_the_artefact_myteam_impl.cpp` | The per-team bookkeeping of a capture-the-artefact match. |
| [`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md) | Capture the artefact's four extra server events: suicide, purchase finished, and entering or leaving a team base — with the base identifier shifted to zero-based. |
| `game_sv_deathmatch.cpp` | The deathmatch mode's server rules: respawn, score by kill, end on the frag or time limit. |
| `game_sv_deathmatch.h` | Declares the deathmatch server mode. |
| [`game_sv_deathmatch_process_event.cpp`](game_sv_deathmatch_process_event.cpp.md) | Deathmatch's two extra server events: a player asked to be killed, and a player finished shopping. |
| [`game_sv_event_queue.cpp`](game_sv_event_queue.cpp.md) | The queue game events cross threads on: the network thread fills it, the simulation thread drains it, and the records are pooled so a busy match does not allocate. |
| [`game_sv_event_queue.h`](game_sv_event_queue.h.md) | Declares the server's inbound game-event queue and the event record, implemented in `game_sv_event_queue.cpp`. |
| [`game_sv_item_respawner.cpp`](game_sv_item_respawner.cpp.md) | Keeps the pickup points stocked: every point holds a prototype entity, and when the item on it is taken a timer starts that clones the prototype back into the world. |
| [`game_sv_item_respawner.h`](game_sv_item_respawner.h.md) | Declares the multiplayer item respawner — the thing that keeps weapons and ammunition appearing at their pickup points — implemented in `game_sv_item_respawner.cpp`. |
| `game_sv_mp.cpp` | What every multiplayer server mode shares: the round clock, the buy phase, the ready check and the vote system. |
| `game_sv_mp.h` | Declares the shared multiplayer server mode. |
| [`game_sv_mp_team.h`](game_sv_mp_team.h.md) | The per-team tuning record: which skins a team may wear, what it spawns holding, and the full money-reward table that drives multiplayer economy. |
| [`game_sv_mp_vote_flags.h`](game_sv_mp_vote_flags.h.md) | The bit set naming which kinds of player vote a server permits. |
| [`game_sv_single.cpp`](game_sv_single.cpp.md) | The single-player session: a game mode whose entire rule set is "there is an alife simulation, and it decides". |
| [`game_sv_single.h`](game_sv_single.h.md) | Declares the single-player session: the game mode that owns an alife simulator. |
| [`game_sv_teamdeathmatch.cpp`](game_sv_teamdeathmatch.cpp.md) | Team deathmatch: two teams whose scores are the sum of their members' frags, with friendly fire, team-kill punishment, automatic balancing and swapping, and a drop-bag that replaces ordinary item pickup. |
| [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md) | Declares team deathmatch: deathmatch plus two teams, a shared score, friendly fire, team balancing and a team base. |
| [`game_sv_teamdeathmatch_process_event.cpp`](game_sv_teamdeathmatch_process_event.cpp.md) | Routes the two team-base events out of the session's event stream; everything else falls through to deathmatch. |
| [`game_type.cpp`](game_type.cpp.md) | Answers, for any code anywhere in the game layer, whether it is running the authoritative side, the local side, or a single-player session. |
| [`game_type.h`](game_type.h.md) | Declares the three free predicates that answer "which side of the client/server split am I on, and is this a single-player session". |
| [`GamePersistent.cpp`](GamePersistent.cpp.md) | The game module's process-lifetime object: it brings the game layer up and down around the engine's own startup, owns the main menu and loading screen, drives the intro chain, plays the weather system's ambient … |
| [`GamePersistent.h`](GamePersistent.h.md) | Declares the game module's process-lifetime object, implemented in `GamePersistent.cpp`. |
| [`Level.cpp`](Level.cpp.md) | The game layer's per-frame heartbeat: it owns the loaded level's managers, drains the network event queue, runs correction prediction, and drives every subsystem once per frame in a fixed order. |
| [`Level.h`](Level.h.md) | Declares the game layer's level — the object that exists for as long as a match does — implemented across `Level.cpp` and a dozen siblings. |
| [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) | Every bullet and fragment in flight, simulated centrally as a ballistic trajectory swept against the world, with hits deferred to a frame boundary. |
| [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) | Declares the bullet record and the central projectile simulation, implemented in `Level_Bullet_Manager.cpp` and `Level_bullet_manager_firetrace.cpp`. |
| [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) | The half of the bullet manager that decides what a bullet does when its ray meets something: whether the target is worth hitting at all, whether the bullet ricochets, sticks or punches through, how much energy … |
| [`level_changer.cpp`](level_changer.cpp.md) | The volume at the edge of a level that asks the player whether to travel, and carries the destination he arrives at. |
| [`level_changer.h`](level_changer.h.md) | Declares the trigger volume that moves the player from one level to another. |
| [`level_debug.cpp`](level_debug.cpp.md) | The development build's diagnostic overlay: per-object floating labels, fixed screen text, and world-space shapes, each owned by whichever system wrote it. |
| [`level_debug.h`](level_debug.h.md) | Declares the development build's scratchpad for on-screen diagnostics: labels floating over objects, fixed-position text, and shapes drawn in the world. |
| [`Level_GameSpy_Funcs.cpp`](Level_GameSpy_Funcs.cpp.md) | Answers the matchmaking service's key-validation challenge on behalf of the connecting client. |
| [`Level_input.cpp`](Level_input.cpp.md) | The level's input dispatch: the fixed chain every key, mouse movement, gamepad event and text character walks through on its way from the window to the controlled entity … |
| [`Level_load.cpp`](Level_load.cpp.md) | the game layer's two hooks into level loading — what must exist before the geometry loads, what must be built after it … |
| [`level_map_locations.cpp`](level_map_locations.cpp.md) | Empty. |
| [`Level_network.cpp`](Level_network.cpp.md) | the level's client side of the network — connecting, sending per-frame state, tearing the world down, and turning a refusal into something the player can read. |
| [`Level_network_compressed_updates.cpp`](Level_network_compressed_updates.cpp.md) | Unpacks a server update packet that carries several entity updates squeezed together under a shared dictionary, and uses its arrival time to decide how many physics steps the client owes. |
| [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) | Records a multiplayer session by logging every server message to a file with its arrival time, and replays it by feeding those messages back into the client at the same relative times, so that a recording is in … |
| [`Level_network_Demo.h`](Level_network_Demo.h.md) | The demo recorder's slice of the level's own declaration: fields and methods spliced directly into the level class, implemented in `Level_network_Demo.cpp`. |
| [`Level_network_digest_computer.cpp`](Level_network_digest_computer.cpp.md) | Hashes the machine's retail product key and sends the hash to the server, so the server can tell two clients apart without ever seeing the key. |
| [`Level_network_map_sync.cpp`](Level_network_map_sync.cpp.md) | The handshake that proves a joining client has the same level data as the server, and the loop that blocks level bring-up until the server has sent its game configuration. |
| [`Level_network_map_sync.h`](Level_network_map_sync.h.md) | Declares the record holding one connection attempt's map-verification state, implemented in `Level_network_map_sync.cpp`. |
| [`Level_network_messages.cpp`](Level_network_messages.cpp.md) | The client's message dispatch: one pass over everything the transport delivered this frame, routing each message either into the deferred game-event queue or straight into the object it names. |
| [`Level_network_spawn.cpp`](Level_network_spawn.cpp.md) | Turns a spawn record into a live object: decodes the record off the wire, asks the factory for a client object, runs the spawn lifecycle, and wires up ownership and player control. |
| [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) | The client's connection sequence, cut into six resumable steps so that the loading screen keeps painting while the connection, the level load and the server handshake proceed. |
| [`Level_secure_messaging.cpp`](Level_secure_messaging.cpp.md) | Wraps a message in an obfuscating cipher with a checksum, unwraps received ones, and re-derives the shared key whenever the server sends a new seed. |
| [`Level_SLS_Default.cpp`](Level_SLS_Default.cpp.md) | asks the server side to build its default world state, the path taken when a level is started without a save to restore from. |
| [`Level_SLS_Load.cpp`](Level_SLS_Load.cpp.md) | the level-side hook for restoring a saved game, which is empty because the restore happens entirely on the server side. |
| [`Level_SLS_Save.cpp`](Level_SLS_Save.cpp.md) | writes the level's saved-game snapshot — a session name and the whole server-side world — as a chunked image. |
| [`level_sounds.cpp`](level_sounds.cpp.md) | The level's ambience: authored point sources that loop or chirp on a schedule, and a music playlist that picks a track appropriate to the hour. |
| [`level_sounds.h`](level_sounds.h.md) | Declares the level's ambient soundscape: authored looping and intermittent point sources, and the time-of-day music playlist. |
| [`Level_start.cpp`](Level_start.cpp.md) | the level's bring-up sequence — a chain of resumable steps that stands up a server, connects a client to it, and reports the game as ready or diagnoses why it is not. |
| [`LevelFogOfWar.cpp`](LevelFogOfWar.cpp.md) | Per-level record of which parts of the map the player has walked near, and the pass that draws the unexplored parts over the map screen. |
| [`LevelFogOfWar.h`](LevelFogOfWar.h.md) | Declares the per-level exploration grid and the manager that keeps one per visited level, implemented in `LevelFogOfWar.cpp`. |
| [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) | The navigation overlays: the level's walkable mesh, the cross-level graph miniature, restrictor borders, per-vertex cover values, and where offline creatures currently are. |
| [`LevelGraphDebugRender.hpp`](LevelGraphDebugRender.hpp.md) | Declares the navigation-data overlay renderer, implemented in `LevelGraphDebugRender.cpp`. |
| [`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md) | Peeks into a save file for four facts — clock, level, level name, actor health — by decompressing it and reading exactly the actor's record and the game graph, then throwing the rest away. |
| [`saved_game_wrapper.h`](saved_game_wrapper.h.md) | Declares the peek-at-a-save-file type: the four facts the load menu needs, without loading the game. |
| [`saved_game_wrapper_inline.h`](saved_game_wrapper_inline.h.md) | The four accessors. |
| [`Spectator.cpp`](Spectator.cpp.md) | The entity a player becomes when he is dead, has not joined yet, or is watching a recorded match: a bodiless camera with five viewing modes and a rule set saying which of them this game mode will allow. |
| [`Spectator.h`](Spectator.h.md) | Declares the bodiless viewer entity implemented in `Spectator.cpp`, and names its five camera modes. |
| [`spectator_camera_first_eye.cpp`](spectator_camera_first_eye.cpp.md) | A first-person camera whose look speed is scaled by a frame time supplied from outside, so that a spectator's free look advances at the same rate whether or not the simulation it is watching is running. |
| [`spectator_camera_first_eye.h`](spectator_camera_first_eye.h.md) | Declares the spectator's first-person camera, which takes its frame time from its owner rather than from the global clock. |

### Tasks, the map and the guide

| File | Role |
|---|---|
| [`encyclopedia_article.cpp`](encyclopedia_article.cpp.md) | Parses one authored article out of XML — title, group, body, icon and type — and defines how the player's copy of an article is persisted. |
| [`encyclopedia_article.h`](encyclopedia_article.h.md) | Declares the authored, shared side of an encyclopedia article, implemented in `encyclopedia_article.cpp`. |
| [`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md) | What the player's copy of an article is — the identifier, when they received it, whether they have read it, and which of the four in-game document types it is. |
| [`GameTask.cpp`](GameTask.cpp.md) | One quest: an authored tree of objectives, each with completion and failure conditions expressed as information portions and script predicates, a map location, an encyclopedia article and a deadline. |
| [`GameTask.h`](GameTask.h.md) | Declares a quest and its objectives, implemented in `GameTask.cpp`. |
| [`GameTaskDefs.h`](GameTaskDefs.h.md) | The vocabulary of the quest system: task state, task type, the identifiers a task and its objectives are addressed by, and the saved registry that carries the player's task list. |
| [`GametaskManager.cpp`](GametaskManager.cpp.md) | The player's quest journal: hands tasks out, re-evaluates every in-progress objective once a frame, decides which task is the *active* one per category, and keeps the map pointer on it. |
| [`GametaskManager.h`](GametaskManager.h.md) | Declares the player's quest journal, implemented in `GametaskManager.cpp`. |
| [`InfoDocument.cpp`](InfoDocument.cpp.md) | A document you can pick up: an inventory item whose only behaviour is to hand one information portion to whoever picks it up. |
| [`InfoDocument.h`](InfoDocument.h.md) | Declares the pick-up document, implemented in `InfoDocument.cpp`. |
| [`map_location.cpp`](map_location.cpp.md) | One marker on the map: where its subject is, whether that is still worth showing, and how it is drawn on the level map, the minimap and the edge-of-screen pointer. |
| [`map_location.h`](map_location.h.md) | Declares a map marker: one entity's presence on the level map, the minimap and the pointer ring, in whatever visual form its spot type describes. |
| [`map_location_defs.h`](map_location_defs.h.md) | The saved form of a map marker: the (spot type, object) key that identifies it, and the per-level registry that persists them. |
| [`map_manager.cpp`](map_manager.cpp.md) | Owns every map marker: creates them, finds them, advances them on a staggered schedule, and reaps the ones whose subject is gone. |
| [`map_manager.h`](map_manager.h.md) | Declares the owner of every map marker: the registry, the lookups, and the per-frame sweep. |
| [`map_spot.cpp`](map_spot.cpp.md) | The widgets a map marker is drawn as: the clickable icon, the edge pointer, the minimap dot that changes with height, and the rich spot with satellite icons and a countdown. |
| [`map_spot.h`](map_spot.h.md) | Declares the four widget kinds a map marker draws itself with. |

### The actor

| File | Role |
|---|---|
| [`Actor.cpp`](Actor.cpp.md) | The player's entity: the one object that is simultaneously a living creature, an inventory owner, an input receiver, a camera rig and a conversation partner … |
| [`Actor.h`](Actor.h.md) | Declares the player's entity and the full set of behaviours mixed into it. |
| [`actor_anim_defs.h`](actor_anim_defs.h.md) | The player character's animation vocabulary: which named motions exist, how they are grouped by posture and by weapon class, and where they come from in the model's animation bank. |
| [`actor_communication.cpp`](actor_communication.cpp.md) | Everything the player learns or is told: information portions and what they unlock, the news feed, the encyclopedia, conversations, and the personal-data-assistant contact list. |
| [`actor_defs.h`](actor_defs.h.md) | The player character's shared vocabulary: the movement command bit set, the camera modes, the context-action kinds, and the three record shapes the network and prediction paths pass around. |
| [`Actor_Events.cpp`](Actor_Events.cpp.md) | Every change to the actor that must be authoritative arrives here as a message: taking and dropping items, eating, equipping, entering a holder, being teleported, being healed. |
| [`Actor_Feel.cpp`](Actor_Feel.cpp.md) | What the player notices: which loose items are near and in view, which of them can be picked up, which primed grenades are worth a warning, and how much noise the player is making. |
| [`Actor_Flags.h`](Actor_Flags.h.md) | The player-behaviour switch set, plus the mouse-sensitivity ranges and the sleep duration — every one of them a console-settable global. |
| [`actor_input_handler.cpp`](actor_input_handler.cpp.md) | Installs and removes an external claim on the player's input, so that something other than the player's own control loop can drive or veto their commands. |
| [`actor_input_handler.h`](actor_input_handler.h.md) | The interface something must satisfy to take over, filter or scale the player's input. |
| [`actor_memory.cpp`](actor_memory.cpp.md) | The player's vision: what the player has actually seen, computed with the player's own camera, so that scripts and creatures can ask what the human is looking at. |
| [`actor_memory.h`](actor_memory.h.md) | Declares the player's vision client, implemented in `actor_memory.cpp`. |
| [`Actor_Movement.cpp`](Actor_Movement.cpp.md) | Reconciles what the player asked for with what the body can do: the real movement state, the acceleration the physics receives, the body's facing … |
| [`actor_mp_client.cpp`](actor_mp_client.cpp.md) | The multiplayer player object: the player character with camera freedom removed, death forced, remote-view smoothing applied, and one extra server-driven event. |
| [`actor_mp_client.h`](actor_mp_client.h.md) | Declares the multiplayer player object, implemented across `actor_mp_client.cpp`, `actor_mp_client_export.cpp` and `actor_mp_client_import.cpp`. |
| [`actor_mp_client_export.cpp`](actor_mp_client_export.cpp.md) | Gathers a networked player's state from the live simulation into the wire record, and decides whether it is worth sending at all. |
| [`actor_mp_client_import.cpp`](actor_mp_client_import.cpp.md) | Applies a received player update: authoritative values are set, the view is snapped to the sender's aim, and the rest is queued for interpolation rather than applied at once. |
| [`actor_mp_server.cpp`](actor_mp_server.cpp.md) | The server-side record of a networked player: the authoritative state the server relays, and the rule that a dead player stops being relayed. |
| [`actor_mp_server.h`](actor_mp_server.h.md) | Declares the server-side networked player record, implemented across `actor_mp_server.cpp`, `actor_mp_server_export.cpp` and `actor_mp_server_import.cpp`. |
| [`actor_mp_server_export.cpp`](actor_mp_server_export.cpp.md) | Fills the server's wire record from the authoritative player record, and relays it. |
| [`actor_mp_server_import.cpp`](actor_mp_server_import.cpp.md) | Accepts a player's own report of their state as authoritative, unless they are dead. |
| [`actor_mp_state.cpp`](actor_mp_state.cpp.md) | The bit-packed wire form of a networked player's state: what is sent, at what precision, and in what order. |
| [`actor_mp_state.h`](actor_mp_state.h.md) | Declares the networked player state record and the holder that writes and reads it; the wire format is in `actor_mp_state.cpp`. |
| [`actor_mp_state_inline.h`](actor_mp_state_inline.h.md) | Initializes a networked player state holder to a state that is zero everywhere a zero is meaningful and to an identity rotation where it is not. |
| [`Actor_Network.cpp`](Actor_Network.cpp.md) | The actor's whole lifecycle plus its replication: what a spawn record becomes, what the wire carries each tick, how a remote player's motion is smoothed into a plausible path, how a save is written … |
| [`Actor_Sleep.cpp`](Actor_Sleep.cpp.md) | Empty. |
| [`actor_statistic_defs.h`](actor_statistic_defs.h.md) | The shape of the player's end-of-game scorecard: named sections, each a list of named tallies, persisted with the save. |
| [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md) | Keeps the player's scorecard: find-or-create a section, find-or-create a tally within it, add to it, and total it — with "not scorable" as a first-class answer. |
| [`actor_statistic_mgr.h`](actor_statistic_mgr.h.md) | Declares the player's scorecard manager, implemented in `actor_statistic_mgr.cpp`. |
| [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) | Where the actor meets a weapon: how far a shot may stray given how the player is moving, where the shot originates, and what recoil does to the view. |
| [`ActorAnimation.cpp`](ActorAnimation.cpp.md) | Chooses, every frame, which three motions the actor's body plays — legs, torso and head — and bends the spine so the body follows where the camera is looking. |
| [`ActorAnimation.h`](ActorAnimation.h.md) | Names for the movement-state bit combinations the animation selector switches on. |
| [`ActorBackpack.cpp`](ActorBackpack.cpp.md) | The backpack: a wearable inventory item whose only job is to alter the actor's carry limit and movement. |
| [`ActorBackpack.h`](ActorBackpack.h.md) | Declares the backpack item implemented in `ActorBackpack.cpp`. |
| [`ActorCameras.cpp`](ActorCameras.cpp.md) | Places the player's eye each frame: on the body at the right height, leaning around corners only as far as the wall allows, smoothed over stairs, and pushed out of geometry it would otherwise see through. |
| [`ActorCondition.cpp`](ActorCondition.cpp.md) | The player's body as a set of slowly-moving numbers: stamina spent by moving, hunger, alcohol, radiation, psychic health, temporary boosts … |
| [`ActorCondition.h`](ActorCondition.h.md) | Declares the actor's condition model and the death sequence, both implemented in `ActorCondition.cpp`. |
| [`ActorEffector.cpp`](ActorEffector.cpp.md) | The actor's camera- and screen-effect stack: how a data-driven shake or a full-screen filter is attached to the player, how strongly it applies … |
| [`ActorEffector.h`](ActorEffector.h.md) | Declares the actor's effector vocabulary: the strength-source interface, the animation-driven camera effectors, and the actor's two-camera effector manager. |
| [`ActorFollowers.cpp`](ActorFollowers.cpp.md) | Dead code: an abandoned squad-of-followers feature, commented out in its entirety. |
| [`ActorFollowers.h`](ActorFollowers.h.md) | Dead code: the declaration half of the withdrawn follower feature, commented out in its entirety. |
| [`ActorHelmet.cpp`](ActorHelmet.cpp.md) | The helmet: a worn item that reduces incoming damage per bone and per damage type, and whose damage formula differs depending on which of the three games' data it came from. |
| [`ActorHelmet.h`](ActorHelmet.h.md) | Declares the helmet implemented in `ActorHelmet.cpp`. |
| [`ActorInput.cpp`](ActorInput.cpp.md) | Turns the player's actions into a wishful movement state, item commands, and the one gesture that does everything — *use*. |
| [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md) | The one transition by which the actor enters and leaves anything it can sit in or operate — a vehicle, a mounted gun, a turret. |
| [`ActorVehicle.cpp`](ActorVehicle.cpp.md) | Seats the actor in a car and gets it out again: swap the walking capsule for the car's physics, rebind the skeleton's aim callbacks, and hide the weapon. |
| [`MPPlayersBag.cpp`](MPPlayersBag.cpp.md) | The bag a killed multiplayer player's belongings drop into: a container that is itself an inventory item, and that removes itself once the match's item-lifetime rule says so. |
| [`MPPlayersBag.h`](MPPlayersBag.h.md) | Declares the multiplayer drop bag implemented in `MPPlayersBag.cpp`. |
| [`player_account.cpp`](player_account.cpp.md) | Who a multiplayer player is according to the account service: their nickname, clan and profile number, and the frozen shape those take on the wire. |
| [`player_account.h`](player_account.h.md) | Declares the online identity a multiplayer player carries into a match — nickname, clan, profile number — implemented in `player_account.cpp`. |
| [`player_hud.cpp`](player_hud.cpp.md) | The first-person view: one pair of arms, up to two items in them, the authored measurements that place each item, the motion aliases that animate it, and the inertia that makes the weapon lag the camera. |
| [`player_hud.h`](player_hud.h.md) | Declares the first-person hands rig and everything attached to it — implemented in `player_hud.cpp`. |
| [`player_hud_tune.cpp`](player_hud_tune.cpp.md) | Positioning a weapon in the player's hands by eye: live sliders over the eleven authored measurements, the debug markers that show where the muzzle actually is, and the configuration text to paste back. |
| [`player_hud_tune.h`](player_hud_tune.h.md) | Declares the in-game tool that lets an artist position a weapon in the player's hands live and paste the result back into configuration. Implemented in `player_hud_tune.cpp`. |
| [`player_name_modifyer.cpp`](player_name_modifyer.cpp.md) | Replaces every character in a player-chosen nickname that would break a file path or a console line. |
| [`player_name_modifyer.h`](player_name_modifyer.h.md) | Declares the one function that makes a player-chosen nickname safe to put in a file path and a console line. |

### Cameras, effectors and post-processing

| File | Role |
|---|---|
| [`CameraEffector.cpp`](CameraEffector.cpp.md) | Empty: the camera-effector types are entirely declared in `CameraEffector.h`. |
| [`CameraEffector.h`](CameraEffector.h.md) | The two frozen identifier spaces the game's camera and screen effects are addressed by. |
| [`CameraFirstEye.cpp`](CameraFirstEye.cpp.md) | The first-person camera: the eye sits exactly where it is put, looks where the player aims, and can be eased onto a point when something else wants to direct the view. |
| [`CameraFirstEye.h`](CameraFirstEye.h.md) | Declares the first-person camera implemented in `CameraFirstEye.cpp`. |
| [`CameraLook.cpp`](CameraLook.cpp.md) | The three third-person cameras: one that orbits the player at a distance the world can push in, one that adds a shoulder offset and an auto-aim lock, and one that holds a fixed framing. |
| [`CameraLook.h`](CameraLook.h.md) | Declares the three third-person cameras implemented in `CameraLook.cpp`. |
| [`CameraRecoil.h`](CameraRecoil.h.md) | The tuning record describing how firing a weapon kicks the view, and how the view recovers. |
| [`EffectorBobbing.cpp`](EffectorBobbing.cpp.md) | The walk cycle's effect on the first-person camera: a figure-of-eight bob whose amplitude and rate follow the gait, faded in and out so that starting and stopping do not snap. |
| [`EffectorBobbing.h`](EffectorBobbing.h.md) | Declares the walk-cycle camera effector implemented in `EffectorBobbing.cpp`. |
| [`EffectorFall.cpp`](EffectorFall.cpp.md) | Two one-shot camera effectors: the knee-bend dip after a landing, and a timed override of the depth-of-field parameters. |
| [`EffectorFall.h`](EffectorFall.h.md) | Declares the landing dip and the timed depth-of-field override implemented in `EffectorFall.cpp`. |
| [`EffectorShot.cpp`](EffectorShot.cpp.md) | Weapon recoil as a camera offset: each shot kicks the aim up and sideways by a randomized amount, the kick accumulates over a burst, and it relaxes back at a configured rate. |
| [`EffectorShot.h`](EffectorShot.h.md) | Declares the recoil model and its camera-chain wrapper, implemented in `EffectorShot.cpp`. |
| [`EffectorShotX.cpp`](EffectorShotX.cpp.md) | Dead file: an abandoned recoil variant that drove the character's camera angles directly instead of publishing an offset. Nothing is compiled. |
| [`EffectorShotX.h`](EffectorShotX.h.md) | Dead file: the declaration matching the abandoned recoil variant in `EffectorShotX.cpp`. Entirely commented out. |
| [`EffectorZoomInertion.cpp`](EffectorZoomInertion.cpp.md) | Aim sway: while a scoped weapon is held steady, the aim point wanders along a random walk whose radius and speed scale with the weapon's current dispersion … |
| [`EffectorZoomInertion.h`](EffectorZoomInertion.h.md) | Declares the aim-sway effector implemented in `EffectorZoomInertion.cpp`. |
| [`PostprocessAnimator.cpp`](PostprocessAnimator.cpp.md) | Drives a keyframed screen-effect curve — colour grading, noise, blur, duality — as a camera effector, in four flavours that differ only in where the blend weight comes from. |
| [`PostprocessAnimator.h`](PostprocessAnimator.h.md) | Declares the four screen-effect effector flavours implemented in `PostprocessAnimator.cpp`. |
| [`pp_effector_custom.cpp`](pp_effector_custom.cpp.md) | An indefinite screen effect that blends toward an authored look by a factor, and the controller that switches it on and off from a condition. |
| [`pp_effector_custom.h`](pp_effector_custom.h.md) | The pattern every game-driven screen effect is built on: an authored post-process state, an effector that blends toward it, and a controller that decides when it runs. |
| [`pp_effector_distance.cpp`](pp_effector_distance.cpp.md) | A screen-effect controller that ramps its post-process effect up as the viewer approaches a source and turns it off at the outer edge. |
| [`pp_effector_distance.h`](pp_effector_distance.h.md) | Declares the distance-driven post-process effector controller. |
| [`SleepEffector.cpp`](SleepEffector.cpp.md) | The screen effect that covers falling asleep and waking up: fade the picture into a sleeping state, hold it for as long as sleep lasts, then fade back. |
| [`SleepEffector.h`](SleepEffector.h.md) | Declares the sleep screen effect implemented in `SleepEffector.cpp`, and the authored record that describes one. |

### Inventory, items, artefacts and outfits

| File | Role |
|---|---|
| [`AdvancedDetector.cpp`](AdvancedDetector.cpp.md) | The directional artefact detector: a hand-held device whose needle points at the nearest hidden artefact and whose beeping speeds up as you close on it. |
| [`AdvancedDetector.h`](AdvancedDetector.h.md) | Declares the directional artefact detector implemented in `AdvancedDetector.cpp`. |
| [`antirad.cpp`](antirad.cpp.md) | An anti-radiation drug: an edible item with no behaviour beyond what its configuration section gives it. |
| [`antirad.h`](antirad.h.md) | Declares the anti-radiation drug class described in `antirad.cpp`. |
| [`Artefact.cpp`](Artefact.cpp.md) | The artefact: an object that glows and hums while it lies in the world, grants passive effects while it is carried, can be activated into an anomaly, and — for some of them … |
| [`Artefact.h`](Artefact.h.md) | Declares the artefact base class and its detector-support helper, both implemented in `Artefact.cpp`. |
| [`artefact_activation.cpp`](artefact_activation.cpp.md) | Runs the timed sequence in which a discarded artefact rises, hangs, and detonates into a new anomaly. |
| [`artefact_activation.h`](artefact_activation.h.md) | Declares the artefact-activation sequence implemented in `artefact_activation.cpp`. |
| [`BastArtifact.cpp`](BastArtifact.cpp.md) | The artefact that fights back: shoot it, and it charges up and hurls itself repeatedly at whoever is standing near. |
| [`BastArtifact.h`](BastArtifact.h.md) | Declares the self-hurling artefact implemented in `BastArtifact.cpp`. |
| [`BlackDrops.cpp`](BlackDrops.cpp.md) | An artefact with no behaviour of its own beyond what its configuration section gives it. |
| [`BlackDrops.h`](BlackDrops.h.md) | Declares an artefact class that adds nothing to the base artefact. |
| [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) | A hovering artefact that, when struck hard enough, detonates a gravitational shockwave that throws and injures everything around it. |
| [`BlackGraviArtifact.h`](BlackGraviArtifact.h.md) | Declares the shockwave artefact implemented in `BlackGraviArtifact.cpp`. |
| [`BottleItem.cpp`](BottleItem.cpp.md) | A drinkable that shatters: hit it hard enough and it breaks, with a sound and a particle burst, and ceases to exist. |
| [`BottleItem.h`](BottleItem.h.md) | Declares the breakable drinkable implemented in `BottleItem.cpp`. |
| [`CustomDetector.cpp`](CustomDetector.cpp.md) | The artefact detector: a held item that senses nearby artefacts through the touch sense, drives a small readout rendered on its own model … |
| [`CustomDetector.h`](CustomDetector.h.md) | Declares the detector item, and defines in full the generic "sense a configured set of classes within a radius" list that both the artefact and the anomaly detectors are built from. |
| [`CustomOutfit.cpp`](CustomOutfit.cpp.md) | The body armour: it wears out as it absorbs damage, reduces incoming damage by type and by which body part was struck, changes the wearer's appearance and first-person arms … |
| [`CustomOutfit.h`](CustomOutfit.h.md) | Declares the body armour implemented in `CustomOutfit.cpp`. |
| [`DummyArtifact.cpp`](DummyArtifact.cpp.md) | An artefact with no effect: the base artefact behaviour under its own class identifier. |
| [`DummyArtifact.h`](DummyArtifact.h.md) | Declares the effectless artefact implemented in `DummyArtifact.cpp`. |
| [`eatable_item.cpp`](eatable_item.cpp.md) | Consumable items: a use counter, the condition and booster effects one use applies, a weight that drops as the item is eaten, and the rule for when a spent item leaves the world. |
| [`eatable_item.h`](eatable_item.h.md) | Declares the consumable-item mixin implemented in `eatable_item.cpp`. |
| [`eatable_item_object.cpp`](eatable_item_object.cpp.md) | The concrete consumable entity: two behaviours joined into one object, with an explicit ordering at every lifecycle point. |
| [`eatable_item_object.h`](eatable_item_object.h.md) | Declares the concrete consumable world object implemented in `eatable_item_object.cpp`. |
| [`ElectricBall.cpp`](ElectricBall.cpp.md) | An artefact that, while carried, keeps its own transform pinned to its carrier's instead of following the usual attachment rules. |
| [`ElectricBall.h`](ElectricBall.h.md) | Declares the carrier-pinned artefact implemented in `ElectricBall.cpp`. |
| [`EliteDetector.cpp`](EliteDetector.cpp.md) | The two detector models with a real screen: a rotating plan view of nearby artefacts drawn onto a bone of the device's own model, and the scientific variant that also shows anomalies. |
| [`EliteDetector.h`](EliteDetector.h.md) | Declares the two screen-bearing detector models implemented in `EliteDetector.cpp`. |
| [`ExoOutfit.cpp`](ExoOutfit.cpp.md) | The powered exoskeleton suit: the base outfit under its own class identifier, with every difference expressed in configuration. |
| [`ExoOutfit.h`](ExoOutfit.h.md) | Declares the exoskeleton outfit implemented in `ExoOutfit.cpp`. |
| [`FadedBall.cpp`](FadedBall.cpp.md) | Another effectless artefact class: identity only, behaviour entirely inherited. |
| [`FadedBall.h`](FadedBall.h.md) | Declares the effectless artefact implemented in `FadedBall.cpp`. |
| [`flare.cpp`](flare.cpp.md) | A hand-held flare: a light that burns for a fixed number of seconds, dims on a fourth-power curve, throws itself away two seconds before it dies, and goes dark. |
| [`flare.h`](flare.h.md) | Declares the hand-held burning flare implemented in `flare.cpp`. |
| [`FoodItem.cpp`](FoodItem.cpp.md) | Food and drink as a class identifier: the consumable item behaviour with no additions. |
| [`FoodItem.h`](FoodItem.h.md) | Declares the food consumable implemented in `FoodItem.cpp`. |
| [`GalantineArtifact.cpp`](GalantineArtifact.cpp.md) | An artefact kind distinguished only by its class identifier and its configuration section; it adds no behaviour to the base artefact. |
| [`GalantineArtifact.h`](GalantineArtifact.h.md) | Declares the behaviourless artefact leaf implemented in `GalantineArtifact.cpp`. |
| [`GraviArtifact.cpp`](GraviArtifact.cpp.md) | The gravitational artefact: while lying loose in the world it repeatedly kicks itself upward so that it hovers unsteadily just above the ground. |
| [`GraviArtifact.h`](GraviArtifact.h.md) | Declares the hovering artefact implemented in `GraviArtifact.cpp`. |
| [`GraviZone.cpp`](GraviZone.cpp.md) | The gravitational anomaly: an inner region that pulls everything toward its centre and an outer blowout that hits whatever reaches it, plus a telekinesis cycle that lifts inert objects into the air and drops th … |
| [`GraviZone.h`](GraviZone.h.md) | Declares the gravitational anomaly implemented in `GraviZone.cpp`. |
| [`Inventory.cpp`](Inventory.cpp.md) | One entity's carried belongings: three storage areas, a slot that is currently in the hands, and the rules that move an item between them. |
| [`Inventory.h`](Inventory.h.md) | Declares the inventory, its slot record and the quick-switch priority group, implemented in `Inventory.cpp` and `inventory_quickswitch.cpp`. |
| [`inventory_item.cpp`](inventory_item.cpp.md) | What it means to be a carryable thing: a name and a weight read from configuration, a condition that wears down, a place in someone's inventory, a physical body when nobody is holding it, and … |
| [`inventory_item.h`](inventory_item.h.md) | Declares the mix-in every carryable thing in the game inherits: a name, a weight, a cost, a condition, a place in a grid, an upgrade list, and a network-synchronized physical body. |
| [`inventory_item_impl.h`](inventory_item_impl.h.md) | One accessor, separated so that an item can reach its owner without every item header depending on the inventory. |
| [`inventory_item_inline.h`](inventory_item_inline.h.md) | The two configuration-application helpers the upgrade system is built on, plus two accessors. |
| [`inventory_item_object.cpp`](inventory_item_object.cpp.md) | The join between "a thing that can be carried" and "a thing that exists physically in the world" — and the order in which the two halves are told about every event. |
| [`inventory_item_object.h`](inventory_item_object.h.md) | Declares the concrete, spawnable "an item lying in the world" class — the join of the carryable mix-in and the physically simulated object — implemented in `inventory_item_object.cpp`. |
| [`inventory_item_object_inline.h`](inventory_item_object_inline.h.md) | Empty. |
| [`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md) | The item's side of the upgrade mechanic: which upgrades it carries, how an upgrade's configuration keys are merged into its tuned numbers, and what must be stripped off the item first. |
| [`inventory_quickswitch.cpp`](inventory_quickswitch.cpp.md) | One key cycles the weapon in a slot through the backpack by authored priority, and one key cycles the grenade in hand through the types carried — the two "next thing" bindings. |
| [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md) | One purchasable upgrade: what it is made of in configuration, the three script hooks it owns, and the verdict it returns when asked whether it may be installed. |
| [`inventory_upgrade.h`](inventory_upgrade.h.md) | Declares one purchasable upgrade — the leaf of the forest — implemented in `inventory_upgrade.cpp`. |
| [`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md) | The two checks every upgrade node performs before anything item-specific is considered: has the player learned of it, and is it already installed. |
| [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) | Declares what every node of an item's upgrade forest has in common — an identifier, a known/unknown flag, and the dependent groups it unlocks — plus the frozen verdict enumeration the whole mechanic answers in. |
| [`inventory_upgrade_base_inline.h`](inventory_upgrade_base_inline.h.md) | The three field reads of an upgrade node: its identifier, that identifier as text, and whether the player has learned of it. |
| [`inventory_upgrade_group.cpp`](inventory_upgrade_group.cpp.md) | The two structural rules of the upgrade mechanic: you must have installed what unlocks this group, and within a group you may install exactly one. |
| [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) | Declares a group — a set of mutually exclusive upgrades gated behind a set of parent upgrades — implemented in `inventory_upgrade_group.cpp`. |
| [`inventory_upgrade_group_inline.h`](inventory_upgrade_group_inline.h.md) | The two field reads of an upgrade group: its identifier, and that identifier as text. |
| [`inventory_upgrade_inline.h`](inventory_upgrade_inline.h.md) | The read-only accessors of one upgrade node: its configuration section, its parent group, and its presentation fields. |
| [`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) | Builds every item's upgrade tree from configuration at startup, and is the single place an upgrade is checked, installed and recorded onto an item. |
| [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) | Declares the registry that owns every upgrade tree in the game and the operations that install upgrades onto items. |
| [`inventory_upgrade_manager_inline.h`](inventory_upgrade_manager_inline.h.md) | Empty. |
| [`inventory_upgrade_property.cpp`](inventory_upgrade_property.cpp.md) | One row of the upgrade screen's parameter display: an icon, a label, and a script function that turns a raw item parameter into the string shown beside it. |
| [`inventory_upgrade_property.h`](inventory_upgrade_property.h.md) | Declares the descriptor of one upgrade *property* — a named, icon-bearing category of item parameter whose displayed value is computed by a script. |
| [`inventory_upgrade_property_inline.h`](inventory_upgrade_property_inline.h.md) | The read-only accessors of an upgrade property descriptor. |
| [`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md) | The entry point of one item's upgrade tree: it reads the item's upgrade declaration, keeps a flat index of everything below it, and resolves the upgrade screen's grid cells. |
| [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md) | Declares the per-item root of an upgrade tree: the node that owns the item's layout scheme and the flat list of every upgrade reachable from it. |
| [`inventory_upgrade_root_inline.h`](inventory_upgrade_root_inline.h.md) | The one accessor of an upgrade root: the name of the scheme that lays its upgrades out on screen. |
| [`InventoryBox.cpp`](InventoryBox.cpp.md) | A container placed in the world: a stash, crate or locker that holds items without being able to use them. |
| [`InventoryBox.h`](InventoryBox.h.md) | Declares the world container, implemented in `InventoryBox.cpp`. |
| [`medkit.cpp`](medkit.cpp.md) | An edible item under its own class identifier, with no behaviour of its own. |
| [`medkit.h`](medkit.h.md) | Declares the medical kit, an edible item that adds nothing to the base behaviour. |
| [`MercuryBall.cpp`](MercuryBall.cpp.md) | An artefact that will not sit still: every so often it gives itself a random horizontal shove and rolls somewhere else. |
| [`MercuryBall.h`](MercuryBall.h.md) | Declares the rolling artefact implemented in `MercuryBall.cpp`. |
| [`MilitaryOutfit.cpp`](MilitaryOutfit.cpp.md) | A protective suit class that exists only so the class-identifier table has something to name: it inherits everything and overrides nothing. |
| [`MilitaryOutfit.h`](MilitaryOutfit.h.md) | Declares the military protective suit implemented — trivially — in `MilitaryOutfit.cpp`. |
| [`MosquitoBald.cpp`](MosquitoBald.cpp.md) | An anomaly that does not throw anything: it simply damages everything inside it, hard at the blowout and continuously between them. |
| [`MosquitoBald.h`](MosquitoBald.h.md) | Declares the continuously damaging anomaly implemented in `MosquitoBald.cpp`. |
| [`Needles.cpp`](Needles.cpp.md) | An artefact class that exists only as a name in the class-identifier table: it inherits everything and overrides nothing. |
| [`Needles.h`](Needles.h.md) | Declares the needles artefact implemented — trivially — in `Needles.cpp`. |
| [`PDA.cpp`](PDA.cpp.md) | The personal data assistant: an inventory item that senses which characters are nearby and tells its owner, which is what populates the contact list and the map. |
| [`PDA.h`](PDA.h.md) | Declares the personal data assistant implemented in `PDA.cpp`. |
| [`pda_space.h`](pda_space.h.md) | The two kinds of message the player's handheld device can carry. |
| [`PdaMsg.h`](PdaMsg.h.md) | Two small records: one log entry for a message exchanged through the personal data assistant, and one record of having talked to somebody. |
| [`purchase_list.cpp`](purchase_list.cpp.md) | Restocks a trader: reads an authored shopping list, rolls each line, spawns what came up, and records how short the roll fell so prices can react. |
| [`purchase_list.h`](purchase_list.h.md) | Declares the trader restocking list and its per-section deficit table. |
| [`purchase_list_inline.h`](purchase_list_inline.h.md) | Reading and writing the deficit table. |
| [`RustyHairArtifact.cpp`](RustyHairArtifact.cpp.md) | The "rusty hair" artefact: a named artefact type with no behaviour of its own. |
| [`RustyHairArtifact.h`](RustyHairArtifact.h.md) | Declares the "rusty hair" artefact implemented in `RustyHairArtifact.cpp`. |
| [`ScientificOutfit.cpp`](ScientificOutfit.cpp.md) | The scientist's protective suit: a named outfit type with no behaviour of its own. |
| [`ScientificOutfit.h`](ScientificOutfit.h.md) | Declares the scientist's suit implemented in `ScientificOutfit.cpp`. |
| [`searchlight.cpp`](searchlight.cpp.md) | A steerable spot light on a two-axis mount: aims at a script-given target, moves both axes so they arrive together, and animates its colour. |
| [`searchlight.h`](searchlight.h.md) | Declares the searchlight entity: a scripted object whose visual carries a steerable spot light and glow. |
| [`SimpleDetector.cpp`](SimpleDetector.cpp.md) | The cheapest artefact detector: it beeps and blinks faster the closer the nearest artefact is, and shows nothing else. |
| [`SimpleDetector.h`](SimpleDetector.h.md) | Declares the cheapest artefact detector implemented in `SimpleDetector.cpp`. |
| [`StalkerOutfit.cpp`](StalkerOutfit.cpp.md) | Exports the stalker's suit to the script virtual machine. |
| [`StalkerOutfit.h`](StalkerOutfit.h.md) | Declares the stalker's suit, whose only source is its script registration in `StalkerOutfit.cpp`. |
| [`ThornArtifact.cpp`](ThornArtifact.cpp.md) | The "thorn" artefact: a named artefact type with no behaviour of its own. |
| [`ThornArtifact.h`](ThornArtifact.h.md) | Declares the "thorn" artefact implemented in `ThornArtifact.cpp`. |
| [`Torch.cpp`](Torch.cpp.md) | The flashlight: three render objects that must follow the player's gaze rather than his model, plus — for historical reasons — the switch that drives night vision. |
| [`Torch.h`](Torch.h.md) | Declares the flashlight and its night-vision companion, implemented in `Torch.cpp`. |
| [`trade.cpp`](trade.cpp.md) | The trading session: who the two parties are, and the fact that one is open. |
| [`trade.h`](trade.h.md) | Declares the trading session and the three kinds of party. |
| [`trade2.cpp`](trade2.cpp.md) | What an item costs, and what happens when it changes hands. |
| [`trade_action_parameters.h`](trade_action_parameters.h.md) | One trading action's configuration: what it is priced at, what it refuses, and what it falls back to. |
| [`trade_action_parameters_inline.h`](trade_action_parameters_inline.h.md) | Delegation to the two tables, and the fallback pair. |
| [`trade_bool_parameters.h`](trade_bool_parameters.h.md) | A set of item sections a party refuses to deal in. |
| [`trade_bool_parameters_inline.h`](trade_bool_parameters_inline.h.md) | The three operations of the refusal set. |
| [`trade_factor_parameters.h`](trade_factor_parameters.h.md) | Per-item-section price multiplier pairs: the sections a party *will* deal in, and on what terms. |
| [`trade_factor_parameters_inline.h`](trade_factor_parameters_inline.h.md) | The four operations of the per-section multiplier table. |
| [`trade_factors.h`](trade_factors.h.md) | A price multiplier pair: what this costs a friend, and what it costs an enemy. |
| [`trade_factors_inline.h`](trade_factors_inline.h.md) | Construction and reads of the price multiplier pair, each guarded against an invalid number. |
| [`trade_parameters.cpp`](trade_parameters.cpp.md) | Loading the *show* action: which items a party will not even display in the trade screen. |
| [`trade_parameters.h`](trade_parameters.h.md) | One party's complete trading configuration: buy, sell and show, and the fallback chain every lookup walks. |
| [`trade_parameters_inline.h`](trade_parameters_inline.h.md) | The fallback chain, and the loader that turns a configuration section into prices. |
| [`WeaponUpgrade.cpp`](WeaponUpgrade.cpp.md) | Applies one purchased weapon upgrade by re-reading a configuration section over the weapon's already-loaded parameters, either for real or as a dry run that only reports which parameters the upgrade would touch … |
| [`ZudaArtifact.cpp`](ZudaArtifact.cpp.md) | The "zuda" artefact: an artefact leaf with no behaviour beyond its class identifier and its configuration section. |
| [`ZudaArtifact.h`](ZudaArtifact.h.md) | Declares the behaviourless "zuda" artefact leaf implemented in `ZudaArtifact.cpp`. |

### Weapons and shooting

| File | Role |
|---|---|
| [`Bolt.cpp`](Bolt.cpp.md) | The bolt: a throwable that is never consumed, used to probe for anomalies. |
| [`Bolt.h`](Bolt.h.md) | Declares the bolt implemented in `Bolt.cpp`. |
| [`CustomRocket.cpp`](CustomRocket.cpp.md) | A rocket in flight: a rigid body pushed forward by a simulated motor, trailing light, smoke and a looping sound, that stops dead at the first surface it is not allowed to pass through. |
| [`CustomRocket.h`](CustomRocket.h.md) | Declares the flying, glowing, smoking rocket a launcher fires, implemented in `CustomRocket.cpp`. |
| [`Explosive.cpp`](Explosive.cpp.md) | What it means to explode: a blast wave that tests line of sight with sampled rays, a cloud of simulated fragments, a light, a sound, a decal and a screen shake … |
| [`Explosive.h`](Explosive.h.md) | Declares the explosion behaviour any object can inherit, implemented in `Explosive.cpp`. |
| [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md) | A canister or gas bottle: an inventory item that takes damage like an item, counts down on a fuse once damaged enough, and then explodes. |
| [`ExplosiveItem.h`](ExplosiveItem.h.md) | Declares the fused environmental explosive — canisters, gas bottles — implemented in `ExplosiveItem.cpp`. |
| [`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md) | The rocket that goes off: the flight from one parent, the explosion from another, and the small amount of glue that decides which one hears each event. |
| [`ExplosiveRocket.h`](ExplosiveRocket.h.md) | Declares the rocket that explodes on contact — flight, inventory item and explosion in one object — implemented in `ExplosiveRocket.cpp`. |
| [`ExplosiveScript.cpp`](ExplosiveScript.cpp.md) | Exports the explosive mixin to the script virtual machine with a single method: detonate now. |
| [`F1.h`](F1.h.md) | One shipped grenade model as its own class identifier, with all behaviour inherited and all numbers in configuration. |
| [`fire_disp_controller.cpp`](fire_disp_controller.cpp.md) | Smooths the crosshair's spread: when the weapon's dispersion changes, the displayed value travels to the new one over time instead of jumping. |
| [`fire_disp_controller.h`](fire_disp_controller.h.md) | Declares the crosshair dispersion smoother, implemented in `fire_disp_controller.cpp`. |
| [`firedeps.h`](firedeps.h.md) | The four points and one direction a weapon's firing effects are placed at, recomputed each shot from the current animation pose. |
| [`first_bullet_controller.cpp`](first_bullet_controller.cpp.md) | The multiplayer "first shot is accurate" rule: a player who has held fire for long enough and is moving slowly enough gets one shot at a reduced dispersion. |
| [`first_bullet_controller.h`](first_bullet_controller.h.md) | Declares the multiplayer-only "first shot is accurate" rule implemented in `first_bullet_controller.cpp`. |
| [`Grenade.cpp`](Grenade.cpp.md) | A thrown grenade: an inventory item that is simultaneously a missile and an explosive, with the sleight of hand that the thing that leaves the hand is a second object and the thing in the inventory is destroyed … |
| [`Grenade.h`](Grenade.h.md) | Declares the thrown grenade, implemented in `Grenade.cpp`. |
| [`GrenadeLauncher.cpp`](GrenadeLauncher.cpp.md) | The under-barrel grenade launcher attachment: an inventory item whose entire content is one tuned number, the muzzle velocity it gives a launched grenade. |
| [`GrenadeLauncher.h`](GrenadeLauncher.h.md) | Declares the under-barrel grenade launcher attachment, implemented in `GrenadeLauncher.cpp`. |
| [`hud_item_object.cpp`](hud_item_object.cpp.md) | Joins the inventory-item role and the held-item role into one object, and fixes the order in which the two halves see every event. |
| [`hud_item_object.h`](hud_item_object.h.md) | Declares the base for every inventory item that also has a first-person view: it is simultaneously an inventory item and a held item. |
| [`HudSound.cpp`](HudSound.cpp.md) | The sound bank held items play from: one alias names a set of interchangeable takes, a collection names many aliases, and a layered collection plays several banks at once as one sound. |
| [`HudSound.h`](HudSound.h.md) | Declares the alias-keyed sound bank held items play from, implemented in `HudSound.cpp`. |
| [`Missile.cpp`](Missile.cpp.md) | The thrown item: a held weapon whose "shot" is a second entity it spawns, charges up, and then releases into the world with a velocity. |
| [`Missile.h`](Missile.h.md) | Declares the thrown-item base class implemented in `Missile.cpp`. |
| [`RGD5.h`](RGD5.h.md) | One named grenade type, existing only so that a class identifier in the spawn data has a class to instantiate. |
| [`RocketLauncher.cpp`](RocketLauncher.cpp.md) | The mixin that lets a weapon carry, release and track *entities* rather than bullets — the grenade launcher and the rocket launcher, whose projectiles are real spawned objects with physics and their own network … |
| [`RocketLauncher.h`](RocketLauncher.h.md) | Declares the entity-projectile launcher mixin implemented in `RocketLauncher.cpp`. |
| [`Scope.cpp`](Scope.cpp.md) | Exports the three weapon attachments to the script virtual machine. |
| [`Scope.h`](Scope.h.md) | Declares the telescopic sight attachment, whose only source is the script registration in `Scope.cpp`. |
| [`ShootingObject.cpp`](ShootingObject.cpp.md) | Everything a thing that fires shares: the rate of fire, the damage curve over difficulty, the dispersion cone, the muzzle light, the muzzle flash, smoke, tracer and casing effects, the silencer's multipliers, a … |
| [`ShootingObject.h`](ShootingObject.h.md) | Declares the firing mixin implemented in `ShootingObject.cpp`, and the small record of silencer multipliers. |
| [`shootingObject_dump_impl.cpp`](shootingObject_dump_impl.cpp.md) | Writes a firing object's currently effective ballistics back out as a configuration section, for the balance-tuning tools. |
| [`Silencer.cpp`](Silencer.cpp.md) | The silencer attachment: an inventory item with an explicit, fully empty lifecycle. |
| [`Silencer.h`](Silencer.h.md) | Declares the silencer attachment implemented in `Silencer.cpp`. |
| [`Tracer.cpp`](Tracer.cpp.md) | Draws a bullet in flight: a camera-facing streak along its path, plus a muzzle-facing disc when the round is the player's own. |
| [`Tracer.h`](Tracer.h.md) | Declares the bullet-in-flight renderer implemented in `Tracer.cpp`. |
| [`Weapon.cpp`](Weapon.cpp.md) | The base of every weapon: what a weapon is made of (a magazine of cartridges, up to three attachable addons, a zoom rig, a wear level), where its muzzle is this frame … |
| [`Weapon.h`](Weapon.h.md) | Declares the weapon base implemented across `Weapon.cpp`, `WeaponFire.cpp`, `WeaponDispersion.cpp` and `WeaponUpgrade.cpp`. |
| [`weapon_ammo_dump_impl.cpp`](weapon_ammo_dump_impl.cpp.md) | Writes one chambered round's effective ballistics back out as a configuration section, for the balance-tuning tools. |
| [`weapon_dump_impl.cpp`](weapon_dump_impl.cpp.md) | Writes a weapon's effective handling — sway and recoil, hipfire and aimed — back out as a configuration section, for the balance-tuning tools. |
| [`WeaponAK74.cpp`](WeaponAK74.cpp.md) | The standard assault rifle: a magazined weapon with an under-barrel grenade launcher, and nothing else. |
| [`WeaponAK74.h`](WeaponAK74.h.md) | Declares the standard assault rifle implemented in `WeaponAK74.cpp`. |
| [`WeaponAmmo.cpp`](WeaponAmmo.cpp.md) | A round and a box of rounds: the eleven numbers that decide what a bullet does on impact, and the container that hands them out one at a time. |
| [`WeaponAmmo.h`](WeaponAmmo.h.md) | Declares the cartridge value type and the ammunition box, implemented in `WeaponAmmo.cpp`. |
| [`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md) | An automatic shotgun: the shell-at-a-time reload of `WeaponShotgun.cpp`, grafted onto full-automatic fire instead of semi-automatic. |
| [`WeaponAutomaticShotgun.h`](WeaponAutomaticShotgun.h.md) | Declares the automatic shotgun implemented in `WeaponAutomaticShotgun.cpp`. |
| [`WeaponBinoculars.cpp`](WeaponBinoculars.cpp.md) | Binoculars: a weapon that cannot fire, whose trigger zooms, and which draws a bracket around every living thing the actor can currently see. |
| [`WeaponBinoculars.h`](WeaponBinoculars.h.md) | Declares the binoculars implemented in `WeaponBinoculars.cpp`. |
| [`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md) | The target brackets drawn through binoculars and alive-detector scopes: four corner marks that converge onto each visible creature and, once locked, colour themselves by whether it is an enemy. |
| [`WeaponBinocularsVision.h`](WeaponBinocularsVision.h.md) | Declares the target-bracket overlay implemented in `WeaponBinocularsVision.cpp`. |
| [`weaponBM16.cpp`](weaponBM16.cpp.md) | The double-barrelled shotgun: a weapon whose every animation is chosen by how many shells are still in it. |
| [`weaponBM16.h`](weaponBM16.h.md) | Declares the double-barrelled shotgun: the shotgun behaviour with every animation selector overridden to branch on shells remaining. |
| [`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md) | A semi-automatic firearm: one round per trigger pull, no burst, and the trigger is not released until the shot's cadence has elapsed. |
| [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) | Declares the semi-automatic firing behaviour implemented in `WeaponCustomPistol.cpp`. |
| [`WeaponDispersion.cpp`](WeaponDispersion.cpp.md) | How accurate a weapon is right now: the base cone, what wear and the silencer do to it, and the shooter's own contribution. |
| [`WeaponFire.cpp`](WeaponFire.cpp.md) | One shot: how the round is scattered, what it costs the weapon in wear, and what leaves the barrel. |
| [`WeaponFN2000.cpp`](WeaponFN2000.cpp.md) | The integrated-optic bullpup rifle: a magazined weapon with nothing added. |
| [`WeaponFN2000.h`](WeaponFN2000.h.md) | Declares the integrated-optic bullpup rifle implemented in `WeaponFN2000.cpp`. |
| [`WeaponFORT.h`](WeaponFORT.h.md) | A service pistol: the pistol behaviour under a distinct class name, with nothing added. |
| [`WeaponGroza.cpp`](WeaponGroza.cpp.md) | The bullpup assault rifle: a magazined weapon with an under-barrel grenade launcher, and nothing else. |
| [`WeaponGroza.h`](WeaponGroza.h.md) | Declares the bullpup assault rifle implemented in `WeaponGroza.cpp`. |
| [`WeaponHPSA.cpp`](WeaponHPSA.cpp.md) | A compact self-loading pistol: the pistol behaviour under its own class name. |
| [`WeaponHPSA.h`](WeaponHPSA.h.md) | Declares the compact self-loading pistol implemented in `WeaponHPSA.cpp`. |
| [`WeaponHUD.h`](WeaponHUD.h.md) | The superseded first-person weapon model: a shared animated visual, four measured points on it, and an animation timer that calls back when a motion ends. |
| [`WeaponKnife.cpp`](WeaponKnife.cpp.md) | The knife: two distinct attacks, each landing on an animation marker, each resolved as a small burst of short-range hits aimed at the bone shapes inside a sphere in front of the player. |
| [`WeaponKnife.h`](WeaponKnife.h.md) | Declares the knife implemented in `WeaponKnife.cpp`. |
| [`WeaponLR300.cpp`](WeaponLR300.cpp.md) | A western assault rifle: a magazined weapon with nothing added. |
| [`WeaponLR300.h`](WeaponLR300.h.md) | Declares the western assault rifle implemented in `WeaponLR300.cpp`. |
| [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) | The firing state machine every conventional firearm runs: draw, idle, burst, jam, reload, holster — plus the addon attachment rules, the fire-mode selector and the ammunition accounting that reload performs. |
| [`WeaponMagazined.h`](WeaponMagazined.h.md) | Declares the firing state machine implemented in `WeaponMagazined.cpp`. |
| [`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md) | A rifle with an under-barrel grenade launcher: two complete weapons in one object, switched by swapping every ammunition field between a primary set and a secondary set. |
| [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) | Declares the rifle with an under-barrel grenade launcher, implemented in `WeaponMagazinedWGrenade.cpp`. |
| [`WeaponPistol.cpp`](WeaponPistol.cpp.md) | A pistol: a semi-automatic weapon whose slide locks back when empty, so every animation has an empty variant. |
| [`WeaponPistol.h`](WeaponPistol.h.md) | Declares the pistol implemented in `WeaponPistol.cpp`. |
| [`WeaponPM.cpp`](WeaponPM.cpp.md) | The starting sidearm: the pistol behaviour under its own class name. |
| [`WeaponPM.h`](WeaponPM.h.md) | Declares the starting sidearm implemented in `WeaponPM.cpp`. |
| [`WeaponRevolver.cpp`](WeaponRevolver.cpp.md) | A revolver: a semi-automatic-feeling handgun whose reload animation depends on how many rounds are still in the cylinder. |
| [`WeaponRevolver.h`](WeaponRevolver.h.md) | Declares the revolver implemented in `WeaponRevolver.cpp`. |
| [`WeaponRG6.cpp`](WeaponRG6.cpp.md) | The revolving grenade launcher: a shotgun's shell-at-a-time reload feeding a launcher that spawns a real, physically simulated grenade for every round loaded. |
| [`WeaponRG6.h`](WeaponRG6.h.md) | Declares the revolving grenade launcher implemented in `WeaponRG6.cpp`. |
| [`WeaponRPG7.cpp`](WeaponRPG7.cpp.md) | The rocket launcher: a single-shot weapon whose loaded round is a visible rocket on the model and a real object in the world at the same time. |
| [`WeaponRPG7.h`](WeaponRPG7.h.md) | Declares the rocket launcher implemented in `WeaponRPG7.cpp`. |
| [`WeaponScript.cpp`](WeaponScript.cpp.md) | Declares the whole weapon class hierarchy to the script virtual machine, so that scripts can identify and cast between weapon kinds. |
| [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md) | A manually cycled shotgun: one shell per trigger pull, and a reload the player can abort mid-way because it loads a shell at a time. |
| [`WeaponShotgun.h`](WeaponShotgun.h.md) | Declares the manually cycled shotgun implemented in `WeaponShotgun.cpp`. |
| [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md) | The mounted machine gun: a world object the player climbs into rather than carries, whose barrel chases a desired direction through two hinge joints, and whose camera is a bone on its own model. |
| [`WeaponStatMgun.h`](WeaponStatMgun.h.md) | Declares the mounted machine gun, implemented across `WeaponStatMgun.cpp`, `WeaponStatMgunFire.cpp` and `WeaponStatMgunIR.cpp`. |
| [`WeaponStatMgunFire.cpp`](WeaponStatMgunFire.cpp.md) | The mounted gun's shot: an unlimited-ammunition burst on a fixed cadence, a camera kick built per shot, and a heat model that stops the gun firing until it cools. |
| [`WeaponStatMgunIR.cpp`](WeaponStatMgunIR.cpp.md) | The mounted gun's input: every pointing device funnelled into one procedure that moves the desired direction, never the barrel. |
| [`WeaponSVD.cpp`](WeaponSVD.cpp.md) | A semi-automatic sniper rifle: one shot per pull, and the weapon stays locked for the whole length of the shot animation. |
| [`WeaponSVD.h`](WeaponSVD.h.md) | Declares the semi-automatic sniper rifle implemented in `WeaponSVD.cpp`. |
| [`WeaponSVU.h`](WeaponSVU.h.md) | The bullpup marksman rifle: a semi-automatic weapon with no behaviour of its own. |
| [`WeaponUSP45.h`](WeaponUSP45.h.md) | A heavy pistol: the pistol behaviour under a distinct class name, with nothing added. |
| [`WeaponVal.cpp`](WeaponVal.cpp.md) | Constructs the VAL silenced rifle as a magazine-fed weapon that reports itself to the sound-perception layer as a submachine gun. |
| [`WeaponVal.h`](WeaponVal.h.md) | Declares the VAL rifle leaf implemented in `WeaponVal.cpp`. |
| [`WeaponVintorez.cpp`](WeaponVintorez.cpp.md) | Constructs the Vintorez silenced sniper rifle as a magazine-fed weapon that reports itself to the sound-perception layer as a sniper rifle. |
| [`WeaponVintorez.h`](WeaponVintorez.h.md) | Declares the Vintorez rifle leaf implemented in `WeaponVintorez.cpp`. |
| [`WeaponWalther.h`](WeaponWalther.h.md) | A pistol leaf that exists only to give the class-identifier factory a distinct tag to construct. |

### Zones and anomalies

| File | Role |
|---|---|
| [`AmebaZone.cpp`](AmebaZone.cpp.md) | An anomaly that slows everything living inside it and, when it discharges, throws them upward. |
| [`AmebaZone.h`](AmebaZone.h.md) | Declares the slowing, upward-throwing anomaly implemented in `AmebaZone.cpp`. |
| [`CustomZone.cpp`](CustomZone.cpp.md) | The anomaly base class: a volume that notices what is inside it, cycles through idle, waking, blowout and recharge, hits everything in range on the blowout, and occasionally leaves an artefact behind. |
| [`CustomZone.h`](CustomZone.h.md) | Declares the anomaly base class implemented in `CustomZone.cpp`, its five states, its twenty configured flags and the per-resident record. |
| [`FryupZone.cpp`](FryupZone.cpp.md) | An anomaly class that exists only as a name in the class-identifier table: it inherits everything and overrides nothing. |
| [`FryupZone.h`](FryupZone.h.md) | Declares the empty anomaly leaf implemented in `FryupZone.cpp`. |
| [`HairsZone.cpp`](HairsZone.cpp.md) | The "hairs" anomaly: a visible zone that wakes only when something inside it moves fast enough, then hits everything it holds in a random upward direction. |
| [`HairsZone.h`](HairsZone.h.md) | Declares the motion-triggered anomaly implemented in `HairsZone.cpp`. |
| [`Mincer.cpp`](Mincer.cpp.md) | The meat-grinder anomaly: a gravity zone that lifts everything caught inside it into a whirlwind, holds them there, and tears them apart when it discharges. |
| [`Mincer.h`](Mincer.h.md) | Declares the meat-grinder anomaly implemented in `Mincer.cpp`. |
| [`NoGravityZone.cpp`](NoGravityZone.cpp.md) | An anomaly that switches gravity off for everything inside it, and gives each object one upward nudge on the way in so that it actually leaves the ground. |
| [`NoGravityZone.h`](NoGravityZone.h.md) | Declares the weightlessness anomaly implemented in `NoGravityZone.cpp`. |
| [`RadioactiveZone.cpp`](RadioactiveZone.cpp.md) | The radiation anomaly: a zone that delivers a steady, distance-scaled dose in fixed time quanta rather than in discrete hits. |
| [`RadioactiveZone.h`](RadioactiveZone.h.md) | Declares the radiation anomaly implemented in `RadioactiveZone.cpp`. |
| [`smart_zone.h`](smart_zone.h.md) | A restrictor volume that additionally asks to be woken every frame, so that a script can watch who is inside it. |
| [`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md) | Leaving an anomaly and probing for one — the two behaviours that keep a stalker alive in a world where the ground kills you. |
| [`stalker_anomaly_actions.h`](stalker_anomaly_actions.h.md) | Declares the two things a stalker does about an anomaly: leave the one it is standing in, and probe for the one it suspects. |
| [`stalker_anomaly_planner.cpp`](stalker_anomaly_planner.cpp.md) | The anomaly branch: two propositions, two actions, and the rule that a sub-planner must publish its conclusion to the planner above it. |
| [`stalker_anomaly_planner.h`](stalker_anomaly_planner.h.md) | Declares the anomaly sub-planner — the branch of a stalker's brain that deals with hazardous ground. |
| [`team_base_zone.cpp`](team_base_zone.cpp.md) | A multiplayer capture zone: an authored volume that reports, authoritatively, when a player of any team enters or leaves it. |
| [`team_base_zone.h`](team_base_zone.h.md) | Declares the multiplayer base capture zone. |
| [`TeleWhirlwind.cpp`](TeleWhirlwind.cpp.md) | The whirlwind anomaly's grip: it drags loose objects along the ground into a funnel, lifts and spins them at the centre, destroys what is fragile enough, and throws the rest back out. |
| [`TeleWhirlwind.h`](TeleWhirlwind.h.md) | Declares the whirlwind grip and the per-object entry it holds, implemented in `TeleWhirlwind.cpp`. |
| [`TorridZone.cpp`](TorridZone.cpp.md) | The burner anomaly that moves: the same damaging zone as its static parent, driven along an authored path so that it drifts through the level. |
| [`TorridZone.h`](TorridZone.h.md) | Declares the moving burner anomaly implemented in `TorridZone.cpp`. |
| [`UIZoneMap.cpp`](UIZoneMap.cpp.md) | The minimap: a rotating slice of the level's map texture under a fixed centre mark, with a compass, a clock and a contacts counter. |
| [`UIZoneMap.h`](UIZoneMap.h.md) | Declares the minimap implemented in `UIZoneMap.cpp`. |
| [`zone_effector.cpp`](zone_effector.cpp.md) | Fades a full-screen post-process over the player as they walk into an anomaly, in proportion to how deep in they are and how much their suit protects them. |
| [`zone_effector.h`](zone_effector.h.md) | Declares the anomaly's screen effect: a named post-process, two radius fractions, and the strength the camera reads back each frame. |
| [`ZoneCampfire.cpp`](ZoneCampfire.cpp.md) | A campfire: an anomaly zone that scripts can switch on and off, cross-fading its light, its particles and its sound over three seconds instead of popping. |
| [`ZoneCampfire.h`](ZoneCampfire.h.md) | Declares the switchable campfire zone implemented in `ZoneCampfire.cpp`. |
| [`ZoneVisual.cpp`](ZoneVisual.cpp.md) | An anomaly whose blowout is an animation: it idles on one motion and plays an attack motion at authored offsets within the blowout timeline. |
| [`ZoneVisual.h`](ZoneVisual.h.md) | Declares the animated anomaly implemented in `ZoneVisual.cpp`. |

### Physics, vehicles and damage

| File | Role |
|---|---|
| [`ActivatingCharCollisionDelay.cpp`](ActivatingCharCollisionDelay.cpp.md) | Retries creating a creature's walking collision capsule until it can be placed somewhere it does not overlap the world. |
| [`ActivatingCharCollisionDelay.h`](ActivatingCharCollisionDelay.h.md) | Declares the capsule-creation retry timer implemented in `ActivatingCharCollisionDelay.cpp`. |
| [`BoneProtections.cpp`](BoneProtections.cpp.md) | The per-bone armour table an outfit or helmet is described by: for each named bone, how much damage it deflects, how much penetration it stops, and whether bullets pass through it at all. |
| [`BoneProtections.h`](BoneProtections.h.md) | Declares the per-bone armour table built in `BoneProtections.cpp`. |
| [`BreakableObject.cpp`](BreakableObject.cpp.md) | A scenery object that is one rigid piece until it is hit hard enough, then becomes a pile of independently falling pieces that clean themselves up after a while. |
| [`BreakableObject.h`](BreakableObject.h.md) | Declares the two-state breakable scenery object implemented in `BreakableObject.cpp`. |
| [`CaptureBoneCallback.h`](CaptureBoneCallback.h.md) | The interface a caller implements to decide which bone of a ragdoll may be grabbed. |
| [`Car.cpp`](Car.cpp.md) | The drivable vehicle: an engine model that turns a pedal into wheel torque through a gearbox, a body made of physics joints whose wheels and doors take damage individually, and a seat the player rides in. |
| [`Car.h`](Car.h.md) | Declares the vehicle and its four nested part records — wheel, door, exhaust, sound — implemented across `Car.cpp` and its five siblings. |
| [`car_memory.cpp`](car_memory.cpp.md) | Gives a vehicle a pair of eyes, so that it can see the actor and nothing else. |
| [`car_memory.h`](car_memory.h.md) | Declares the vehicle's vision client implemented in `car_memory.cpp`. |
| [`CarCameras.cpp`](CarCameras.cpp.md) | The three vehicle cameras and the one rule that distinguishes them: in first person the rider's head follows the camera, everywhere else it does not. |
| [`CarDamageParticles.cpp`](CarDamageParticles.cpp.md) | The smoke a damaged vehicle emits: two severity levels, each a named effect played from an authored set of bones. |
| [`CarDamageParticles.h`](CarDamageParticles.h.md) | Declares the record holding a vehicle's damage-smoke effect names and emitter bones, implemented in `CarDamageParticles.cpp`. |
| [`CarDoors.cpp`](CarDoors.cpp.md) | A vehicle door as a motor-driven hinge with five states, and the harder half: the geometry that answers "can a person get in or out through this doorway". |
| [`CarExhaust.cpp`](CarExhaust.cpp.md) | An exhaust emitter: a particle effect pinned to a bone of a moving rigid body, and given that body's local velocity so the smoke is left behind rather than dragged along. |
| [`CarInput.cpp`](CarInput.cpp.md) | The vehicle's control surface: which action does what at the wheel, how an analogue stick becomes the same three-valued steering a key produces, and how a script drives a car with no driver. |
| [`CarLights.cpp`](CarLights.cpp.md) | A vehicle headlight: a spot light, a glow sprite and a bone that is only drawn while the light is on, kept together so that switching one switches all three. |
| [`CarLights.h`](CarLights.h.md) | Declares the vehicle headlight and the group it belongs to, implemented in `CarLights.cpp`. |
| [`CarScript.cpp`](CarScript.cpp.md) | The vehicle as scripts see it: a turret to aim and fire, an engine to start and stop, a fuel tank to read and write, and a way to blow the whole thing up. |
| [`CarSound.cpp`](CarSound.cpp.md) | the car's engine-audio state machine — a small four-state automaton that turns the vehicle's mechanical state into a start sample, a pitched loop and a stop sample. |
| [`CarWeapon.cpp`](CarWeapon.cpp.md) | a turret bolted to a vehicle's skeleton — it slews two bones toward a desired direction at a rate-limited speed, refuses to fire until it is on target, and fires through the shared shooting machinery. |
| [`CarWeapon.h`](CarWeapon.h.md) | declares the turret weapon a vehicle carries — the surface implemented in `CarWeapon.cpp`. |
| [`CarWheels.cpp`](CarWheels.cpp.md) | the vehicle's wheels — how a skeleton bone with a wheel joint becomes a driven, steered, braked and damageable axle, and how damage is expressed as joint softening. |
| [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) | The transition from a walking character to a falling body: while alive the creature is an upright capsule steered by its movement controller, and on death it becomes a ragdoll built from its own skeleton … |
| [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) | Declares the alive-capsule / dead-ragdoll boundary implemented in `CharacterPhysicsSupport.cpp`. |
| [`ClimableObject.cpp`](ClimableObject.cpp.md) | A ladder: an oriented box placed in the level that publishes the geometric questions a climbing character needs answered — where is the axis, which way is up it, am I in front of it, how far to the top … |
| [`ClimableObject.h`](ClimableObject.h.md) | Declares the ladder implemented in `ClimableObject.cpp` and the geometric questions it answers for a climbing character. |
| [`DamagableItem.cpp`](DamagableItem.cpp.md) | Turns a continuous health value into a small number of discrete damage levels, and fires one step per level crossed so that visible damage stages are never skipped. |
| [`DamagableItem.h`](DamagableItem.h.md) | Declares the staged-damage mixin implemented in `DamagableItem.cpp`. |
| [`damage_manager.cpp`](damage_manager.cpp.md) | Per-bone damage multipliers: how much a hit on this bone hurts, how much it wounds, and whether the first aimed shot gets its own number. |
| [`damage_manager.h`](damage_manager.h.md) | Declares the per-bone damage multiplier mix-in, implemented in `damage_manager.cpp`. |
| [`DBG_Car.cpp`](DBG_Car.cpp.md) | the vehicle's developer overlay — a live readout of the drivetrain and three plots of the engine's power and torque curves against revolutions. |
| [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) | A physics prop that breaks: it accumulates damage through the armour and per-bone scaling tables, and on reaching zero replaces itself with a pre-authored broken version, with a sound and a burst of particles o … |
| [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) | Declares the breakable physics prop implemented in `DestroyablePhysicsObject.cpp`. |
| [`DynamicHeightMap.cpp`](DynamicHeightMap.cpp.md) | A scrolling cache of ground height around the camera, rebuilt a slot at a time by raycasting straight down onto the level's static collision geometry. Dead code in this branch. |
| [`DynamicHeightMap.h`](DynamicHeightMap.h.md) | Declares the camera-following ground-height cache implemented in `DynamicHeightMap.cpp`, and fixes its dimensions as constants. |
| [`HangingLamp.cpp`](HangingLamp.cpp.md) | A light fixture that can be shot out: up to three render lights riding on named bones of a physically-jointed model, with a health value and a bone whose destruction kills the lamp. |
| [`HangingLamp.h`](HangingLamp.h.md) | Declares the destructible light fixture, implemented in `HangingLamp.cpp`. |
| [`Helicopter.cpp`](Helicopter.cpp.md) | The attack helicopter's core: configuration, spawn, the fixed-rate flight integration that moves it along a path, and persistence. The flying is here; the fighting and the path building are in its siblings. |
| [`helicopter.h`](helicopter.h.md) | Declares the attack helicopter: a scripted flying gun platform with its own flight, body-attitude and target-tracking sub-states. |
| [`Helicopter2.cpp`](Helicopter2.cpp.md) | The helicopter's damage model, death handover and script-facing controls, plus the two small state records that hold its target and its airframe attitude … |
| [`HelicopterMovementManager.cpp`](HelicopterMovementManager.cpp.md) | Where the helicopter is going: path following over authored patrol paths, a procedurally built orbit, the arrival test, terrain-following altitude, and the speed-dependent turn rates the flight model asks for. |
| [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) | The helicopter's guns: a turret aimed within the model's own joint limits, a cannon whose burst pattern is a line of fire walked across the target, and a pair of rocket pods fired in a configured cadence. |
| [`Hit.cpp`](Hit.cpp.md) | One damage event, as it travels from the thing that caused it to the thing that receives it — and, unchanged, across the network. |
| [`Hit.h`](Hit.h.md) | Declares the damage event record, implemented in `Hit.cpp`. |
| [`hit_immunity.cpp`](hit_immunity.cpp.md) | The per-damage-type multiplier table: how an outfit, a creature or a vehicle resists each kind of damage differently. |
| [`hit_immunity.h`](hit_immunity.h.md) | Declares the mix-in that gives an object a per-damage-type multiplier table. |
| [`hit_immunity_space.h`](hit_immunity_space.h.md) | Names the fixed-length table type that holds one damage multiplier per damage type. |
| [`hit_memory_manager.cpp`](hit_memory_manager.cpp.md) | The sense of being hurt: a bounded list of who has damaged this creature, how hard, from what direction, and when — the input the brain's danger model reads. |
| [`hit_memory_manager.h`](hit_memory_manager.h.md) | Declares the third sense: memory of who has hurt this creature and from where. |
| [`hit_memory_manager_inline.h`](hit_memory_manager_inline.h.md) | The hit memory's cheap accessors and its construction. |
| [`HitMarker.cpp`](HitMarker.cpp.md) | The directional damage indicator and the grenade warning: full-screen sprites rotated to point at where the hit came from, or at where a live grenade is, fading out on a shared authored curve. |
| [`HitMarker.h`](HitMarker.h.md) | Declares the directional damage and grenade indicators, implemented in `HitMarker.cpp`. |
| [`ik_anim_state.cpp`](ik_anim_state.cpp.md) | Reads the playing animation's footfall markers to decide, for one limb, whether the foot is planted, whether it may be pinned to the ground, and whether two animations are blending across that decision. |
| [`ik_anim_state.h`](ik_anim_state.h.md) | Declares the four booleans that tell the foot-placement solver what the animation is currently doing with this limb. |
| [`ik_calculate_data.cpp`](ik_calculate_data.cpp.md) | Initializes one limb's solve working set. |
| [`ik_calculate_data.h`](ik_calculate_data.h.md) | The working set one limb's foot-placement solve is carried in: the inputs, the persistent state, and the outputs. |
| [`ik_calculate_state.h`](ik_calculate_state.h.md) | The per-limb state the foot-placement solver carries between frames: where the foot was put, what it collided with, and what it is blending toward. |
| [`ik_collide_data.h`](ik_collide_data.h.md) | The three points on a foot that are tested against the ground, and the result of testing them. |
| [`ik_dbg_matrix.cpp`](ik_dbg_matrix.cpp.md) | Captures every intermediate transform of a foot-placement solve into a bounded history. |
| [`ik_dbg_matrix.h`](ik_dbg_matrix.h.md) | Declares the snapshot of every intermediate transform in a foot-placement solve, for offline inspection. |
| [`ik_foot_collider.cpp`](ik_foot_collider.cpp.md) | Finds the surface under a foot by casting three rays — toe, heel and outer edge — and reduces them to a single plane the foot will be aligned to. |
| [`ik_foot_collider.h`](ik_foot_collider.h.md) | Declares the ground test for a foot: three ray queries, the memory that suppresses repeating them, and the reach constants. |
| [`ik_limb_state.cpp`](ik_limb_state.cpp.md) | Converts a saved limb placement between the two bones it can be expressed against. |
| [`ik_limb_state.h`](ik_limb_state.h.md) | The saved state of one limb between frames, and the rule that decides when it has gone stale. |
| [`ik_limb_state_predict.h`](ik_limb_state_predict.h.md) | What one limb knows about its *next* footfall, so the body can start lowering before the foot lands. |
| [`ik_object_shift.cpp`](ik_object_shift.cpp.md) | Moves a character's whole body vertically along a smooth cubic so its feet can reach the ground, with an overshoot guard that degrades to a parabola when the cubic would bounce. |
| [`ik_object_shift.h`](ik_object_shift.h.md) | Declares the smoother that moves a character's whole body up or down so its feet can reach the ground. |
| [`IKLimbsController.cpp`](IKLimbsController.cpp.md) | Foot placement for one character: it hooks into the pose evaluation, corrects each leg onto the ground it is standing on, and lifts or drops the whole body so that every planted foot can reach. |
| [`IKLimbsController.h`](IKLimbsController.h.md) | Declares one character's foot-placement controller, implemented in `IKLimbsController.cpp`. |
| [`imotion_position.cpp`](imotion_position.cpp.md) | Plays a death animation by teleporting the body to each animated pose, but only after speculatively advancing the animation and checking that the poses ahead do not drive it into the world. |
| [`imotion_position.h`](imotion_position.h.md) | Declares the death-animation variant that drives the body by moving it to the animated pose, with a look-ahead that rejects poses which would intersect the world. |
| [`imotion_velocity.cpp`](imotion_velocity.cpp.md) | Plays a death animation by asking the physics solver to reach each animated pose through velocity, so the body can be stopped by the world instead of passing through it. |
| [`imotion_velocity.h`](imotion_velocity.h.md) | Declares the death-animation variant that drives the body by setting velocities rather than positions. |
| [`magic_box3.cpp`](magic_box3.cpp.md) | Whether two oriented boxes overlap, and where the eight corners of one are. |
| [`magic_box3.h`](magic_box3.h.md) | Declares an oriented bounding box: a centre, three orthonormal axes and three half-extents. |
| [`magic_box3_inline.h`](magic_box3_inline.h.md) | Construction and part access for an oriented bounding box. |
| [`magic_minimize_1d.cpp`](magic_minimize_1d.cpp.md) | Finds the smallest value a scalar function takes on an interval, without derivatives, by recursively subdividing and then fitting parabolas once a minimum is trapped. |
| [`magic_minimize_1d.h`](magic_minimize_1d.h.md) | Declares a derivative-free search for the minimum of a scalar function on an interval. |
| [`magic_minimize_1d_inline.h`](magic_minimize_1d_inline.h.md) | Writable access to the scalar minimizer's three tuning values. |
| [`magic_minimize_nd.h`](magic_minimize_nd.h.md) | Declares a derivative-free minimizer over several variables on a box-shaped domain, built from repeated line searches. |
| [`magic_minimize_nd_inline.h`](magic_minimize_nd_inline.h.md) | Minimizes a function of several variables by searching along a set of directions that it improves as it goes, clipping each search to the domain. |
| [`MathUtils.cpp`](MathUtils.cpp.md) | A hand-written ray-versus-capped-cylinder intersection with its own correctness and speed harness, kept beside the math header but wired to nothing. |
| [`MathUtils.h`](MathUtils.h.md) | The game layer's own small math vocabulary: horizontal-plane operations, projections onto axes and planes, ballistic launch solving … |
| [`matrix_utils.h`](matrix_utils.h.md) | Comparing, limiting and differentiating rigid transforms: the arithmetic behind "has this bone moved enough to matter" and "how fast is it moving". |
| [`min_obb.cpp`](min_obb.cpp.md) | Fits the tightest oriented box around a set of points, by searching orientation space for the rotation that minimizes the enclosed volume. |
| [`moving_object.cpp`](moving_object.cpp.md) | One creature's presence in the world's obstacle-avoidance index: registered on construction, unregistered on destruction, reindexed whenever it moves. |
| [`moving_object.h`](moving_object.h.md) | Declares the per-creature record the obstacle-avoidance system tracks, implemented in `moving_object.cpp`. |
| [`moving_object_inline.h`](moving_object_inline.h.md) | The avoidance record's accessors, and the one rule that makes a decision's timestamp mean "since when", not "as of when". |
| [`moving_objects.cpp`](moving_objects.cpp.md) | Owns the spatial index of every moving creature: built per level, maintained by register, unregister and reindex. |
| [`moving_objects.h`](moving_objects.h.md) | Declares the world's obstacle-avoidance system — the index of moving creatures and the solver that decides which of two converging creatures waits. |
| [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) | Sweeps outward from one creature through everyone its path could reach, collects every predicted collision, and assigns each involved creature move or wait. |
| [`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md) | Decides, for one pair of converging creatures, which of the two should be the one that waits. |
| [`moving_objects_impl.h`](moving_objects_impl.h.md) | The six numbers that set how far ahead obstacle avoidance looks, how finely it samples, and how stubborn a waiting creature is. |
| [`moving_objects_inline.h`](moving_objects_inline.h.md) | One accessor: the collisions the avoidance solver resolved last frame. |
| [`moving_objects_static.cpp`](moving_objects_static.cpp.md) | Finds the level furniture a creature's next second of walking would run into, and records it as that creature's static obstacle set. |
| [`ph_shell_interface.h`](ph_shell_interface.h.md) | The one-method interface an object implements to say "I know how to build my own physics body". |
| [`PHCollisionDamageReceiver.cpp`](PHCollisionDamageReceiver.cpp.md) | Turns a physical collision against a particular bone into a damage event, scaled by a per-bone factor authored in the model. |
| [`PHCollisionDamageReceiver.h`](PHCollisionDamageReceiver.h.md) | Declares the collision-to-damage mixin implemented in `PHCollisionDamageReceiver.cpp`. |
| [`PHDebug.cpp`](PHDebug.cpp.md) | The physics visualizer: a deferred draw list that lets code running inside the physics step — on another thread, at a different rate than the frame … |
| [`PHDebug.h`](PHDebug.h.md) | The physics instrument panel: the drawing primitives, the per-step counters and the object tracker used to see what the rigid-body solver is doing. |
| [`PHDestroyable.cpp`](PHDestroyable.cpp.md) | Turning one physically simulated object into several: spawning the debris, waiting for every piece to arrive, and handing each one the momentum of the blow that broke it. |
| [`PHDestroyable.h`](PHDestroyable.h.md) | Declares the mixin that lets a physically simulated object break into separately spawned pieces, implemented in `PHDestroyable.cpp`. |
| [`PHDestroyableNotificate.cpp`](PHDestroyableNotificate.cpp.md) | A freshly spawned piece of debris reporting back to the object it broke off from. |
| [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md) | Declares the debris-piece side of the destruction handshake, implemented in `PHDestroyableNotificate.cpp`. |
| [`PHMovementControl.cpp`](PHMovementControl.cpp.md) | The bridge between "I want to walk there" and a physically simulated body: it turns a desired path or an input acceleration into forces on a character body, and reports back what the body hit on the way. |
| [`PHMovementControl.h`](PHMovementControl.h.md) | Declares the character-movement bridge, implemented in `PHMovementControl.cpp` and `PHMovementDynamicActivate.cpp`. |
| [`PHMovementDynamicActivate.cpp`](PHMovementDynamicActivate.cpp.md) | Changing a character's collision box without pushing it through the world, and remembering when that failed. |
| [`Phrase.cpp`](Phrase.cpp.md) | One line of a conversation: its text, the goodwill it demands, and whether it is a dead end. |
| [`Phrase.h`](Phrase.h.md) | Declares the phrase record, implemented in `Phrase.cpp`. |
| [`PhraseDialog.cpp`](PhraseDialog.cpp.md) | One conversation: a graph of phrases, two speakers taking strict turns, and the rules that decide which replies are offered next. |
| [`PhraseDialog.h`](PhraseDialog.h.md) | Declares the dialog: a shared authored phrase graph plus one conversation's traversal of it, implemented in `PhraseDialog.cpp`. |
| [`PhraseDialogDefs.h`](PhraseDialogDefs.h.md) | Names the shared handle to a conversation and the list-of-dialog-identifiers type, so parties to a dialog can refer to it without depending on its definition. |
| [`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md) | The half of a conversation that belongs to a *participant*: the set of dialogs this character could start, the set it is currently inside … |
| [`PhraseDialogManager.h`](PhraseDialogManager.h.md) | Declares the conversation-participant mixin implemented in `PhraseDialogManager.cpp`. |
| [`PhraseScript.cpp`](PhraseScript.cpp.md) | The script and information-portion gate attached to every phrase and every dialog: the predicates that decide whether a line may be said, and the effects that fire when it is. |
| [`PhraseScript.h`](PhraseScript.h.md) | Declares the per-phrase script and information gate implemented in `PhraseScript.cpp`. |
| [`PHShellCreator.cpp`](PHShellCreator.cpp.md) | Builds a rigid-body shell for an object straight from its skeleton, with no per-object authoring. |
| [`PHShellCreator.h`](PHShellCreator.h.md) | Declares the default physics-shell builder, implemented in `PHShellCreator.cpp`. |
| [`PHSkeleton.cpp`](PHSkeleton.cpp.md) | A skeleton whose joints can break: splitting one articulated body into two objects at a broken bone, and aging the pieces out again. |
| [`PHSkeleton.h`](PHSkeleton.h.md) | Declares the breakable-skeleton mixin, implemented in `PHSkeleton.cpp`. |
| [`PHSoundPlayer.cpp`](PHSoundPlayer.cpp.md) | One collision sound at a time per physical object, chosen by the pair of materials that struck. |
| [`PHSoundPlayer.h`](PHSoundPlayer.h.md) | Declares the per-object collision-sound gate, implemented in `PHSoundPlayer.cpp`. |
| [`physic_item.cpp`](physic_item.cpp.md) | The simplest physical object: a rigid body while loose in the world, nothing at all while carried, and two shell shapes derived from the model's own bounding box. |
| [`physic_item.h`](physic_item.h.md) | Declares the simplest physical object — a thing that falls, bounces and can be carried — implemented in `physic_item.cpp`. |
| [`physic_item_inline.h`](physic_item_inline.h.md) | An empty file: the inline-accessor header the physical item never needed. |
| [`PhysicObject.cpp`](PhysicObject.cpp.md) | The generic physics prop: a level object whose visual is driven by a rigid-body shell of one of four shapes, optionally playing a startup animation, and — in multiplayer … |
| [`PhysicObject.h`](PhysicObject.h.md) | Declares the generic physics prop and the buffer shape its multiplayer interpolation reads from, implemented in `PhysicObject.cpp`. |
| [`physics_game.cpp`](physics_game.cpp.md) | The bridge from a physics contact to what the player sees and hears: a scuff mark, a spark, a thud — chosen by the two surfaces' material pair and gated by impact speed and camera distance. |
| [`physics_game.h`](physics_game.h.md) | Declares the two contact callbacks the physics world calls back into the game with, implemented in `physics_game.cpp`. |
| [`physics_shell_animated.cpp`](physics_shell_animated.cpp.md) | A rigid-body assembly that is driven by an animation rather than solved: it exists so that a skeletally animated thing can push and be pushed without the solver ever taking over its pose. |
| [`physics_shell_animated.h`](physics_shell_animated.h.md) | Declares the animation-driven rigid-body assembly: a physics body that follows a pose instead of being solved for one. Implemented in `physics_shell_animated.cpp`. |
| [`PhysicsGamePars.cpp`](PhysicsGamePars.cpp.md) | Holds the speed thresholds at which a physics collision becomes audible, visible as a decal, or worth spawning particles for. |
| [`PhysicsGamePars.h`](PhysicsGamePars.h.md) | Declares the collision-effect thresholds and volumes defined in `PhysicsGamePars.cpp`. |
| [`PhysicsSkeletonObject.cpp`](PhysicsSkeletonObject.cpp.md) | A level prop whose whole skeleton is a jointed rigid-body assembly — the breakable, hinged scenery: fences, chains, hanging bodies, destructible frames. |
| [`PhysicsSkeletonObject.h`](PhysicsSkeletonObject.h.md) | Declares the jointed breakable prop implemented in `PhysicsSkeletonObject.cpp`. |
| [`pose_extrapolation.cpp`](pose_extrapolation.cpp.md) | Guessing where something will be from the last two places it was: a two-sample history at a fixed rate, and the linear fit through it. |
| [`pose_extrapolation.h`](pose_extrapolation.h.md) | Declares the two-sample pose history and the transform algebra it extrapolates with. Implemented in `pose_extrapolation.cpp`. |
| [`poses_blending.cpp`](poses_blending.cpp.md) | Moving smoothly from one rigid pose to another: rotation on the shortest arc, position in a straight line, over a fixed duration. |
| [`poses_blending.h`](poses_blending.h.md) | Declares the two-pose blend: interpolation between a start and an end transform, and the timed run between them. Implemented in `poses_blending.cpp`. |
| [`raypick.cpp`](raypick.cpp.md) | Runs a script's ray cast against the loaded level and keeps the nearest hit. |
| [`raypick.h`](raypick.h.md) | Declares the script-facing ray cast: a configurable query object and the hit record scripts read back from it. |
| [`SpaceUtils.h`](SpaceUtils.h.md) | Derives a spatial-index bound — centre, half-extents and radius — from a dynamics-library collision space. |
| [`trajectories.cpp`](trajectories.cpp.md) | Does this thrown or jumping thing clear the geometry between here and there? Answered by approximating a parabola with as few straight segments as its curvature allows. |
| [`trajectories.h`](trajectories.h.md) | Declares the ballistic line-of-flight test and the record its diagnostic overlay draws. |
| [`Wound.cpp`](Wound.cpp.md) | One accumulated injury on one bone of a creature: how much of each damage type it has taken, how it heals, and how it survives a save or a network update. |
| [`Wound.h`](Wound.h.md) | Declares the per-bone injury record implemented in `Wound.cpp`. |

### Animation

| File | Role |
|---|---|
| [`aimers_base.cpp`](aimers_base.cpp.md) | The aiming solver: given a bone and something rigidly attached to it that points somewhere, find the rotation of that bone which makes the attached thing point at a target instead. |
| [`aimers_base.h`](aimers_base.h.md) | Declares the shared base of every aimer: the aiming solver, the bone-sampling helper and the skeleton hook. |
| [`aimers_base_inline.h`](aimers_base_inline.h.md) | Samples what a set of bones *would* look like under a candidate animation, on a scratch animation channel, without disturbing the pose the player is currently seeing. |
| [`aimers_bone.h`](aimers_bone.h.md) | Declares the chain aimer: aiming a target by spreading one correction across a fixed-length chain of bones. |
| [`aimers_bone_inline.h`](aimers_bone_inline.h.md) | Distributes one aiming correction over a chain of bones, so a character turns to face a target with its whole spine rather than snapping one joint. |
| [`aimers_weapon.cpp`](aimers_weapon.cpp.md) | Aims a held weapon rather than a bone: works out where the muzzle actually is, from the weapon's configured mounting and fire point, then rotates two of the carrier's bones so the bullet's path passes through t … |
| [`aimers_weapon.h`](aimers_weapon.h.md) | Declares the weapon aimer: two carrier bones, two weapon anchor bones and their parent, and the pair of corrections it produces. |
| [`aimers_weapon_inline.h`](aimers_weapon_inline.h.md) | The weapon aimer's single accessor. |
| [`animation_movement_controller.cpp`](animation_movement_controller.cpp.md) | Lets an animation drive the object's world transform: the root bone's displacement is stripped out of the pose and applied to the object instead. |
| [`animation_movement_controller.h`](animation_movement_controller.h.md) | Declares the root-motion controller implemented in `animation_movement_controller.cpp`. |
| [`animation_utils.cpp`](animation_utils.cpp.md) | Pins one bone to a fixed offset from its parent regardless of what the animation says, and answers whether one bone is an ancestor of another. |
| [`animation_utils.h`](animation_utils.h.md) | Declares the bone-freeze record and the ancestry query implemented in `animation_utils.cpp`. |
| [`death_anims.cpp`](death_anims.cpp.md) | Loads the per-creature table of death animations from configuration and, given a fatal hit, runs the kill-type predicates in order to pick one. |
| [`death_anims.h`](death_anims.h.md) | Declares the three-level table that picks a scripted death animation from the shot that caused it, implemented in `death_anims.cpp` and `death_anims_predicates.cpp`. |
| [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md) | The seven conditions that decide which authored death animation a fatal hit earns, and the geometry that turns a hit direction into one of four body-relative quadrants. |
| [`IKFoot.cpp`](IKFoot.cpp.md) | One foot's geometry and its ground contact: where the toe and heel are, which way the sole faces, and what rotation and shift of the leg's last bone would put the foot flat on the surface under it. |
| [`IKFoot.h`](IKFoot.h.md) | Declares one foot's derived geometry and its ground-contact correction, implemented in `IKFoot.cpp`. |
| [`IKFoot_inl.h`](IKFoot_inl.h.md) | The foot's vector accessors: hand out the toe, heel and sole normal in the reference bone's space, converting through the bind-pose relation between the two foot bones. |
| [`interactive_animation.cpp`](interactive_animation.cpp.md) | An animation driving a physics body that fades itself out as soon as the body pushes into the world too far. |
| [`interactive_animation.h`](interactive_animation.h.md) | Declares an animated physics body that stops its own animation when it hits something. |
| [`interactive_motion.cpp`](interactive_motion.cpp.md) | Plays an authored death animation on a physically simulated body, and gives up into ragdoll the instant the animation stops being possible. |
| [`interactive_motion.h`](interactive_motion.h.md) | Declares the base for a death animation that is allowed to collide with the world and bail out into ragdoll when it cannot continue. |
| [`moving_bones_snd_player.cpp`](moving_bones_snd_player.cpp.md) | Plays a looping sound at a bone, with its pitch driven by how fast that bone is rotating, so machinery sounds like it is working. |
| [`moving_bones_snd_player.h`](moving_bones_snd_player.h.md) | Declares the per-bone motion sound player implemented in `moving_bones_snd_player.cpp`. |
| [`step_manager.cpp`](step_manager.cpp.md) | Turns a walk animation into footsteps: sounds, dust and a camera shake, fired at authored times within the animation rather than driven by the feet. |
| [`step_manager.h`](step_manager.h.md) | Declares the footstep scheduler and the two points a creature class overrides. |
| [`step_manager_defs.h`](step_manager_defs.h.md) | The two records footsteps are described with: the authored schedule and the live per-clip state. |

### Creature machinery: memory, senses and judgement

| File | Role |
|---|---|
| [`action_base.h`](action_base.h.md) | Declares the planner *operator* every creature action derives from: the fixed lifecycle, the world-state access and the edge cost. Behaviour is in `action_base_inline.h`. |
| [`action_base_inline.h`](action_base_inline.h.md) | The planner operator's lifecycle, its hysteresis timer, and its edge cost: what it means for an action to be chosen, run and abandoned. |
| [`action_management_config.h`](action_management_config.h.md) | One switch: whether the planner's actions log their lifecycle. |
| [`action_planner.h`](action_planner.h.md) | Declares the brain: the component that holds a creature's evaluators and actions, searches for a plan from the world as it is to the world as it wants it … |
| [`action_planner_action.h`](action_planner_action.h.md) | Declares the composite that is simultaneously a planner and an action — the mechanism that makes a creature's behaviour a hierarchy of plans rather than one flat list. Behaviour is in `action_planner_action_inl … |
| [`action_planner_action_inline.h`](action_planner_action_inline.h.md) | Makes a sub-plan behave as a single action: its goal is its own declared effect, its update is one step of its inner plan, and its inertia is the action's. |
| [`action_planner_inline.h`](action_planner_inline.h.md) | Runs a creature's decision cycle: re-solve the plan every update, switch actions only when the plan's first step changes, and persist the whole brain across a save. |
| [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) | Decides which squad member reacts to which fallen comrade, so that a squad losing two people does not have everyone shout about the same body. |
| [`agent_corpse_manager.h`](agent_corpse_manager.h.md) | Declares the squad-level assignment of who reacts to which fallen comrade; behaviour is in `agent_corpse_manager.cpp`. |
| [`agent_corpse_manager_inline.h`](agent_corpse_manager_inline.h.md) | Registration and access for the squad's pending-death list. |
| [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) | Target assignment for a squad: pool what every member knows, decide who fights whom, swap assignments until nobody is running past a nearer target, and share the knowledge back. |
| [`agent_enemy_manager.h`](agent_enemy_manager.h.md) | Declares the squad's target assignment; behaviour is in `agent_enemy_manager.cpp`. |
| [`agent_enemy_manager_inline.h`](agent_enemy_manager_inline.h.md) | Construction and accessors for the squad's target assignment. |
| [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) | Registers a live grenade as a danger area for the whole squad, and decides which single member shouts the warning. |
| [`agent_explosive_manager.h`](agent_explosive_manager.h.md) | Declares the squad's live-explosive tracking and reaction assignment; behaviour is in `agent_explosive_manager.cpp`. |
| [`agent_explosive_manager_inline.h`](agent_explosive_manager_inline.h.md) | Construction and accessors for the squad's explosive manager. |
| [`agent_location_manager.cpp`](agent_location_manager.cpp.md) | The squad's shared opinion of places: which spots are dangerous and for how long, and which cover point a member may claim without crowding a comrade. |
| [`agent_location_manager.h`](agent_location_manager.h.md) | Declares the squad's shared danger map and cover arbiter; behaviour is in `agent_location_manager.cpp`. |
| [`agent_location_manager_inline.h`](agent_location_manager_inline.h.md) | Construction, reset, and looking a danger up by the object that caused it. |
| [`agent_manager.cpp`](agent_manager.cpp.md) | The squad brain: one shared decision-making body owned by a group of stalkers, which pools their perception, distributes enemies and corpse reactions among them … |
| [`agent_manager.h`](agent_manager.h.md) | Declares the squad brain and its accessors for the seven subordinate managers. |
| [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) | The four squad-level actions, each a thin shell whose real work is telling the right subordinate managers to run their distribution passes this cycle. |
| [`agent_manager_actions.h`](agent_manager_actions.h.md) | Declares the four squad-level operators. |
| [`agent_manager_inline.h`](agent_manager_inline.h.md) | The squad brain's subordinate accessors, each asserting the part exists before handing it out. |
| [`agent_manager_planner.cpp`](agent_manager_planner.cpp.md) | Wires the squad's planner: four evaluators that read the pooled squad picture, four operators that act on it, and the standing goal "the squad has an order". |
| [`agent_manager_planner.h`](agent_manager_planner.h.md) | Declares the squad planner as the generic planner specialized to the squad manager. |
| [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) | The three squad evaluators: each answers one yes/no question by polling the roster for any member whose own perception already selected something. |
| [`agent_manager_properties.h`](agent_manager_properties.h.md) | Declares the three squad evaluators and the squad-bound aliases of the generic evaluator kinds. |
| [`agent_manager_properties_inline.h`](agent_manager_properties_inline.h.md) | The three squad evaluators' constructors, each a pass-through to the generic base. |
| [`agent_manager_space.h`](agent_manager_space.h.md) | The vocabulary of the squad planner: the four world properties it reasons over and the four operators it plans with. |
| [`agent_member_manager.cpp`](agent_member_manager.cpp.md) | The squad roster: who is in the group, which of them are currently fighting, and the group-wide interlocks (grenade throwing, cover detouring, who may speak) that only make sense across the whole roster. |
| [`agent_member_manager.h`](agent_member_manager.h.md) | Declares the squad roster and its mask vocabulary. |
| [`agent_member_manager_inline.h`](agent_member_manager_inline.h.md) | Roster lookups: index-to-bit and bit-to-index, and the one-line predicates that define squad membership tests. |
| [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) | The squad's shared perception: three lists of remembered objects — seen, heard, hit by — whose per-record squad masks make one member's sighting the whole squad's knowledge. |
| [`agent_memory_manager.h`](agent_memory_manager.h.md) | Declares the squad's shared perception manager and the three list types it propagates masks over. |
| [`agent_memory_manager_inline.h`](agent_memory_manager_inline.h.md) | The bit-deletion primitive behind roster renumbering, plus the shared perception manager's list installation and access. |
| [`control_action.h`](control_action.h.md) | The base an animated-behaviour step derives from: a bound subject and five lifecycle hooks that all default to doing nothing. |
| [`control_action_inline.h`](control_action_inline.h.md) | The do-nothing defaults of the behaviour-step base. |
| [`danger_cover_location.cpp`](danger_cover_location.cpp.md) | A danger location whose position is borrowed from a cover point rather than stored: the place a creature was shot at from, remembered as "that corner". |
| [`danger_cover_location.h`](danger_cover_location.h.md) | Declares the danger location that names a cover point, implemented in `danger_cover_location.cpp` and `danger_cover_location_inline.h`. |
| [`danger_cover_location_inline.h`](danger_cover_location_inline.h.md) | Construction of a cover-point danger location: every base field is set here, so the record is complete the moment it exists. |
| [`danger_explosive.cpp`](danger_explosive.cpp.md) | The record of one live grenade a creature has noticed, and the rule that lets it be matched by object identifier. |
| [`danger_explosive.h`](danger_explosive.h.md) | Declares the tracked-grenade record, implemented in `danger_explosive.cpp` and `danger_explosive_inline.h`. |
| [`danger_explosive_inline.h`](danger_explosive_inline.h.md) | Construction of a tracked-grenade record, and its cheap identity comparison. |
| [`danger_location.cpp`](danger_location.cpp.md) | The one non-inline rule of a danger location: it stops being useful once its interval has run out. |
| [`danger_location.h`](danger_location.h.md) | The interface a "dangerous place" must satisfy: a position, a lifetime, a radius, and the set of squad members it warns. |
| [`danger_location_inline.h`](danger_location_inline.h.md) | Position matching by proximity, the base refusal to match an entity, and the mask accessor. |
| [`danger_manager.cpp`](danger_manager.cpp.md) | One creature's threat list: turns raw perceptions into typed danger records, ages and prunes them, and names the single most urgent one for the brain to react to. |
| [`danger_manager.h`](danger_manager.h.md) | Declares the per-creature danger list implemented in `danger_manager.cpp`. |
| [`danger_manager_inline.h`](danger_manager_inline.h.md) | Construction, the shallow reset, and the field accessors of the danger list. |
| [`danger_object.cpp`](danger_object.cpp.md) | Nothing: the danger record is entirely declared in its header. |
| [`danger_object.h`](danger_object.h.md) | One recorded threat perception: who, where, when, what kind, and through which sense it arrived. |
| [`danger_object_inline.h`](danger_object_inline.h.md) | The danger record's construction, its field reads, and the identity rule that makes a repeated perception one danger instead of many. |
| [`danger_object_location.cpp`](danger_object_location.cpp.md) | A squad warning that follows an object: its position is the object's, it never times out, and it is dropped when that object goes away. |
| [`danger_object_location.h`](danger_object_location.h.md) | Declares the danger location that follows a game object instead of sitting at a fixed point, implemented in `danger_object_location.cpp`. |
| [`danger_object_location_inline.h`](danger_object_location_inline.h.md) | Construction of the object-bound danger location. |
| [`ef_base.h`](ef_base.h.md) | What every evaluation function must provide: a real-valued answer, a declared range, and the rule that turns the answer into one of a few discrete buckets. |
| [`ef_pattern.cpp`](ef_pattern.cpp.md) | Loads a trained evaluation function from its data file and answers by summing one fitted weight per feature pattern. |
| [`ef_pattern.h`](ef_pattern.h.md) | Declares the trained, table-driven evaluation function implemented in `ef_pattern.cpp`, and the indexing scheme its tables are addressed by. |
| [`ef_primary.cpp`](ef_primary.cpp.md) | The leaf evaluation functions: each reads one property off the creature or item currently in the shared parameter block, from whichever of the two worlds — live objects or alife records — is in use. |
| [`ef_primary.h`](ef_primary.h.md) | Declares the thirty primary evaluation functions and, in their constructors, the declared range of each — which is the numbering the trained tables were fitted against. |
| [`ef_storage.cpp`](ef_storage.cpp.md) | Builds every evaluation function at startup, assigns each primary one a fixed numeric identity, and loads the twenty-four trained functions from data files. |
| [`ef_storage.h`](ef_storage.h.md) | Declares the registry of every evaluation function and the shared parameter block they all read their inputs from; implemented in `ef_storage.cpp`, `ef_storage_inline.h` and `ef_storage_script.cpp`. |
| [`ef_storage_inline.h`](ef_storage_inline.h.md) | Choosing which of the two parameter blocks is live, by clearing the other. |
| [`enemy_manager.cpp`](enemy_manager.cpp.md) | Which of the creatures I know about am I fighting right now: the filter that says who counts as an enemy, the score that ranks them, and the hysteresis that stops a creature flicking between two targets. |
| [`enemy_manager.h`](enemy_manager.h.md) | Declares the per-creature enemy selection, implemented in `enemy_manager.cpp` and `enemy_manager_inline.h`. |
| [`enemy_manager_inline.h`](enemy_manager_inline.h.md) | Accessors for the enemy manager, and the one rule among them that is not an accessor: a forced enemy overrides selection entirely. |
| [`GlobalFeelTouch.cpp`](GlobalFeelTouch.cpp.md) | A touch-sense participant that senses nothing and exists only to hold a set of temporarily ignored objects with expiry times. |
| [`GlobalFeelTouch.hpp`](GlobalFeelTouch.hpp.md) | Declares the deny-list-only touch sense implemented in `GlobalFeelTouch.cpp`. |
| [`item_manager.cpp`](item_manager.cpp.md) | Decides which of the items a creature can currently see is worth walking over to pick up. |
| [`item_manager.h`](item_manager.h.md) | Declares a creature's memory of the pickable items it has seen: which ones are worth going for, and which one is currently chosen. |
| [`item_manager_inline.h`](item_manager_inline.h.md) | Dead code: an inline constructor that the implementation file defines again and overrides. |
| [`member_corpse.h`](member_corpse.h.md) | One squad member's corpse, and which surviving member has been assigned to react to it. |
| [`member_corpse_inline.h`](member_corpse_inline.h.md) | Construction and accessors for the squad's corpse-reaction record. |
| [`member_enemy.h`](member_enemy.h.md) | One enemy as a squad sees it: who knows about him, who has been assigned to him, and how likely he is to be the one worth shooting. |
| [`member_enemy_inline.h`](member_enemy_inline.h.md) | Construction, equality and ordering for a squad's shared enemy entry. |
| [`member_order.h`](member_order.h.md) | What the squad has decided about one of its members this cycle: his cover, his target, his share of the enemy list, and the two events he has been told to react to. |
| [`member_order_inline.h`](member_order_inline.h.md) | Construction and field access for a squad member's standing order. |
| [`memory_manager.cpp`](memory_manager.cpp.md) | A creature's whole memory: three senses feeding three judgements, the fixed order in which they advance each frame, and the merged answer to "what do I know about him". |
| [`memory_manager.h`](memory_manager.h.md) | Declares a creature's whole memory: the three senses, and the three derived judgements built on them. |
| [`memory_manager_inline.h`](memory_manager_inline.h.md) | The accessors onto a creature's six memory sub-managers, and the enumeration of remembered objects that count as enemies. |
| [`memory_space.h`](memory_space.h.md) | The record shapes of a creature's memory: what it remembers having seen, heard and been hit by, and how each memory is scoped to a squad. |
| [`memory_space_impl.h`](memory_space_impl.h.md) | How a memory record is filled from a live object: which position is captured, which navigation vertex, and what the previous refresh time becomes. |
| [`object_actions.cpp`](object_actions.cpp.md) | The concrete operators of the object-handling planner: eighteen small state machines that draw, sling, aim, reload, fire, throw and drop whatever a creature is holding. |
| [`object_actions.h`](object_actions.h.md) | Declares the eighteen planner operators through which a creature draws, stows, aims, reloads, fires, throws and drops what it is holding — implemented in `object_actions.cpp`. |
| [`object_actions_inline.h`](object_actions_inline.h.md) | The two bases every object-handling operator derives from: what they clear on setup, and how an operator declares its own completion. |
| [`object_handler.cpp`](object_handler.cpp.md) | The creature's hands: owns the object-handling planner, keeps it in step with the inventory, and answers where a weapon is attached and whether it is slung. |
| [`object_handler.h`](object_handler.h.md) | Declares the creature-side facade over the object-handling planner, implemented in `object_handler.cpp`. |
| [`object_handler_inline.h`](object_handler_inline.h.md) | Three accessors on the object handler. |
| [`object_handler_planner.cpp`](object_handler_planner.cpp.md) | Turns "use this object this way" into a target world state, keeps the operator set in step with the inventory, and rolls the creature's burst rhythm. |
| [`object_handler_planner.h`](object_handler_planner.h.md) | Declares the object-handling planner — one goal-directed search per creature over what its hands are doing — implemented in `object_handler_planner.cpp` and its two item-kind siblings. |
| [`object_handler_planner_impl.h`](object_handler_planner_impl.h.md) | How one 32-bit word names both an item and a property: the packing, and the taking-apart. |
| [`object_handler_planner_inline.h`](object_handler_planner_inline.h.md) | Three accessors: the creature, and the low half of an operator identifier. |
| [`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) | The thrown-object handling model as planner data: six evaluators and six operators, and a throw that is a three-stage gesture. |
| [`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md) | The whole firearm handling model as planner data: thirty evaluators and twenty-five operators, installed per weapon a creature picks up. |
| [`object_handler_space.h`](object_handler_space.h.md) | The planner vocabulary for "how a creature handles the thing in its hands": thirty-eight world properties and thirty-six operators. |
| [`object_manager.h`](object_manager.h.md) | Declares the generic "keep a set of candidates and pick the best one" base, whose whole substance is in `object_manager_inline.h`. |
| [`object_manager_inline.h`](object_manager_inline.h.md) | The candidate-and-winner pattern every per-creature manager is built on: admit, score every frame, keep the minimum. |
| [`object_property_evaluators.cpp`](object_property_evaluators.cpp.md) | The planner's eyes: eleven small observers that turn the live state of a weapon or grenade into the booleans the object-handling search runs on. |
| [`object_property_evaluators.h`](object_property_evaluators.h.md) | Declares the eleven observers through which the object-handling planner reads the real state of a weapon or a thrown object — implemented in `object_property_evaluators.cpp`. |
| [`object_property_evaluators_inline.h`](object_property_evaluators_inline.h.md) | The evaluator base's construction and its one accessor. |
| [`property_evaluator.h`](property_evaluator.h.md) | The planner's question-asking half: an object that answers one boolean question about the world, so the planner can reason about preconditions and effects. |
| [`property_evaluator_const.h`](property_evaluator_const.h.md) | An evaluator whose answer is fixed at construction: the way a planner is told a condition it cannot measure. |
| [`property_evaluator_inline.h`](property_evaluator_inline.h.md) | The default behaviour of an evaluator that overrides nothing. |
| [`property_evaluator_member.h`](property_evaluator_member.h.md) | An evaluator that answers by comparing *another* question's answer to a literal — the planner's way of aliasing and negating conditions. |
| [`property_evaluator_member_inline.h`](property_evaluator_member_inline.h.md) | Implementations of the comparing evaluator. |
| [`property_storage.h`](property_storage.h.md) | The planner's answer board: the current boolean answer to every question a creature's plan is written against. |
| [`property_storage_inline.h`](property_storage_inline.h.md) | Implementations of the planner's answer board. |
| [`sight_action.cpp`](sight_action.cpp.md) | Turns one look order into head and torso target angles, once per sight-manager tick and again every frame for the two orders that need it. |
| [`sight_action.h`](sight_action.h.md) | Declares one look order for a creature: which sight type, its payload, and the per-type execution state it accumulates while running. |
| [`sight_action_inline.h`](sight_action_inline.h.md) | The six ways to state a look order, and the equality rule that decides whether a newly issued order is the one already running. |
| [`sight_control_action.h`](sight_control_action.h.md) | Wraps a look order with the two things the action selector needs from it: a selection weight and a minimum time it must stay chosen. |
| [`sight_control_action_inline.h`](sight_control_action_inline.h.md) | The bodies for the look-order wrapper: construction by copying the order, and the inertia test. |
| [`sight_manager.cpp`](sight_manager.cpp.md) | Advances a creature's head and torso toward the angles its look order asked for, decides when it must turn its feet, and produces the additive bone rotations the animation layer lays over the playing clip. |
| [`sight_manager.h`](sight_manager.h.md) | Declares the per-creature aiming manager: the currently chosen look order, the smoothed bone rotations it produces, and the torso-twist limits that make a creature turn its whole body. |
| [`sight_manager_inline.h`](sight_manager_inline.h.md) | The cheap queries on the aiming manager, the argument-forwarding order constructors, and the switch that arms the bone solver. |
| [`sight_manager_space.h`](sight_manager_space.h.md) | The closed set of ways a creature can be told where to look. |
| [`sight_manager_target.cpp`](sight_manager_target.cpp.md) | Computes what angle a creature should want: aim solves against a point, the direction of travel, and the search over the level graph for the yaw that exposes the least of the map. |
| [`sound_memory_manager.cpp`](sound_memory_manager.cpp.md) | A creature's ear: it weighs an incoming sound by category, compares it against a threshold that rises with every noise and decays back down between them … |
| [`sound_memory_manager.h`](sound_memory_manager.h.md) | Declares a creature's memory of sounds it has heard: what was recorded, how loud a noise has to be to register, and how that threshold decays. |
| [`sound_memory_manager_inline.h`](sound_memory_manager_inline.h.md) | Construction and the small accessors of the sound memory, including the threshold override that lets a creature be temporarily made deaf or sharp-eared. |
| [`sound_user_data_visitor.h`](sound_user_data_visitor.h.md) | The interface a listener implements to interrogate the AI payload attached to a sound it has heard, without knowing what kind of payload it is. |
| [`team_hierarchy_holder.cpp`](team_hierarchy_holder.cpp.md) | One team's squads, created the first time anybody asks for one. |
| [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md) | Declares the middle level of the team/squad/group hierarchy. |
| [`team_hierarchy_holder_inline.h`](team_hierarchy_holder_inline.h.md) | Construction and the two upward accessors. |
| [`vision_client.cpp`](vision_client.cpp.md) | Sight for an entity that needs to see but is not a creature: the frustum pass and the ray-test pass, run on alternate scheduler ticks. |
| [`vision_client.h`](vision_client.h.md) | Declares the standalone vision sensor — a scheduled eye that can be attached to an entity that is not a creature — implemented in `vision_client.cpp`. |
| [`vision_client_inline.h`](vision_client_inline.h.md) | The one accessor of `vision_client` that is worth not going through a call for. |
| [`visual_memory_manager.cpp`](visual_memory_manager.cpp.md) | Decides what a creature can see, how long it takes to notice, and what it remembers having seen. |
| [`visual_memory_manager.h`](visual_memory_manager.h.md) | Declares a creature's sight: the accumulator that decides when an object has been *noticed*, and the set of sightings it goes on believing in afterwards. |
| [`visual_memory_manager_inline.h`](visual_memory_manager_inline.h.md) | The small accessors of visual memory, and the one of them that is not an accessor: aiming a creature's sighting list at its squad's. |
| [`visual_memory_params.cpp`](visual_memory_params.cpp.md) | Loads one vision profile from a configuration section, and decides which of its numbers a monster is allowed to need. |
| [`visual_memory_params.h`](visual_memory_params.h.md) | One tuned vision profile: the ten numbers that decide how fast a creature notices, how far it sees off-axis, and how long it keeps believing. |

### Stalkers

| File | Role |
|---|---|
| [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) | How a stalker decides what to carry: a virtual shopping pass that re-equips a character from everything it can reach, the rule that keeps it from hoarding two of the same weapon class … |
| [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) | Two stalker behaviours for when nothing is happening: the idle stance a stalker holds on a level with no alife simulation, and the walk-over-and-take-it that picks up an item the stalker has noticed. |
| [`stalker_alife_actions.h`](stalker_alife_actions.h.md) | Declares the two idle-time stalker operators: hold station with no alife simulation, and go pick up a noticed item. |
| [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md) | The stalker's default brain: three world properties and three operators covering the only three things a stalker does when nothing is demanding its attention. |
| [`stalker_alife_planner.h`](stalker_alife_planner.h.md) | Declares the stalker's default idle brain as a script-extensible planner that is itself an operator. |
| [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md) | The two operators that carry out what the alife simulation decided off-screen: hold station when there is nothing to do, and travel to the job a smart terrain has assigned. |
| [`stalker_alife_task_actions.h`](stalker_alife_task_actions.h.md) | Declares the two operators that execute an alife assignment: hold station, and travel to the smart terrain's job. |
| [`stalker_animation_callbacks.cpp`](stalker_animation_callbacks.cpp.md) | Aiming: three bones — head, shoulder, spine — are rotated toward the sight target after the pose has been computed, with the weapon's recoil layered on … |
| [`stalker_animation_data.cpp`](stalker_animation_data.cpp.md) | Loads one stalker skeleton's entire animation table by name from three prefix families, once per distinct model. |
| [`stalker_animation_data.h`](stalker_animation_data.h.md) | Declares the three animation tables a stalker model's motions are resolved into, loaded together and shared between every stalker using that model. |
| [`stalker_animation_data_storage.cpp`](stalker_animation_data_storage.cpp.md) | Shares one loaded animation table between every stalker whose model draws on the same motion banks, keyed by the bank list rather than by the model. |
| [`stalker_animation_data_storage.h`](stalker_animation_data_storage.h.md) | Declares the shared cache of loaded stalker animation tables, keyed by motion-bank list. |
| [`stalker_animation_data_storage_inline.h`](stalker_animation_data_storage_inline.h.md) | The process-wide animation-table cache, created the first time anything asks for it. |
| [`stalker_animation_global.cpp`](stalker_animation_global.cpp.md) | The whole-body channel: it yields to any subsystem that has claimed it, and otherwise plays a critical-wound stagger chosen by wound site and weapon, or a panic run. |
| [`stalker_animation_head.cpp`](stalker_animation_head.cpp.md) | The head channel: three-way choice between a talking head, a listening head and a still one, driven by whether this stalker is the one currently speaking. |
| [`stalker_animation_legs.cpp`](stalker_animation_legs.cpp.md) | Locomotion: which direction the legs are carrying the body relative to where it is looking, with hysteresis so a stalker does not flicker between strafes … |
| [`stalker_animation_manager.cpp`](stalker_animation_manager.cpp.md) | Brings a stalker's animation system to a known state on spawn and on every reinitialization, binds it to the shared animation table for its model, and fires one-shot reaction motions. |
| [`stalker_animation_manager.h`](stalker_animation_manager.h.md) | Declares the stalker's animation system: five independent channels — global, head, torso, legs, script — each holding one blend, resolved every frame in a fixed priority order. |
| [`stalker_animation_manager_debug.cpp`](stalker_animation_manager_debug.cpp.md) | Checked-build instrumentation that counts, per animation and per transition between animations, how often each is played … |
| [`stalker_animation_manager_impl.h`](stalker_animation_manager_impl.h.md) | Five shared predicates the animation channels ask about the stalker's current situation, kept in a header because several channel files need them and none owns them. |
| [`stalker_animation_manager_inline.h`](stalker_animation_manager_inline.h.md) | The animation manager's channel slots, script-queue operations, hook accessors, and the one predicate that decides whether the skeleton's blend tracks need advancing at all. |
| [`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md) | The per-frame resolution: a strict three-tier priority ladder — scripted animation, whole-body animation, or the head/torso/legs trio … |
| [`stalker_animation_names.cpp`](stalker_animation_names.cpp.md) | The fragment tables whose cartesian product is a stalker's entire animation set. |
| [`stalker_animation_names.h`](stalker_animation_names.h.md) | The name-fragment tables from which every stalker animation identifier is assembled, plus the critical-wound taxonomy. |
| [`stalker_animation_offsets.cpp`](stalker_animation_offsets.cpp.md) | Loads and serves the per-animation aim correction that keeps a stalker's weapon pointing where the AI thinks it points. |
| [`stalker_animation_offsets.hpp`](stalker_animation_offsets.hpp.md) | Declares the per-animation aim-offset table implemented in `stalker_animation_offsets.cpp`. |
| [`stalker_animation_pair.cpp`](stalker_animation_pair.cpp.md) | One animation channel: decides when to actually start a motion, how to pick among authored variants, and how a whole-body animation is spread across bone parts. |
| [`stalker_animation_pair.h`](stalker_animation_pair.h.md) | Declares one animation channel — the thing that remembers what a body part is playing — implemented in `stalker_animation_pair.cpp`. |
| [`stalker_animation_pair_inline.h`](stalker_animation_pair_inline.h.md) | The trivial half of the animation channel: field access, the stale flag, and the callback list. |
| [`stalker_animation_state.cpp`](stalker_animation_state.cpp.md) | Resolves one body state's worth of animation names into motion handles, once, at model load. |
| [`stalker_animation_state.h`](stalker_animation_state.h.md) | Declares the per-body-state bundle of loaded animations implemented in `stalker_animation_state.cpp`. |
| [`stalker_animation_state_inline.h`](stalker_animation_state_inline.h.md) | Empty: the inline companion to `stalker_animation_state.h` has no content. |
| [`stalker_animation_torso.cpp`](stalker_animation_torso.cpp.md) | Chooses the motion a stalker's upper body plays, from what it is holding, what that item is doing, how the body is postured and how it is moving. |
| [`stalker_base_action.cpp`](stalker_base_action.cpp.md) | The two rules every stalker action obeys on entry and exit, put in one place so no individual action has to remember them. |
| [`stalker_base_action.h`](stalker_base_action.h.md) | Declares the base every stalker planner action derives from. |
| [`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md) | What every combat action shares: when a burst is allowed, how long it is, what the creature shouts, and how it is sent to cover. |
| [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md) | Declares the base shared by every combat action: the firing helpers, the burst-parameter selector and the combat voice lines. |
| [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) | Every leaf behaviour of a firefight: arming, taking cover, shooting, looking out, holding, flanking, fleeing, grenades and the moments either side of the fight. |
| [`stalker_combat_actions.h`](stalker_combat_actions.h.md) | Declares every leaf action a stalker can take in a firefight. |
| [`stalker_combat_actions_inline.h`](stalker_combat_actions_inline.h.md) | Empty. |
| [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) | Twenty operators over twenty-five propositions: the whole of how a stalker fights, expressed as preconditions. |
| [`stalker_combat_planner.h`](stalker_combat_planner.h.md) | Declares the combat branch of a stalker's brain — the largest planner in the engine. |
| [`stalker_danger_by_sound_actions.cpp`](stalker_danger_by_sound_actions.cpp.md) | An unfinished branch: five differently-named actions with one identical body, none of which the planner can reach. |
| [`stalker_danger_by_sound_actions.h`](stalker_danger_by_sound_actions.h.md) | Declares five actions for reacting to a suspicious sound. None of them is ever reached, and all five have the same body. |
| [`stalker_danger_by_sound_planner.cpp`](stalker_danger_by_sound_planner.cpp.md) | A placeholder branch: one evaluator hardwired to false, one operator the author labelled "fake". |
| [`stalker_danger_by_sound_planner.h`](stalker_danger_by_sound_planner.h.md) | Declares the sub-planner for a sound-only threat. |
| [`stalker_danger_grenade_actions.cpp`](stalker_danger_grenade_actions.cpp.md) | Surviving a grenade: the decision to throw the rifle over your shoulder and run, and everything that follows the blast. |
| [`stalker_danger_grenade_actions.h`](stalker_danger_grenade_actions.h.md) | Declares the five steps of surviving a grenade: run, crouch, run again, sweep, stop. |
| [`stalker_danger_grenade_planner.cpp`](stalker_danger_grenade_planner.cpp.md) | The grenade branch: one plan before the blast and another after it, separated by a proposition the world decides. |
| [`stalker_danger_grenade_planner.h`](stalker_danger_grenade_planner.h.md) | Declares the sub-planner for a live grenade. |
| [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) | Five reactions to a threat with a bearing, and the three different cover evaluators that give each of them its character. |
| [`stalker_danger_in_direction_actions.h`](stalker_danger_in_direction_actions.h.md) | Declares the five steps of reacting to a threat you can face: cover, look out, hold, flank, search. |
| [`stalker_danger_in_direction_planner.cpp`](stalker_danger_in_direction_planner.cpp.md) | A five-step chain of pure progress propositions: nothing here is computed from the world except whether the danger still exists. |
| [`stalker_danger_in_direction_planner.h`](stalker_danger_in_direction_planner.h.md) | Declares the sub-planner for a threat with a known bearing. |
| [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md) | The danger branch: one proposition per kind of threat, one sub-planner per kind, and the two reactions that run whatever kind it is. |
| [`stalker_danger_planner.h`](stalker_danger_planner.h.md) | Declares the danger branch of a stalker's brain — the router that picks which kind of danger it is reacting to. |
| [`stalker_danger_planner_inline.h`](stalker_danger_planner_inline.h.md) | Empty. |
| [`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) | Classifying a threat, and the one evaluator that does real work: deciding whether a chosen cover point is still the right one. |
| [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) | Declares the questions a stalker's danger branch may ask about the threat it has selected. |
| [`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md) | What a stalker does about a threat with no direction: take cover, crouch and sweep, then mark the spot so the squad avoids it. |
| [`stalker_danger_unknown_actions.h`](stalker_danger_unknown_actions.h.md) | Declares the three steps of reacting to a threat you cannot locate: get behind something, sweep, then remember the place is dangerous. |
| [`stalker_danger_unknown_planner.cpp`](stalker_danger_unknown_planner.cpp.md) | Three steps in a fixed order — cover, sweep, publish — expressed as preconditions rather than as a sequence. |
| [`stalker_danger_unknown_planner.h`](stalker_danger_unknown_planner.h.md) | Declares the sub-planner for a threat with no known direction. |
| [`stalker_death_actions.cpp`](stalker_death_actions.cpp.md) | The dying reflex: a stalker shot mid-burst keeps firing as it falls, then drops what it was holding and becomes lootable. |
| [`stalker_death_actions.h`](stalker_death_actions.h.md) | Declares the action that runs while a stalker is dying. |
| [`stalker_death_planner.cpp`](stalker_death_planner.cpp.md) | Two operators: die, then be dead. The second does nothing, and that is its purpose. |
| [`stalker_death_planner.h`](stalker_death_planner.h.md) | Declares the death branch of a stalker's brain. |
| [`stalker_decision_space.h`](stalker_decision_space.h.md) | The whole vocabulary of a stalker's brain: every proposition its planners may reason about, every operator that may change one, and the two ways it may aim its eyes. |
| [`stalker_get_distance_actions.cpp`](stalker_get_distance_actions.cpp.md) | Closing the range in bounds: run to the next piece of cover, breathe, run again. |
| [`stalker_get_distance_actions.h`](stalker_get_distance_actions.h.md) | Declares the two actions of breaking off a fight you cannot win at this range: run to cover, then wait there. |
| [`stalker_get_distance_planner.cpp`](stalker_get_distance_planner.cpp.md) | Two operators that alternate forever, until the enemy is close enough to shoot. |
| [`stalker_get_distance_planner.h`](stalker_get_distance_planner.h.md) | Declares the sub-planner for closing on an enemy that is out of effective range. |
| [`stalker_kill_wounded_actions.cpp`](stalker_kill_wounded_actions.cpp.md) | Executing a downed enemy: who is allowed to do it, which weapon does it, what is said first, and the guarantee that it actually happens. |
| [`stalker_kill_wounded_actions.h`](stalker_kill_wounded_actions.h.md) | Declares the five steps of finishing a downed enemy: walk over, aim, say something, shoot, pause. |
| [`stalker_kill_wounded_planner.cpp`](stalker_kill_wounded_planner.cpp.md) | Five steps, one of which may only start once, and a flag telling the combat planner not to interfere. |
| [`stalker_kill_wounded_planner.h`](stalker_kill_wounded_planner.h.md) | Declares the sub-planner that executes a downed enemy. |
| [`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md) | Low cover: a wall you can hide behind or shoot over, but not both, so the choice is made every cycle. |
| [`stalker_low_cover_actions.h`](stalker_low_cover_actions.h.md) | Declares the three actions of fighting from cover that only protects you crouched. |
| [`stalker_low_cover_planner.cpp`](stalker_low_cover_planner.cpp.md) | Pin the creature at the cover, then choose between ducking, shooting and watching — by whether it can see anything. |
| [`stalker_low_cover_planner.h`](stalker_low_cover_planner.h.md) | Declares the sub-planner for fighting from cover that only protects a crouched creature. |
| [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) | The bottom of a stalker's locomotion: choosing a destination it is allowed to stand at, a set of speeds it is allowed to use, and then reading back off the path what it is actually doing. |
| [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md) | Declares the layer that turns "walk there, crouched, alarmed" into a path, a speed and a body orientation. |
| [`stalker_movement_manager_base_inline.h`](stalker_movement_manager_base_inline.h.md) | Field access over the current and target movement records, plus two small predicates. |
| [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md) | What a walking human does about a world that moves: wait for a door, stop for someone in the way, replan around something that appeared, and give up quietly for a second when no path exists. |
| [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md) | Declares the movement manager layer that makes a human walk around obstacles, wait for doors and not get stuck — implemented in `stalker_movement_manager_obstacles.cpp` and `stalker_movement_manager_obstacles_p … |
| [`stalker_movement_manager_obstacles_inline.h`](stalker_movement_manager_obstacles_inline.h.md) | The one accessor of the obstacle layer: its obstacle-aware restrictor. |
| [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md) | Finding a path that survives the walk: plan it, simulate walking it against the obstacles it will meet, and replan until the simulated walk finishes. |
| [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md) | Getting a human into a piece of authored furniture and keeping them there: walking to the entry point, handing the body over to an animation, and the target the planner is driving toward inside. |
| [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) | declares the top layer of a stalker's movement manager — the layer that knows how to enter, traverse and leave a **smart cover**. |
| [`stalker_movement_manager_smart_cover_fov_range.cpp`](stalker_movement_manager_smart_cover_fov_range.cpp.md) | answers "can I see, and can I reach, that position from this loophole" — the questions the combat planner asks before it commits a stalker to a piece of cover. |
| [`stalker_movement_manager_smart_cover_inline.h`](stalker_movement_manager_smart_cover_inline.h.md) | the field accessors of the smart-cover movement layer, split out of the header for compilation reasons only. |
| [`stalker_movement_manager_smart_cover_loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md) | routes a stalker through a smart cover — picks which aperture to enter by, which sequence of transitions to play, and which one to leave by. |
| [`stalker_movement_manager_space.h`](stalker_movement_manager_space.h.md) | the bit-mask vocabulary that names a stalker's locomotion state, so that one integer selects an animation set and a speed. |
| [`stalker_movement_params.cpp`](stalker_movement_params.cpp.md) | the movement-state record — what a stalker's locomotion should look like — and the lazy, time-throttled choice of which loophole of a smart cover it occupies. |
| [`stalker_movement_params.h`](stalker_movement_params.h.md) | declares the record that describes one complete "where and how a stalker should be moving" state. |
| [`stalker_movement_params_inline.h`](stalker_movement_params_inline.h.md) | the small setters of the movement-state record — the ones whose side effects are the interesting part. |
| [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md) | declares the predicate a stalker hands to the cover search so that squadmates do not all pick the same cover point. |
| [`stalker_movement_restriction_inline.h`](stalker_movement_restriction_inline.h.md) | the three-operation filter a stalker hands to the cover search — admit, weight, claim — all delegated to the squad's shared location bookkeeping. |
| [`stalker_planner.cpp`](stalker_planner.cpp.md) | the root of a stalker's brain — the small symbolic world model, and the six mutually exclusive modes of being that compete to satisfy it. |
| [`stalker_planner.h`](stalker_planner.h.md) | declares the root planner of a stalker's brain. |
| [`stalker_planner_inline.h`](stalker_planner_inline.h.md) | the stalker planner's one flag accessor. |
| [`stalker_property_evaluators.cpp`](stalker_property_evaluators.cpp.md) | The twenty questions a stalker's brain is allowed to ask about the world — one small object per question, each answering `true` or `false`. |
| [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) | Declares the stalker's evaluator set and the three shapes an evaluator can take. |
| [`stalker_property_evaluators_inline.h`](stalker_property_evaluators_inline.h.md) | Empty: the inline sidecar the evaluator header reserves and never fills. |
| [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md) | The three actions of the lost-enemy search: walk to his last known place, take a spot that watches it, and sit there until he is written off. |
| [`stalker_search_actions.h`](stalker_search_actions.h.md) | Declares the three actions of the lost-enemy search. |
| [`stalker_search_planner.cpp`](stalker_search_planner.cpp.md) | What a stalker does after the enemy disappears: walk to where he was, find a spot that watches it, and wait there. |
| [`stalker_search_planner.h`](stalker_search_planner.h.md) | Declares the lost-enemy search sub-planner. |
| [`stalker_sound_data.cpp`](stalker_sound_data.cpp.md) | The sidecar a stalker attaches to every sound it emits, so that whoever hears it learns who made it. |
| [`stalker_sound_data.h`](stalker_sound_data.h.md) | Declares the stalker's sound payload. |
| [`stalker_sound_data_inline.h`](stalker_sound_data_inline.h.md) | Construction and emitter access for the stalker sound payload. |
| [`stalker_sound_data_visitor.cpp`](stalker_sound_data_visitor.cpp.md) | How one stalker's noise becomes another stalker's knowledge: hearing a comrade fight tells you there is something to fight. |
| [`stalker_sound_data_visitor.h`](stalker_sound_data_visitor.h.md) | Declares the listener-side handler for a heard stalker. |
| [`stalker_sound_data_visitor_inline.h`](stalker_sound_data_visitor_inline.h.md) | Construction and listener access for the stalker sound visitor. |
| [`stalker_velocity_collection.cpp`](stalker_velocity_collection.cpp.md) | The nineteen movement speeds of one kind of stalker, read out of configuration once. |
| [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md) | Declares the per-section movement speed table. |
| [`stalker_velocity_collection_inline.h`](stalker_velocity_collection_inline.h.md) | The speed lookup, and the legality rules it enforces on the posture it is asked about. |
| [`stalker_velocity_holder.cpp`](stalker_velocity_holder.cpp.md) | One speed table per configuration section, shared by every stalker that uses it. |
| [`stalker_velocity_holder.h`](stalker_velocity_holder.h.md) | Declares the shared registry of per-section speed tables, and its process-wide handle. |
| [`stalker_velocity_holder_inline.h`](stalker_velocity_holder_inline.h.md) | Lazy creation of the process-wide speed-table registry. |

### Monsters and other creatures

| File | Role |
|---|---|
| [`ai_monster_space.h`](ai_monster_space.h.md) | The shared vocabulary of creature behaviour: posture, gait, mental state, the actions a creature can perform with a held object, and the animation categories scripts steer monsters by. |
| [`controller_state_panic_inline.h`](controller_state_panic_inline.h.md) | An unfinished behaviour state: the psychic creature's panic, which selects nothing and does nothing. |
| [`CustomMonster.cpp`](CustomMonster.cpp.md) | The base every thinking creature is built on: it owns the senses, the memory, the movement and the sound player, splits the frame between a slow thinking tick and a fast presentation tick … |
| [`CustomMonster.h`](CustomMonster.h.md) | Declares the thinking-creature base implemented in `CustomMonster.cpp`, and names the five capabilities every creature composes. |
| [`CustomMonster_inline.h`](CustomMonster_inline.h.md) | The small, hot operations of the creature base: a bounded angle turn that cannot overshoot, a normalization that cannot fail, and the accessors for the four subsystems. |
| [`CustomMonster_VCPU.cpp`](CustomMonster_VCPU.cpp.md) | Turning "look that way" into an actual pose: converts a direction into a yaw/pitch pair, and advances the creature's body rotation toward its target at a bounded rate. |
| [`monster_community.cpp`](monster_community.cpp.md) | Which species-group a creature belongs to, and the one square table that answers how any two groups feel about each other. |
| [`monster_community.h`](monster_community.h.md) | Declares the non-human faction: which species-group a creature belongs to, the team it fights on, and the square table of how any two groups feel about each other. Implemented in `monster_community.cpp`. |
| [`rat_state_base.cpp`](rat_state_base.cpp.md) | Binds a rat state to its rat. |
| [`rat_state_base.h`](rat_state_base.h.md) | The interface every rat behaviour state implements: enter, run, leave — over a rat that is handed in once. |
| [`rat_state_base_inline.h`](rat_state_base_inline.h.md) | Construction and the checked access to the bound rat. |
| [`rat_state_manager.cpp`](rat_state_manager.cpp.md) | Runs the rat's state machine: one step per update, with enter and leave fired only on an actual change. |
| [`rat_state_manager.h`](rat_state_manager.h.md) | Declares the rat's pushdown state machine: a table of states, a stack of state identifiers, and the transition rhythm. |
| [`rat_state_manager_inline.h`](rat_state_manager_inline.h.md) | Replacing the current state in place. |
| [`rat_states.cpp`](rat_states.cpp.md) | The rat's whole behaviour: twelve states, each a short priority-ordered test of the world that either hands the machine to another state or acts. |
| [`rat_states.h`](rat_states.h.md) | Declares the twelve behaviour states a rat can be in. |

### Navigation, movement and restrictors

| File | Role |
|---|---|
| [`abstract_location_selector.h`](abstract_location_selector.h.md) | Declares the reusable "pick a good vertex to go to" component whose behaviour is written in `abstract_location_selector_inline.h`. |
| [`abstract_location_selector_inline.h`](abstract_location_selector_inline.h.md) | Chooses a destination vertex by scoring the navigation graph with a caller-supplied evaluator, throttled so that a creature does not re-decide where to go every frame. |
| [`abstract_path_manager.h`](abstract_path_manager.h.md) | Declares the reusable "find and hold a path to a known destination" component whose behaviour is written in `abstract_path_manager_inline.h`. |
| [`abstract_path_manager_inline.h`](abstract_path_manager_inline.h.md) | Finds a route to a known destination, remembers whether that route still stands, and refuses to re-run a search that has already failed between the same two vertices. |
| [`ai_obstacle.cpp`](ai_obstacle.cpp.md) | Fits a tight oriented box around one dynamic object's visible bones and converts it into the set of navigation vertices that object blocks, so the pathfinder can route around a body that was never part of the l … |
| [`ai_obstacle.h`](ai_obstacle.h.md) | Declares the per-object obstacle: the set of navigation vertices one dynamic object blocks. |
| [`ai_obstacle_inline.h`](ai_obstacle_inline.h.md) | The obstacle's laziness: every reader forces the computation first, and a move is the only thing that invalidates it. |
| [`detail_path_builder.h`](detail_path_builder.h.md) | Runs a detail-path build on the engine's parallel work queue and reports the outcome back into the movement manager's state machine. |
| [`detail_path_manager.cpp`](detail_path_manager.cpp.md) | The detail path's lifecycle and its queries: build, validate, report where the follower is, how far is left, and construct the two turning circles a point's heading and speed imply. |
| [`detail_path_manager.h`](detail_path_manager.h.md) | Declares the detail path builder and its geometric vocabulary; implemented in `detail_path_manager.cpp`, `detail_path_manager_smooth.cpp` and `detail_path_manager_inline.h`. |
| [`detail_path_manager_inline.h`](detail_path_manager_inline.h.md) | The detail path's input setters, each of which is also the staleness test, plus the completion rule and the velocity table. |
| [`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md) | The smoothing algorithm: reduce a list of navigation cells to a handful of corners, pull each corner outward to widen its turn … |
| [`detail_path_manager_space.h`](detail_path_manager_space.h.md) | The two things the rest of the engine needs from the detail path: which smoothing style was asked for, and what one point of the finished path carries. |
| [`doors.h`](doors.h.md) | The vocabulary of the door subsystem: the two states a door can be in, and the two constants that size every decision about one. |
| [`doors_actor.cpp`](doors_actor.cpp.md) | One creature's dealings with doors: which doors its path actually goes through, whether each one needs to be open or shut, and releasing each claim once the creature is past. |
| [`doors_actor.h`](doors_actor.h.md) | Declares the per-creature door agent implemented in `doors_actor.cpp`. |
| [`doors_door.cpp`](doors_door.cpp.md) | One door: the two positions its leaf can be in, a claim list of the creatures that want it open or shut, and the rule that restores it to how it was found once they have all gone. |
| [`doors_door.h`](doors_door.h.md) | Declares one door's state machine and geometry, implemented in `doors_door.cpp`. |
| [`doors_manager.cpp`](doors_manager.cpp.md) | The level's door registry: a spatial index of every door, and the query that hands one creature the doors it is about to have an opinion about. |
| [`doors_manager.h`](doors_manager.h.md) | Declares the level-wide door registry implemented in `doors_manager.cpp`. |
| [`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md) | Avoiding other creatures rather than walls: take the navigation cells the traffic registry says are claimed by somebody else, and stand still entirely when the registry has told this creature to wait. |
| [`dynamic_obstacles_avoider.h`](dynamic_obstacles_avoider.h.md) | Declares the moving-obstacle avoider implemented in `dynamic_obstacles_avoider.cpp`. |
| [`dynamic_obstacles_avoider_inline.h`](dynamic_obstacles_avoider_inline.h.md) | Nothing: the dynamic avoider has no inline definitions. |
| [`game_location_selector.h`](game_location_selector.h.md) | Chooses where on the game graph a creature should wander next: either a search toward terrain matching its preferences, or a random walk that does not immediately double back. |
| [`game_location_selector_inline.h`](game_location_selector_inline.h.md) | The random walk across the game graph: count the neighbours that match the creature's terrain preference, pick one uniformly, and never step straight back where you came from. |
| [`game_path_manager.h`](game_path_manager.h.md) | Declares the cross-level path manager: a path over the game graph, walked one vertex at a time, with the behaviour in `game_path_manager_inline.h`. |
| [`game_path_manager_inline.h`](game_path_manager_inline.h.md) | Walking a game-graph path: the index starts at the second vertex, not the first, because the creature is already standing on the first. |
| [`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md) | A group of creatures that pool their perception: joining a group redirects a member's sight, hearing and hit memory into lists the whole group reads. |
| [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md) | Declares the group level of the squad hierarchy: the tier that owns a group's shared perception and its coordination manager. |
| [`group_hierarchy_holder_inline.h`](group_hierarchy_holder_inline.h.md) | The group holder's trivial accessors and its initial state. |
| [`level_location_selector.h`](level_location_selector.h.md) | Declares the level-graph flavour of "find me the best nearby place to stand", by specializing the generic location selector for the fine navigation mesh. |
| [`level_location_selector_inline.h`](level_location_selector_inline.h.md) | Makes a best-place-to-stand search on the fine navigation mesh respect the searcher's restrictors — and temporarily widen them so the answer is reachable. |
| [`level_path_builder.h`](level_path_builder.h.md) | The step of a creature's path pipeline that runs the coarse level-graph search, off the main thread, with a cooldown so a creature that cannot get anywhere stops trying every frame. |
| [`level_path_manager.h`](level_path_manager.h.md) | Declares the level-graph flavour of the path manager: a cached route across one level's navigation mesh, aware of the searcher's restrictors. |
| [`level_path_manager_inline.h`](level_path_manager_inline.h.md) | The level-graph path search: validated at entry, restricted to what the searcher is allowed to walk on, and told to forget its failures when those restrictions change. |
| [`location_manager.cpp`](location_manager.cpp.md) | Where a creature is willing to be: the terrain masks the alife simulation scores cross-level graph vertices against when moving it off-screen. |
| [`location_manager.h`](location_manager.h.md) | Declares a creature's terrain preference: which kinds of game-graph terrain it is willing to travel through, and how strongly. |
| [`location_manager_inline.h`](location_manager_inline.h.md) | Construction and the one accessor of a creature's terrain preference. |
| [`movement_manager.cpp`](movement_manager.cpp.md) | The path pipeline's lifecycle and state driver: what invalidates a path, which stage runs next, and where a moving creature will be a moment from now. |
| [`movement_manager.h`](movement_manager.h.md) | Declares the movement manager — the three-level path pipeline every walking creature moves through — implemented across `movement_manager.cpp` and its four siblings. |
| [`movement_manager_game.cpp`](movement_manager_game.cpp.md) | Drives the path pipeline when the destination is on another level: pick a game vertex, search the coarse graph, then resolve one level-sized leg at a time. |
| [`movement_manager_impl.h`](movement_manager_impl.h.md) | An empty file: the template-body header the movement manager never needed. |
| [`movement_manager_inline.h`](movement_manager_inline.h.md) | The movement manager's accessors, and the three setters that quietly invalidate the path. |
| [`movement_manager_level.cpp`](movement_manager_level.cpp.md) | Drives the path pipeline when the destination is a vertex of the loaded level: search the navigation mesh, smooth it, follow it. |
| [`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md) | Drives the path pipeline when the destination comes from an authored patrol route: pick the next point, walk to it, pick the next. |
| [`movement_manager_physic.cpp`](movement_manager_physic.cpp.md) | Turns the finished detail path into an actual position each frame, by handing a velocity to the physics character when anything is nearby and by teleporting the character when nothing is. |
| [`movement_manager_space.h`](movement_manager_space.h.md) | The four kinds of path a creature can be asked to follow. |
| [`obstacles_query.cpp`](obstacles_query.cpp.md) | Maintains one creature's blocked-vertex set: the union of the navigation vertices its known obstacles cover, kept fresh with a checksum rather than a rebuild. |
| [`obstacles_query.h`](obstacles_query.h.md) | Declares the accumulated obstacle set — a set of objects plus the union of the navigation vertices they block — implemented in `obstacles_query.cpp`. |
| [`obstacles_query_inline.h`](obstacles_query_inline.h.md) | The obstacle set's lazy-recompute protocol: what invalidates the cached area, and what forces it back. |
| [`patrol_path_manager.cpp`](patrol_path_manager.cpp.md) | Walks a creature along an authored waypoint graph: choose where to join it, then at each waypoint pick an outgoing edge by weighted chance, skipping anything the creature is not allowed to reach. |
| [`patrol_path_manager.h`](patrol_path_manager.h.md) | Declares the walker of authored patrol routes, implemented in `patrol_path_manager.cpp`. |
| [`patrol_path_manager_inline.h`](patrol_path_manager_inline.h.md) | Construction and the setters, where changing a policy invalidates the route and changing it to its current value does not. |
| [`quadtree.h`](quadtree.h.md) | A fixed-depth quadtree over the horizontal plane, used to answer "what is near this point" for cover points, doors and moving objects. |
| [`quadtree_inline.h`](quadtree_inline.h.md) | The quadtree's operations: descend by quadrant to a fixed depth, and a radius query that visits only the quadrants a circle actually overlaps. |
| [`refreshable_obstacles_query.h`](refreshable_obstacles_query.h.md) | Declares an obstacles query that periodically widens its own search radius so distant obstacle changes are eventually noticed. |
| [`refreshable_obstacles_query_inline.h`](refreshable_obstacles_query_inline.h.md) | The refresh cadence: a two-metre query normally, a hundred-metre query on the tick — except that the tick test compares the wrong two quantities. |
| [`restricted_object.cpp`](restricted_object.cpp.md) | Where a creature learns where it is allowed to go: builds its restrictor set at spawn, answers accessibility, and installs a temporary border around a path in progress. |
| [`restricted_object.h`](restricted_object.h.md) | Declares the per-creature view of the restrictor system: which restrictors constrain me, is a place accessible, and a temporary border for the duration of one path. |
| [`restricted_object_inline.h`](restricted_object_inline.h.md) | Construction and the three trivial reads. |
| [`restricted_object_obstacle.cpp`](restricted_object_obstacle.cpp.md) | Masks the navigation vertices blocked by other objects out of the graph for the duration of one path — while never masking the path's own endpoints. |
| [`restricted_object_obstacle.h`](restricted_object_obstacle.h.md) | Declares a restricted object that also fences out the navigation vertices currently blocked by other objects. |
| [`seniority_hierarchy_holder.cpp`](seniority_hierarchy_holder.cpp.md) | The root of the command hierarchy: hands out team holders by index, creating each on first demand and owning them all. |
| [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) | Declares the root of the command hierarchy: the registry of teams for one level. |
| [`seniority_hierarchy_holder_inline.h`](seniority_hierarchy_holder_inline.h.md) | Construction and read-only access for the team registry. |
| [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md) | Shared helpers for the four-level command hierarchy: a number-to-text formatter for assertion messages, and a fixed-vector filler. |
| [`setup_manager.h`](setup_manager.h.md) | Declares the generic weighted-random action selector that the sight manager and its kin are built from. |
| [`setup_manager_inline.h`](setup_manager_inline.h.md) | The weighted-random action selector: holds one action running until it finishes, then draws its successor by weight from the applicable others. |
| [`space_restriction.cpp`](space_restriction.cpp.md) | One entity's *effective movement space*: the permitted volumes it must stay inside, minus the forbidden volumes it must stay out of, reduced to a single navigation-mesh border that can be stamped onto the level … |
| [`space_restriction.h`](space_restriction.h.md) | Declares one entity's composed movement restriction: a permitted volume, a forbidden volume, and the merged border between them. |
| [`space_restriction_abstract.h`](space_restriction_abstract.h.md) | The interface every restriction shares: it owns a *border* — the set of navigation vertices on its edge — and can name the subset of that border from which a step across is actually possible. |
| [`space_restriction_abstract_inline.h`](space_restriction_abstract_inline.h.md) | The lazy border build and the accessible-neighbour cache, defined inline because the neighbour walk is a template over the restriction type. |
| [`space_restriction_base.cpp`](space_restriction_base.cpp.md) | Decides what it means for a navigation vertex to be inside a volume — by testing the vertex's four corners and its centre — and fixes the spatial sort order that every border lookup depends on. |
| [`space_restriction_base.h`](space_restriction_base.h.md) | Declares the layer that turns "is this sphere inside the volume" into "is this navigation vertex inside the volume", and fixes the ordering the border list is kept in. |
| [`space_restriction_base_inline.h`](space_restriction_base_inline.h.md) | The checked-build accessor for a restriction's border-connectivity verdict. |
| [`space_restriction_bridge.cpp`](space_restriction_bridge.cpp.md) | The indirection cell every restriction is held through, so that a named restrictor can be swapped from a not-yet-spawned placeholder to real geometry without invalidating anybody's handle … |
| [`space_restriction_bridge.h`](space_restriction_bridge.h.md) | Declares the replaceable, reference-counted cell that every restriction handle points at. |
| [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md) | The nearest-legal-position search: from an illegal point, find the closest navigation vertex on the legal side of a restriction and a point inside that cell to actually stand on. |
| [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md) | Several named restrictors as one volume: the union of their shapes, with a single enclosing sphere for cheap rejection and a border that is the rim of the union rather than the concatenation of the rims. |
| [`space_restriction_composition.h`](space_restriction_composition.h.md) | Declares the union-of-restrictors volume, which doubles as the placeholder for a named restrictor that has not spawned. |
| [`space_restriction_composition_inline.h`](space_restriction_composition_inline.h.md) | Construction of a union volume from a member list, and the three constant answers it gives about itself. |
| [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md) | The level's registry of restrictor volumes: it canonicalizes a comma-joined name list into a cache key, hands out one shared handle per distinct list, swaps geometry in and out as restrictor entities spawn and … |
| [`space_restriction_holder.h`](space_restriction_holder.h.md) | Declares the level's registry of restrictor volumes and the two level-wide default restriction lists. |
| [`space_restriction_holder_inline.h`](space_restriction_holder_inline.h.md) | Construction of an empty registry, and the two accessors for the level-wide default restriction lists. |
| [`space_restriction_inline.h`](space_restriction_inline.h.md) | Stamps and clears the merged border on the level graph, and answers containment in the permitted space with the strictness flag inverted between the two senses. |
| [`space_restriction_manager.cpp`](space_restriction_manager.cpp.md) | Binds entities to restrictions: it keeps each entity's authored restriction lists, merges the level's defaults into them under a conflict rule, resolves the result to one shared restriction object … |
| [`space_restriction_manager.h`](space_restriction_manager.h.md) | Declares the per-entity restriction layer on top of the level's restrictor registry. |
| [`space_restriction_manager_inline.h`](space_restriction_manager_inline.h.md) | Stamps an entity's restriction border onto the level graph ahead of a path search, given the movement about to be planned. |
| [`space_restriction_shape.cpp`](space_restriction_shape.cpp.md) | Turns one restrictor entity's authored spheres and boxes into a border: the set of navigation vertices that straddle the volume's edge, found by scanning the mesh under each primitive's footprint. |
| [`space_restriction_shape.h`](space_restriction_shape.h.md) | Declares the restriction that wraps one restrictor entity's collision volume. |
| [`space_restriction_shape_inline.h`](space_restriction_shape_inline.h.md) | Construction of a shape restriction — which builds its border immediately — and the two per-primitive measurements the scan bounds are derived from. |
| [`space_restrictor.cpp`](space_restrictor.cpp.md) | The client object for a restrictor entity: an invisible, non-simulated volume assembled from authored spheres and boxes, which registers itself with the level's restriction registry on spawn and answers exact c … |
| [`space_restrictor.h`](space_restrictor.h.md) | Declares the restrictor client object: an invisible volume entity, base class of every zone and smart terrain. |
| [`space_restrictor_inline.h`](space_restrictor_inline.h.md) | Construction of a restrictor and the two one-line answers about its kind and its cache validity. |
| [`squad_hierarchy_holder.cpp`](squad_hierarchy_holder.cpp.md) | One squad's slot table of groups, created on first mention so that the three-level team/squad/group hierarchy costs nothing for the combinations a level never uses. |
| [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) | Declares the squad node of the team/squad/group hierarchy creatures are addressed by. |
| [`squad_hierarchy_holder_inline.h`](squad_hierarchy_holder_inline.h.md) | Construction of a squad node — an empty slot table of fixed length and a link to the parent team — and the two accessors. |
| [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md) | Keeps a creature's route honest about the doors, crates and other creatures standing in it — and refuses to adopt an obstacle set that would leave it with nowhere to go. |
| [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md) | Declares the obstacle arbiter and marks the points a subclass may take over. |
| [`static_obstacles_avoider_inline.h`](static_obstacles_avoider_inline.h.md) | Binding, clearing, and the four set accessors. |
| [`steering_behaviour.cpp`](steering_behaviour.cpp.md) | Six forces that pull a flying or driving thing around, and the accumulator that adds them up into one acceleration per frame. |
| [`steering_behaviour.h`](steering_behaviour.h.md) | Declares the six steering forces, the supplier interface each takes its inputs through, and the accumulator. |
| [`steering_behaviour_alignment.h`](steering_behaviour_alignment.h.md) | Declares the alignment force of the abandoned rat flocking design. Never implemented. |
| [`steering_behaviour_base.h`](steering_behaviour_base.h.md) | An abandoned second steering interface, written for rat flocking, never implemented and never included. |
| [`steering_behaviour_base_inline.h`](steering_behaviour_base_inline.h.md) | The enabled flag of the abandoned rat steering interface. |
| [`steering_behaviour_cohesion.h`](steering_behaviour_cohesion.h.md) | Declares the cohesion force of the abandoned rat flocking design. Never implemented. |
| [`steering_behaviour_separation.h`](steering_behaviour_separation.h.md) | Declares the separation force of the abandoned rat flocking design. Never implemented. |

### Smart covers and smart terrains

| File | Role |
|---|---|
| [`cover_evaluators.cpp`](cover_evaluators.cpp.md) | Six ways to score a cover position, each answering a different tactical question, all reduced to "keep the candidate with the lowest number". |
| [`cover_evaluators.h`](cover_evaluators.h.md) | Declares the six cover evaluators implemented in `cover_evaluators.cpp`. |
| [`cover_evaluators_inline.h`](cover_evaluators_inline.h.md) | The evaluators' lifecycle operations and, more importantly, the rule by which a previous answer stays valid. |
| [`cover_manager.cpp`](cover_manager.cpp.md) | Builds the level's static cover point set from the navigation mesh at load time, and maintains the authored smart covers layered over it. |
| [`cover_manager.h`](cover_manager.h.md) | Declares the level's cover index, built in `cover_manager.cpp` and queried by the template search in `cover_manager_inline.h`. |
| [`cover_manager_inline.h`](cover_manager_inline.h.md) | The cover search itself: given a position, a radius and a caller-supplied scoring policy, pick the best cover point — or keep the previous answer when re-deciding would only make the creature twitch. |
| [`cover_point.h`](cover_point.h.md) | One place a creature can stand to be hidden: a position, the navigation vertex it sits on, and whether it is an authored cover or a computed one. |
| [`cover_point_inline.h`](cover_point_inline.h.md) | The cover point's constructor, accessors and approximate equality. |
| [`smart_cover.cpp`](smart_cover.cpp.md) | Places one authored cover on a level: binds each enabled loophole to a navigation vertex, and answers the question the AI actually asks — which loophole should I use against a target standing there. |
| [`smart_cover.h`](smart_cover.h.md) | Declares one smart cover placed on a level: a description, the subset of its loopholes this instance enables, their navigation vertices, and the choice of which loophole to use against a given position. |
| [`smart_cover_action.cpp`](smart_cover_action.cpp.md) | Builds one loophole action from its authored table: the optional movement target, and the animation lists keyed by purpose. |
| [`smart_cover_action.h`](smart_cover_action.h.md) | Declares one thing a creature can do while standing at a loophole: a named set of animation lists, and optionally a position the creature must move to first. |
| [`smart_cover_action_inline.h`](smart_cover_action_inline.h.md) | The accessors on a loophole action, including the animation lookup that names the cover in its failure. |
| [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) | Builds the operator set that defines everything a creature can do inside a smart cover, and owns the in-cover lifecycle around it. |
| [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) | Declares the planner that runs a creature's whole life inside a smart cover: the sub-plan that alternates idling, looking out, firing, reloading and leaving. |
| [`smart_cover_animation_planner_inline.h`](smart_cover_animation_planner_inline.h.md) | The planner's timing accessors, and the two random dwell draws that keep a squad in cover from moving in unison. |
| [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) | Runs one planning cycle per clip boundary, turns the resulting action into a motion the model can play, and reports the clip's third marker back to the action as the moment its effect lands. |
| [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md) | Declares the bridge between the smart-cover plan and the animation layer: it owns the planner, is asked for the next clip when one ends, and reports a marker inside the current clip back to the running action. |
| [`smart_cover_animation_selector_inline.h`](smart_cover_animation_selector_inline.h.md) | The two accessors the smart-cover actions reach the planner and the world state through. |
| [`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md) | The peaceful half of cover behaviour: with no enemy to fight, alternate between staying down and looking out, on the dwell timers. |
| [`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md) | Declares the small planner that decides what a creature in a cover should be doing when it has no enemy: idle, or look out. |
| [`smart_cover_default_behaviour_planner_inline.hpp`](smart_cover_default_behaviour_planner_inline.hpp.md) | The two unused dwell-interval accessors on the default behaviour planner. |
| [`smart_cover_description.cpp`](smart_cover_description.cpp.md) | Loads one kind of smart cover out of the authored script tables: its loopholes, the graph of transitions between them, and the four connectivity guarantees that make the cover usable. |
| [`smart_cover_description.h`](smart_cover_description.h.md) | Declares the shared, authored shape of one *kind* of smart cover: its loopholes and the graph of transitions between them. |
| [`smart_cover_description_inline.h`](smart_cover_description_inline.h.md) | The three field reads on a smart-cover description. |
| [`smart_cover_detail.cpp`](smart_cover_detail.cpp.md) | Reads typed fields out of an authored script table with a hard failure on anything malformed, and names the two pseudo-loopholes that represent standing outside the cover. |
| [`smart_cover_detail.h`](smart_cover_detail.h.md) | Declares the strict readers that pull a smart cover's authored description out of a script table, and the two reserved loophole names that stand for entering and leaving. |
| [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) | The bodies of the smart-cover world-state questions, including the two dwell timers that flip the creature between idling and looking out. |
| [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) | Declares the thirteen world-state questions the smart-cover planners reason over: am I in a cover, is it the one I want, can I leave, has the idle run long enough. |
| [`smart_cover_inline.h`](smart_cover_inline.h.md) | Transforms a loophole's authored local geometry into world space through the placed object's transform, and the four field reads. |
| [`smart_cover_loophole.cpp`](smart_cover_loophole.cpp.md) | Parses one loophole out of its authored table, derives whether it is usable at all, and builds the inner graph of moves between its actions. |
| [`smart_cover_loophole.h`](smart_cover_loophole.h.md) | Declares one firing position inside a smart cover: where it looks, how far and how wide, what a creature can do there, and how to get from one action to another. |
| [`smart_cover_loophole_inline.h`](smart_cover_loophole_inline.h.md) | The field reads on a loophole, plus the action-presence test the planner uses as a precondition. |
| [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) | What a creature does while standing at a loophole: where it looks, which clip it plays, when it pulls the trigger, and how it changes posture. |
| [`smart_cover_loophole_planner_actions.h`](smart_cover_loophole_planner_actions.h.md) | Declares what a creature actually does at a loophole — idle, look out, fire, reload — and the four posture transitions between them. |
| [`smart_cover_loophole_planner_actions_inline.h`](smart_cover_loophole_planner_actions_inline.h.md) | An empty inline sibling. |
| [`smart_cover_object.cpp`](smart_cover_object.cpp.md) | Spawns a placed smart cover: builds its volume from the server record, registers the cover with the cover manager, then switches the entity off so it costs nothing for the rest of the level. |
| [`smart_cover_object.h`](smart_cover_object.h.md) | Declares the placed entity that carries a smart cover: an invisible, non-updating volume whose only jobs are to own a transform, own a shape, and hand the AI a cover. |
| [`smart_cover_object_inline.h`](smart_cover_object_inline.h.md) | The two enemy-distance thresholds and the cover accessor. |
| [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) | The three operators that get a creature from one loophole to another, or out of the cover — and the rule that a smart-cover action's effect lands when its clip ends, not when it is chosen. |
| [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) | Declares the base every smart-cover action shares, and the three operators that move a creature between loopholes and out of a cover. |
| [`smart_cover_planner_actions_inline.h`](smart_cover_planner_actions_inline.h.md) | An empty inline sibling. |
| [`smart_cover_planner_target_provider.cpp`](smart_cover_planner_target_provider.cpp.md) | Hands the animation planner its next goal, and decides when a creature has spent long enough firing that it should drop back into cover. |
| [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md) | Declares the actions whose whole job is to hand the animation planner its next goal, plus the two that also decide when a creature has been firing for too long. |
| [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md) | The top of the smart-cover stack: decides whether a creature in a cover should look out, fire, fire blind, or fall back to peaceful behaviour, and tells the script layer every cycle. |
| [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md) | Declares the sub-planner that decides *which* smart-cover behaviour a creature in cover should currently be pursuing. |
| [`smart_cover_planner_target_selector_inline.h`](smart_cover_planner_target_selector_inline.h.md) | The read accessor for the target selector's script callback. |
| [`smart_cover_storage.cpp`](smart_cover_storage.cpp.md) | Caches parsed smart-cover descriptions so that the same authored cover table is read from configuration once, and frees them only after a grace period. |
| [`smart_cover_storage.h`](smart_cover_storage.h.md) | Declares the process-wide cache of smart-cover *descriptions*, keyed by the configuration table that defines them. |
| [`smart_cover_transition.cpp`](smart_cover_transition.cpp.md) | Builds a smart-cover transition edge from its authored table, asks the script layer whether the edge is currently permitted, and picks the animation that lands the creature in the body state the caller wants. |
| [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) | Declares one edge of a smart cover's transition graph: a guarded set of animations that carries a creature from one loophole to another. |
| [`smart_cover_transition_animation.cpp`](smart_cover_transition_animation.cpp.md) | Constructs the immutable animation entry of a smart-cover transition. |
| [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md) | Declares one animation entry of a smart-cover transition: where the creature stands, what clip plays, and the posture and gait it implies. |
| [`smart_cover_transition_animation_inline.hpp`](smart_cover_transition_animation_inline.hpp.md) | Field reads on a smart-cover transition animation entry, and the one rule that an empty motion name means "posture change only". |

### Dialogue, information and characters

| File | Role |
|---|---|
| [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) | The non-player half of a conversation: given a phrase graph the player has advanced, pick the reply that matches how much this character likes the player, and say it. |
| [`AI_PhraseDialogManager.h`](AI_PhraseDialogManager.h.md) | Declares the non-player conversation mix-in implemented in `AI_PhraseDialogManager.cpp`. |
| [`character_community.cpp`](character_community.cpp.md) | The faction a character belongs to, and the shared tables that say how any two factions feel about each other. |
| [`character_community.h`](character_community.h.md) | Declares the community value and its shared tables, implemented in `character_community.cpp`. |
| [`character_hit_animations.cpp`](character_hit_animations.cpp.md) | Makes a living character flinch from a hit — twisting, doubling over or staggering in the direction the hit came from — over whatever animation is already playing. |
| [`character_hit_animations.h`](character_hit_animations.h.md) | Declares the hit-reaction controller implemented in `character_hit_animations.cpp`. |
| [`character_hit_animations_params.h`](character_hit_animations_params.h.md) | The seven numbers that tune how hard and how often a character flinches. |
| [`character_rank.cpp`](character_rank.cpp.md) | Turns a character's numeric rating into a named rank band, and holds the tables that say how ranks regard each other and what a kill is worth. |
| [`character_rank.h`](character_rank.h.md) | Declares the rank band value and its shared tables, implemented in `character_rank.cpp`. |
| [`character_reputation.cpp`](character_reputation.cpp.md) | Turns a character's numeric reputation into a named band, and holds the table of how those bands regard each other. |
| [`character_reputation.h`](character_reputation.h.md) | Declares the reputation band value and its shared table, implemented in `character_reputation.cpp`. |
| [`character_shell_control.cpp`](character_shell_control.cpp.md) | Tunes the ragdoll a character becomes when it dies: how hard the killing hit throws it, how fast it stiffens, and how much it slides. |
| [`character_shell_control.h`](character_shell_control.h.md) | Declares the death-ragdoll tuning implemented in `character_shell_control.cpp`. |
| [`ContextMenu.cpp`](ContextMenu.cpp.md) | A data-driven list of named commands rendered as text and dispatched into the engine's event bus when one is picked. |
| [`ContextMenu.h`](ContextMenu.h.md) | Declares the configuration-driven command menu implemented in `ContextMenu.cpp`. |
| [`InfoPortion.cpp`](InfoPortion.cpp.md) | Reads one info portion's authored definition out of the XML pool, and serializes the record of having received one. |
| [`InfoPortion.h`](InfoPortion.h.md) | Declares the info portion — the game's unit of authored knowledge — and the shared definition it resolves to. |
| [`relation_registry.cpp`](relation_registry.cpp.md) | Stores and clamps the two goodwill maps, and owns the process-wide registry they live in. |
| [`relation_registry.h`](relation_registry.h.md) | Declares the world's single opinion registry: who likes whom, how much, and what actions change it. |
| [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md) | The consequence table: what killing, attacking or helping someone does to the player's standing with that character's group and faction, and to his own rank and reputation. |
| [`relation_registry_defs.h`](relation_registry_defs.h.md) | The stored shape of one character's opinions: a goodwill number per other character and per faction. |
| [`relation_registry_fights.cpp`](relation_registry_fights.cpp.md) | Keeps a short-lived list of who is currently fighting whom, so that "helping" can be recognized. |
| [`relation_registry_inline.h`](relation_registry_inline.h.md) | The attitude formula: five contributions summed into one signed number, then cut into friend, neutral and enemy by two configured thresholds. |
| [`UIDialogHolder.cpp`](UIDialogHolder.cpp.md) | The screen stack and the input router: it decides which window is modal, which windows draw, what happens to a key the top window does not want, and when the pointer is visible. |
| [`UIDialogHolder.h`](UIDialogHolder.h.md) | Declares the screen stack and input router implemented in `UIDialogHolder.cpp`, and the two small records it keeps its collections in. |

### Heads-up display and in-game screens

| File | Role |
|---|---|
| [`HUDCrosshair.cpp`](HUDCrosshair.cpp.md) | The aiming reticle: four ticks whose distance from the screen centre is the weapon's current dispersion cone projected onto the screen. |
| [`HUDCrosshair.h`](HUDCrosshair.h.md) | Declares the dispersion-driven aiming reticle, implemented in `HUDCrosshair.cpp`. |
| [`HudItem.cpp`](HudItem.cpp.md) | Anything the player holds in their hands: the state machine every held item runs, the animation whose *end* drives the next state, the sound bank keyed by alias … |
| [`HudItem.h`](HudItem.h.md) | Declares the base of every item the player can hold, implemented in `HudItem.cpp`. |
| [`HUDManager.cpp`](HUDManager.cpp.md) | The game's filling of the engine's heads-up-display hook: it owns the in-game screen set, the first-person weapon's two render passes, the look-at target and the damage-direction markers. |
| [`HUDManager.h`](HUDManager.h.md) | Declares the game's heads-up-display object, implemented in `HUDManager.cpp`. |
| [`HUDTarget.cpp`](HUDTarget.cpp.md) | What the player is looking at: a ray cast from the camera each frame that sees through glass and foliage, and the cursor, name plate and reticle colour derived from whatever it hits. |
| [`HUDTarget.h`](HUDTarget.h.md) | Declares the look-at ray and the cursor drawn where it lands, implemented in `HUDTarget.cpp`. |
| [`UIFrameRect.cpp`](UIFrameRect.cpp.md) | A resizable decorated rectangle: nine texture pieces — four corners, four tiled edges, one tiled interior — laid out so a frame of any size is drawn from art of one size. |
| [`UIFrameRect.h`](UIFrameRect.h.md) | Declares the nine-slice decorated rectangle implemented in `UIFrameRect.cpp`. |
| [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md) | The artefact-hunt interface: the team-deathmatch screen plus a reinforcement timer and the offer to pay for an early respawn. |
| [`UIGameAHunt.h`](UIGameAHunt.h.md) | Declares the artefact-hunt interface implemented in `UIGameAHunt.cpp`. |
| [`UIGameCTA.cpp`](UIGameCTA.cpp.md) | The capture-the-artefact interface, and the only place in the game where a player's *live inventory* is turned back into a shopping list — which is what makes their loadout survive a round. |
| [`UIGameCTA.h`](UIGameCTA.h.md) | Declares the capture-the-artefact interface, including the record that maps a buy-menu selection to a slot and item, implemented in `UIGameCTA.cpp`. |
| [`UIGameCustom.cpp`](UIGameCustom.cpp.md) | The in-game interface: the base every game mode's screen set derives from, owning the heads-up display, the inventory and handheld-computer screens, the message log, the script-driven text overlays … |
| [`UIGameCustom.h`](UIGameCustom.h.md) | Declares the in-game interface base, the timed text overlay, and the multiplayer map catalogue — implemented in `UIGameCustom.cpp`. |
| [`UIGameDM.cpp`](UIGameDM.cpp.md) | The deathmatch interface: nine captions the game mode writes strings into, the money and rank readouts, the frag limit, the scoreboard and the vote banner. |
| [`UIGameDM.h`](UIGameDM.h.md) | Declares the deathmatch interface implemented in `UIGameDM.cpp`. |
| [`UIGameMP.cpp`](UIGameMP.cpp.md) | What every multiplayer mode's interface has in common: the server's greeting screen, and the playback controls for a recorded match. |
| [`UIGameMP.h`](UIGameMP.h.md) | Declares the layer every multiplayer mode's interface shares — the server greeting and the recorded-match controls — implemented in `UIGameMP.cpp`. |
| [`UIGameSP.cpp`](UIGameSP.cpp.md) | The single-player game's heads-up layer: which full-screen dialog opens for which key or gameplay event, and the level-change confirmation that freezes the world while the player decides. |
| [`UIGameSP.h`](UIGameSP.h.md) | Declares the single-player game UI and the level-change prompt implemented in `UIGameSP.cpp`. |
| [`UIGameTDM.cpp`](UIGameTDM.cpp.md) | The team-deathmatch heads-up layer: two team score readouts, the team-panel scoreboard, and the hold-to-reveal player-name toggle. |
| [`UIGameTDM.h`](UIGameTDM.h.md) | Declares the team-deathmatch game UI implemented in `UIGameTDM.cpp`. |
| [`UIPanelsClassFactory.cpp`](UIPanelsClassFactory.cpp.md) | Maps a scoreboard panel's authored name to the team it displays. |
| [`UIPanelsClassFactory.h`](UIPanelsClassFactory.h.md) | Declares the scoreboard panel factory implemented in `UIPanelsClassFactory.cpp`. |
| [`UIPlayerItem.cpp`](UIPlayerItem.cpp.md) | One row of the multiplayer scoreboard: a data-driven set of text and icon fields filled from one player's network state each frame. |
| [`UIPlayerItem.h`](UIPlayerItem.h.md) | Declares the scoreboard row implemented in `UIPlayerItem.cpp`. |
| [`UITeamHeader.cpp`](UITeamHeader.cpp.md) | The header strip above one team's scoreboard: authored column labels and authored aggregate fields refreshed from the team panel each frame. |
| [`UITeamHeader.h`](UITeamHeader.h.md) | Declares the team scoreboard header implemented in `UITeamHeader.cpp`. |
| [`UITeamPanels.cpp`](UITeamPanels.cpp.md) | The whole multiplayer scoreboard: every team's panel, plus the rule that decides which panels are visible in which phase of the match. |
| [`UITeamPanels.h`](UITeamPanels.h.md) | Declares the scoreboard container implemented in `UITeamPanels.cpp`. |
| [`UITeamState.cpp`](UITeamState.cpp.md) | One team's half of the multiplayer scoreboard: the rows for its members, spread across several authored columns, kept sorted by contribution and safely mutated while being iterated. |
| [`UITeamState.h`](UITeamState.h.md) | Declares one team's scoreboard panel, implemented in `UITeamState.cpp`. |
| [`UITimeDilator.cpp`](UITimeDilator.cpp.md) | Slows the simulation clock while certain menus are open, if the player asked for it. |
| [`UITimeDilator.h`](UITimeDilator.h.md) | Declares the menu-time-dilation policy implemented in `UITimeDilator.cpp`. |

### Multiplayer: transport, accounts and anti-cheat

| File | Role |
|---|---|
| [`account_manager.cpp`](account_manager.cpp.md) | Creating, deleting and looking up a player's online account: field validation done locally, the rest asked of the matchmaking service and answered by callback. |
| [`account_manager.h`](account_manager.h.md) | Declares the online-account administration surface implemented in `account_manager.cpp`. |
| [`account_manager_console.cpp`](account_manager_console.cpp.md) | The developer console's account commands: sign in and out, create and delete a profile, list and inspect profiles, all routed to the managers the main menu owns. |
| [`account_manager_console.h`](account_manager_console.h.md) | Declares the console commands implemented in `account_manager_console.cpp`. |
| [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md) | The interface an object implements to have its live tuning values dumped for server-side comparison against the shipped configuration. |
| [`cdkey_ban_list.cpp`](cdkey_ban_list.cpp.md) | The multiplayer server's persistent list of banned players, keyed by the hash of a player's product key. |
| [`cdkey_ban_list.h`](cdkey_ban_list.h.md) | Declares the server's persistent ban list implemented in `cdkey_ban_list.cpp`. |
| [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md) | The artefact that is the objective in capture-the-artefact: it refuses to be used by the wrong team and re-anchors itself at its home point when carried back to base. |
| [`cta_game_artefact.h`](cta_game_artefact.h.md) | Declares the capture-the-artefact objective artefact, implemented in `cta_game_artefact.cpp`. |
| [`cta_game_artefact_activation.cpp`](cta_game_artefact_activation.cpp.md) | The activation sequence for a capture-the-artefact objective: the same staged timeline as a normal artefact, with the visual effects and the self-destruction removed. |
| [`cta_game_artefact_activation.h`](cta_game_artefact_activation.h.md) | Declares the capture-the-artefact activation sequence, implemented in `cta_game_artefact_activation.cpp`. |
| [`DemoInfo.cpp`](DemoInfo.cpp.md) | The summary block at the head of a recorded multiplayer demo: which map, which mode, the final score, who recorded it, and a per-player scoreboard … |
| [`DemoInfo.h`](DemoInfo.h.md) | Declares the demo summary record and its per-player line, implemented in `DemoInfo.cpp`. |
| [`DemoInfo_Loader.cpp`](DemoInfo_Loader.cpp.md) | Reads the summary block out of a recorded multiplayer demo file and caches it by filename, so a browse screen can list many demos without re-reading any. |
| [`DemoInfo_Loader.h`](DemoInfo_Loader.h.md) | Declares the caching demo-summary reader implemented in `DemoInfo_Loader.cpp`. |
| [`DemoPLay_Control.cpp`](DemoPLay_Control.cpp.md) | Demo playback that can be told "run until someone kills Ivan, then stop": it subscribes to one kind of recorded game event, optionally fast-forwards until a matching one arrives, and pauses there. |
| [`DemoPlay_Control.h`](DemoPlay_Control.h.md) | Declares the event-seeking demo playback controller implemented in `DemoPLay_Control.cpp`. |
| [`file_transfer.cpp`](file_transfer.cpp.md) | The two ends of in-band file transfer: the server, which runs many transfers keyed by destination and source, and the client, which sends at most one at a time. |
| [`file_transfer.h`](file_transfer.h.md) | Declares the two file-transfer sites — server and client — implemented in `file_transfer.cpp`. |
| [`filereceiver_node.cpp`](filereceiver_node.cpp.md) | One inbound transfer: accumulate arriving chunks into a file or a memory buffer, learn the expected size from the first chunk, and know when it is finished. |
| [`filereceiver_node.h`](filereceiver_node.h.md) | Declares one inbound transfer session, implemented in `filereceiver_node.cpp`. |
| [`filetransfer_common.h`](filetransfer_common.h.md) | The vocabulary of the in-band file transfer: the three commands on the wire, the two progress vocabularies, and the chunk-size bounds that decide how fast a transfer may go. |
| [`filetransfer_node.cpp`](filetransfer_node.cpp.md) | One outbound transfer: four kinds of source behind one reading interface, plus the adaptive chunk size that decides how much goes out per update. |
| [`filetransfer_node.h`](filetransfer_node.h.md) | Declares one outbound transfer session and the four sources it can read from, implemented in `filetransfer_node.cpp`. |
| [`gsc_dsigned_ltx.cpp`](gsc_dsigned_ltx.cpp.md) | Writes and reads a configuration file that carries a signature over its own text, so a client can prove the server's settings were not edited. |
| [`gsc_dsigned_ltx.h`](gsc_dsigned_ltx.h.md) | Declares the writer and reader for a digitally signed configuration file. |
| [`id_generator.h`](id_generator.h.md) | Hands out entity identifiers from a fixed range and takes them back, choosing the identifier that has been free the longest. |
| [`login_manager.cpp`](login_manager.cpp.md) | Signing in to the multiplayer account service: a two-stage online login, an offline stand-in, nickname claiming, and the credentials remembered between runs. |
| [`login_manager.h`](login_manager.h.md) | Declares the multiplayer account session: the logged-in profile, the two asynchronous operations that establish it, and the stored credentials. |
| [`MainMenu.cpp`](MainMenu.cpp.md) | The main menu: a dialogue holder that takes over input and rendering, suspends the running level while it is up, and restores everything it changed on the way out. |
| [`MainMenu.h`](MainMenu.h.md) | Declares the main menu overlay implemented in `MainMenu.cpp`, and the small record the patch downloader reports progress through. |
| [`Message_Filter.cpp`](Message_Filter.cpp.md) | A tap on the message stream: decodes just enough of each message to identify it, hands matching ones to a registered observer, and logs a readable trace of everything it saw. |
| [`Message_Filter.h`](Message_Filter.h.md) | Declares the non-consuming message tap implemented in `Message_Filter.cpp`. |
| [`mp_config_sections.cpp`](mp_config_sections.cpp.md) | Serializes the configuration sections that decide a multiplayer match, so a server can compare a client's tuning against its own. |
| [`mp_config_sections.h`](mp_config_sections.h.md) | Declares the two anti-cheat configuration reporters implemented in `mp_config_sections.cpp`. |
| [`mpactor_dump_impl.cpp`](mpactor_dump_impl.cpp.md) | Declares which of the multiplayer actor's numbers are worth cheating with: sixteen movement and weapon-dispersion values, reported as they stand in memory. |
| [`mt_config.h`](mt_config.h.md) | The ten switches that decide which of the game's per-frame workloads may be handed to a worker thread. |
| [`NET_Queue.h`](NET_Queue.h.md) | The deferred game-event queue: the buffer that holds world-changing messages between arriving and being applied, so they are applied in arrival order at one point in the frame. |
| [`profile_data_types.h`](profile_data_types.h.md) | The frozen vocabulary of a multiplayer player profile: the award set, the best-score set, and the shape of one award record. |
| [`profile_store.cpp`](profile_store.cpp.md) | All that remains of profile loading: confirm somebody is signed in, and report success with nothing attached. |
| [`profile_store.h`](profile_store.h.md) | Declares the player-profile store: the holder of a signed-in player's awards and best scores. |
| [`screenshot_server.cpp`](screenshot_server.cpp.md) | Demands a screenshot or a configuration dump from one client and forwards it to the administrator who asked, starting the forward leg before the download has finished. |
| [`screenshot_server.h`](screenshot_server.h.md) | Declares the server-side anti-cheat relay: an administrator asks a suspected client for its screen or its configuration, and the server pipes it back. |
| [`secure_messaging.cpp`](secure_messaging.cpp.md) | Obfuscates network message payloads with a seed-derived keystream chained against the previous word, and returns a plaintext checksum as the tamper check. |
| [`secure_messaging.h`](secure_messaging.h.md) | Declares the multiplayer message obfuscation: a seed source, a derived variable-length key, and a symmetric transform over a buffer. |
| [`traffic_optimization.cpp`](traffic_optimization.cpp.md) | Loading the two pre-trained compression models that shrink multiplayer update packets. |
| [`traffic_optimization.h`](traffic_optimization.h.md) | Declares the two pre-trained model loaders and the flag set that says which compressor a session uses. |
| [`xrClientsPool.cpp`](xrClientsPool.cpp.md) | Holds a disconnected multiplayer client's state for a while, so that a player who drops and comes straight back gets their score, team and inventory back rather than a fresh start. |
| [`xrClientsPool.h`](xrClientsPool.h.md) | Declares the reconnect pool: parked client records, the identity test that reclaims one, and the expiry sweep. |
| [`xrGameSpy_GameSpyFuncs.cpp`](xrGameSpy_GameSpyFuncs.cpp.md) | Brings the two matchmaking subsystems up and down, and runs the challenge–response handshake that proves a client owns a licensed copy. |
| [`xrGameSpyServer.cpp`](xrGameSpyServer.cpp.md) | The public multiplayer server: what it advertises about itself, how it decides who may enter, and what it does with a client that keeps sending it garbage. |
| [`xrGameSpyServer.h`](xrGameSpyServer.h.md) | Declares the multiplayer server variant that registers with the vendor master service, answers its queries, and challenges every client to prove its copy-protection key. |
| [`xrGameSpyServer_callbacks.cpp`](xrGameSpyServer_callbacks.cpp.md) | An empty translation unit: the matchmaking callbacks it was meant to hold live elsewhere. |
| [`xrGameSpyServer_callbacks.h`](xrGameSpyServer_callbacks.h.md) | Nothing of its own: it forwards to the matchmaking service's key-name definitions. |

### Sound, particles and world effects

| File | Role |
|---|---|
| [`ai_sounds.cpp`](ai_sounds.cpp.md) | The display-name table for sound *kinds*: the mapping from the AI-perception sound classification to the text a designer or a debug view sees. |
| [`AnselManager.cpp`](AnselManager.cpp.md) | Photo mode: hand the vehicle-maker's screenshot tool control of the camera's orientation while the world stands still, and take it back cleanly. |
| [`AnselManager.h`](AnselManager.h.md) | Declares photo mode — a camera, an infinite-lifetime camera effector and the manager that brackets a capture session — implemented in `AnselManager.cpp`. |
| [`DelayedActionFuse.cpp`](DelayedActionFuse.cpp.md) | A fuse that arms when its host's condition falls to a threshold, then burns the remaining condition away over a fixed time and fires when either runs out. |
| [`DelayedActionFuse.h`](DelayedActionFuse.h.md) | Declares the condition-coupled fuse implemented in `DelayedActionFuse.cpp`. |
| [`material_manager.cpp`](material_manager.cpp.md) | Footsteps: pairs the material a creature is made of with the material it is standing on, and plays a step sound at the right interval from the right place. |
| [`material_manager.h`](material_manager.h.md) | Declares the per-creature footstep system: which material it is made of, which it is standing on, and the sounds that pairing produces. |
| [`material_manager_inline.h`](material_manager_inline.h.md) | The two material indices and the pairing they select. |
| [`particle_params.h`](particle_params.h.md) | A three-vector bundle — offset, orientation, velocity — that scripts construct to place a particle effect. |
| [`ParticlesObject.cpp`](ParticlesObject.cpp.md) | One playing particle effect as a world entity: a renderer-owned emitter wrapped in a scheduled, spatially indexed object that can end itself. |
| [`ParticlesObject.h`](ParticlesObject.h.md) | Declares the world-entity wrapper around one playing particle effect, implemented in `ParticlesObject.cpp`. |
| [`ParticlesPlayer.cpp`](ParticlesPlayer.cpp.md) | Plays particle effects on an animated object's bones: effects are attached to authored skeleton points, follow the pose every frame, age out, and die with their carrier. |
| [`ParticlesPlayer.h`](ParticlesPlayer.h.md) | Declares the mix-in that lets any skinned object play particle effects on its bones, implemented in `ParticlesPlayer.cpp`. |
| [`sound_collection_storage.cpp`](sound_collection_storage.cpp.md) | Interns sound collections by the value of their parameters, so that a hundred creatures with the same voice hold one copy of its samples. |
| [`sound_collection_storage.h`](sound_collection_storage.h.md) | Declares the process-wide table that makes identical sound sets load once and be shared by every creature that uses them. |
| [`sound_collection_storage_inline.h`](sound_collection_storage_inline.h.md) | Reaches the single sound-collection storage, creating it on first use. |
| [`sound_player.cpp`](sound_player.cpp.md) | The per-creature sound scheduler: it owns which sound kinds a creature knows, arbitrates between kinds that must not overlap, delays and staggers playback randomly … |
| [`sound_player.h`](sound_player.h.md) | Declares the per-object sound player: a creature's catalogue of sound kinds, the priority and mutual-exclusion rules between them, and the queue of sounds currently scheduled or playing on its bones. |
| [`sound_player_inline.h`](sound_player_inline.h.md) | The sound player's queries, plus the two mask operations that are the only way whole categories of a creature's sounds are silenced. |
| [`wallmark_manager.cpp`](wallmark_manager.cpp.md) | Sprays a burst of decals onto every static surface near a point — the blood and scorch an explosion leaves on its surroundings. |
| [`wallmark_manager.h`](wallmark_manager.h.md) | Declares the burst-decal placer: a point, a pool of decal appearances, and the sweep that marks everything around it. |

### Shared utilities and debugging

| File | Role |
|---|---|
| [`ai_debug.h`](ai_debug.h.md) | The AI layer's debug switch set: one bit per diagnostic, in one flag word the whole game module reads. |
| [`ai_debug_variables.cpp`](ai_debug_variables.cpp.md) | A global scratch table of named numbers, so a probe deep in the AI can publish a value that a console command or another subsystem can read back without a plumbing change. |
| [`ai_debug_variables.h`](ai_debug_variables.h.md) | Declares the AI's named-value debug scratch table. |
| [`ai_space.cpp`](ai_space.cpp.md) | The AI layer's single global: it owns the navigation graphs, the cover database, the dynamic-obstacle registry, the door manager, the evaluation-function store and the script virtual machine … |
| [`ai_space.h`](ai_space.h.md) | Declares the AI layer's global service holder and the short name every caller reaches it by. |
| [`ai_space_inline.h`](ai_space_inline.h.md) | The AI space's service accessors, each asserting the service exists, plus the one-word alias the rest of the codebase calls it by. |
| [`BlockAllocator.h`](BlockAllocator.h.md) | A bump allocator over reusable fixed-size blocks: appends objects cheaply, and resets to empty without freeing anything. |
| [`CycleConstStorage.h`](CycleConstStorage.h.md) | A fixed-capacity ring of recent values, indexed from the oldest, that never allocates and never reports a size. |
| [`date_time.cpp`](date_time.cpp.md) | The game world's calendar: a proleptic Gregorian date packed into a single 64-bit millisecond count, and the exact inverse that unpacks it. |
| [`date_time.h`](date_time.h.md) | Declares the two conversions between a calendar date and the game's single scalar clock, implemented in `date_time.cpp`. |
| [`dbg_draw_frustum.cpp`](dbg_draw_frustum.cpp.md) | Builds a view frustum from camera parameters, and draws one as wireframe for debugging. |
| [`debug_renderer.cpp`](debug_renderer.cpp.md) | Turns the three volume shapes the game layer wants to visualize — oriented box, axis-aligned box, ellipsoid — into indexed line lists for the renderer's debug channel. |
| [`debug_renderer.h`](debug_renderer.h.md) | Declares the game layer's wireframe scratchpad, implemented in `debug_renderer.cpp` and `debug_renderer_inline.h`. |
| [`debug_renderer_inline.h`](debug_renderer_inline.h.md) | The two cheapest debug shapes and the per-frame flush. |
| [`debug_text_tree.cpp`](debug_text_tree.cpp.md) | Measures a debug text tree into aligned columns, and provides the two sinks that render it — to the screen in alternating colours, or to the log. |
| [`debug_text_tree.h`](debug_text_tree.h.md) | Declares the on-screen debug text tree implemented in `debug_text_tree.cpp` and `debug_text_tree_inline.h`, plus the value-to-text conversions its lines are built from. |
| [`debug_text_tree_inline.h`](debug_text_tree_inline.h.md) | Building a debug text tree line by line, and the emission pass that pads each line into the measured columns. |
| [`ini_id_loader.h`](ini_id_loader.h.md) | Turns a configuration line of names into a dense index space, so that the names can be used as array subscripts everywhere else. |
| [`ini_table_loader.h`](ini_table_loader.h.md) | Loads a configuration section into a two-dimensional table whose rows are addressed by a name registry rather than by position. |
| [`mixed_delegate.h`](mixed_delegate.h.md) | One callback slot that a script or the engine may fill interchangeably — the mechanism by which an asynchronous operation reports back to whichever side asked for it. |
| [`mixed_delegate_unique_tags.h`](mixed_delegate_unique_tags.h.md) | The six names that keep otherwise identical callback slots distinct, one per account operation. |
| [`ObjectDump.cpp`](ObjectDump.cpp.md) | Formats an object's identity, registration flags, recent position history and visual bounds into readable text, for a developer staring at a bug. |
| [`ObjectDump.h`](ObjectDump.h.md) | Declares the object-state text dumpers implemented in `ObjectDump.cpp`. |
| [`queued_async_method.h`](queued_async_method.h.md) | Serialises calls to one asynchronous method: while a request is in flight a second request is held, and only the newest held request runs next. |
| [`Random.cpp`](Random.cpp.md) | Defines the game module's one shared pseudo-random generator instance. |
| [`Random.hpp`](Random.hpp.md) | Names the game module's one shared pseudo-random generator. |
| [`RegistryFuncs.cpp`](RegistryFuncs.cpp.md) | Reads and writes a handful of values in the host's machine-wide settings store, under the key the retail installer created — the only place the engine keeps state outside its own files. |
| [`RegistryFuncs.h`](RegistryFuncs.h.md) | Declares the six accessors to the host's machine-wide settings store, implemented in `RegistryFuncs.cpp`. |
| [`safe_map_iterator.h`](safe_map_iterator.h.md) | Declares a keyed registry that is walked round-robin across frames under a time budget, and stays correct when entries are added or removed mid-walk. |
| [`safe_map_iterator_inline.h`](safe_map_iterator_inline.h.md) | The round-robin walk: advance the cursor past an entry *before* updating it, so that an entry which removes itself mid-update cannot invalidate the walk. |
| [`static_cast_checked.hpp`](static_cast_checked.hpp.md) | A downcast that asserts, in diagnostic builds, that the cheap conversion and the safe one agree — and compiles away to the cheap one otherwise. |
| [`static_cast_checked_test.cpp`](static_cast_checked_test.cpp.md) | Not built: a scratch file recording, by example, what the checked downcast must accept and what it must reject. |
| [`xr_time.cpp`](xr_time.cpp.md) | The game clock as a value: a single millisecond count that the scripts add to, subtract from, and format as a calendar date. |
| [`xr_time.h`](xr_time.h.md) | Declares the script-visible game-time value: comparison, arithmetic, calendar split and formatting over one millisecond count. |
