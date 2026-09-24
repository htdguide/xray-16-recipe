# src/xrGame/GameObject.cpp

> The base class every entity in the game is: the client object's lifecycle from spawn record to destruction, its place in the spatial and scheduling registries, its navigation-graph position, its script binding and its script callback table.

**Needs** — [`GameObject.h`](GameObject.h.md) · [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md) · [`script_binder.h`](script_binder.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`Hit.h`](Hit.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai_obstacle.h`](ai_obstacle.h.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`magic_box3.h`](magic_box3.h.md) · [`doors.h`](doors.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`GameObject.h`](GameObject.h.md); callers name that, not this file.
**Tier floor** — T2: registry bookkeeping, transform arithmetic and lifecycle ordering; the only T1 pressure is one lock-free per-frame flag and the fact that it is on every entity's hot path

## Purpose

Every **client object** in the game — actor, weapon, creature, anomaly, lamp, door —
derives from this class. It is the meeting point of five separate roles the engine
imposes on an entity, and the value of the file is the *ordering* it establishes between
them, not the individual steps:

- it is **spatial**: registered in the space partition so visibility, sound and the
  senses can find it by region;
- it is **scheduled**: registered with the update scheduler, which calls it at a rate
  that degrades with distance;
- it is **renderable** and **collidable**;
- it has a **navigation position**: a level-graph vertex and a game-graph vertex, kept
  current as it moves, because almost all AI reasoning is in graph coordinates and not in
  world coordinates;
- it is **script-visible**: it owns a lazily-created **game object** facade and a table of
  named callbacks scripts can subscribe to.

The class is also the boundary between the **server object** (the authoritative spawn
record) and the live instance: `net_Spawn` is where a record becomes a running object,
and it fixes the order in which those five roles are entered. A rebuild that reorders
those steps will break in ways that are hard to diagnose — a script binder that runs
before the navigation position is assigned sees an entity at graph vertex "invalid", and
a scheduler registration before the spatial registration can deliver an update to an
object the space partition has never heard of.

## State

```text
RECORD GameObjectIdentity
  net_id        : int (16-bit)   # the entity identifier: names it in saves, network and every registry
  name          : text           # instance name, from the spawn record; may differ from the section
  section       : text           # the configuration section supplying every tuned number
  visual_name   : text           # model path, extension stripped, lowercased
  class_id      : int            # the class identifier from the spawn record
  script_clsid  : int            # the same class as an index into the script-visible class enumeration

RECORD GameObjectFlags            # one packed word; each is consulted per frame
  visible          : bool        # also gates registration as renderable in the space partition
  enabled          : bool        # also gates registration as collidable
  destroy          : bool        # queued for destruction at the end of the frame
  active_counter   : int (8-bit) # nesting count of per-frame-update requests; see processing_activate
  net_local        : bool        # this host owns the object and may move it
  net_ready        : bool        # fully spawned; the scheduler may touch it
  net_sv_update    : bool
  crow             : bool        # scheduled for a per-frame update this frame
  pre_destroy      : bool

RECORD GameObjectLinks
  parent           : optional<object>   # the container: an item in an inventory, a scope on a rifle
  ai_location      : navigation position (level vertex + game vertex)
  ai_obstacle      : the object's footprint as an obstacle in the navigation mesh
  spawn_ini        : optional<config>   # per-instance overrides carried in the spawn record itself
  story_id         : int                # a stable authored identity scripts look entities up by
  lua_game_object  : optional<facade>   # created on first script access, destroyed with the object
  callbacks        : map<callback_kind, script callback list>
  visual_callbacks : list<pose callback>
  anim_mov_ctrl    : optional<animation-driven movement controller>

RECORD GameObjectSavedPosition          # the position history, at most four entries
  time      : int
  position  : vector
```

Invariants the code asserts continuously:

- an object is **spawned exactly once** and **destroyed exactly once**; both hooks check
  a flag and the destructor checks that every optional above has been released.
- an object with a parent is **not** in the space partition. Attaching unregisters it;
  detaching registers it.
- the per-frame update is called **at most once per frame** per object; a second call in
  the same frame is a fatal error in debug builds because it silently doubles every
  velocity integration in the engine.
- an object registered as collidable **has** a collision model.
- the active-update counter is a nesting count, not a boolean: it must never underflow or
  overflow, because two subsystems may each independently need the object updated.

## `net_Spawn`

**Contract** — turns a server object record into a live client object. Returns whether the
spawn succeeded; a failure leaves the object unusable and the caller drops it. The single
most order-sensitive function in the game layer.

**Invariants** — the entity identifier must be free: spawning onto an identifier that
already names a live object is refused (this is the "no entity is registered twice"
runtime invariant). After a successful return, the object is in the object registry, in
the space partition, in the scheduler, has a valid navigation position, has run its script
binder's own spawn, and is marked ready.

```text
FUNCTION spawn_from_record(record) -> bool
  REQUIRE not already spawned
  mark spawned; remember the frame number as the spawn time
  create the obstacle footprint

  # 1. identity and appearance, before anything can ask for either
  IF record carries a visual THEN
    set visual from record          # replaces the model; see set_visual
    IF record flags it an obstacle THEN mark spatial type as obstacle
  name = record.name; section = record.name
  IF record carries a display-name override THEN name = that override

  # 2. claim the identifier — refuse a duplicate
  IF an object already exists with record.id THEN RETURN false
  set id = record.id

  # 3. transform straight from the record, before any registry sees the position
  transform = rotation from record.angles, translation from record.position
  REQUIRE the transform is finite and non-degenerate

  # 4. per-instance configuration embedded in the record, parsed as a config file
  IF record carries an override config text THEN spawn_ini = parse(that text)
  story_id = record.story_id

  # 5. ownership, then registration in the object registry
  net_local = record says this host owns it
  net_ready = true
  register in the level's object registry

  # 6. server flags decide whether the AI can see it and whether it uses graph positions
  server_flags = record.flags
  set or clear "visible to AI" in the spatial type from those flags

  # 7. class resolution and script binder — reload then reinit, in that order,
  #    because reload picks the class and reinit clears per-life state
  reload(section);  script_binder.reload(section)
  reinit();         script_binder.reinit()

  # 8. saved per-object state, if this spawn came from a save game
  IF record carries client data THEN load_from(client data)

  # 9. navigation position
  IF a level graph is loaded THEN
    IF the record has no parent THEN
      take the level vertex and game vertex from the record when they are valid
      re-derive them from the position when they are not
      IF this object uses graph positions AND the position lies inside its vertex THEN
        snap the position's height onto the navigation mesh's plane at that vertex
    ELSE
      take the vertices from the record without re-deriving — a contained object
      inherits its container's place

  # 10. fall back to the section for a model, and build a collision model if configured
  clear the position history
  IF no visual AND the section names one THEN set visual from the section
  IF no collision model AND the section requests one THEN
    REQUIRE a visual exists                 # the collision model is built from the skeleton
    collision model = skeleton collider over the visual

  # 11. enter the registries — spatial first, then the scheduler
  spatial_register()
  IF this class wants scheduling THEN schedule_register()
  request per-frame updates
  destroy flag = false
  mark me for a per-frame update this frame

  # 12. authored contents, then the script side's own spawn
  spawn_supplies()
  RETURN script_binder.spawn(record)
```

**Notes**

- Step 9's height snap is why objects authored slightly above or below the floor sit
  correctly on it at load: the navigation mesh, not the collision geometry, is the
  authority on floor height for anything that uses graph positions.
- Steps 7 and 12 bracket the whole engine-side spawn around the script side, so a script
  binder is guaranteed a fully-formed object and the engine never half-spawns one.
- The refusal on a duplicate identifier is a *recovery*, not an assertion: hand-authored
  and mod-authored spawn files do contain duplicates, and the choice is to skip the
  second rather than abort the level load.

## `net_Destroy`

**Contract** — tears the live object down. Must leave no subsystem holding a reference:
this is the conformance invariant "a destroyed entity is unreferenced by the scheduler,
the render graph and the physics world before its memory is released". Asserts that the
object was spawned and that the destroy flag is already set — destruction is a two-phase
affair, flag first, teardown second, because the flag is raised from inside a frame and
the teardown runs at a safe point.

```text
FUNCTION destroy()
  REQUIRE spawned AND destroy flag is set
  destroy the animation movement controller if one is active
  release the per-instance config
  detach the pose callback from the skeleton     # or the skeleton calls into freed memory
  release the collision model
  IF scheduled THEN schedule_unregister()
  spatial_unregister()
  clear the visual (releases the model through the renderer)
  net_ready = false
  unregister from the level's object registry
  IF I am the level's current entity THEN
    give up control and clear the current entity      # but do not switch to another
  remove myself from the physics correction/prediction list
  script_binder.destroy()
  class index = invalid
  release the script facade
  mark not spawned
```

**Notes** — the pose callback detach appears twice over the object's life (here and in
`remove_visual_callback`) because the skeleton holds a raw back-pointer to the object; in
a rebuild this is a subscription whose lifetime is tied to the object's.

## `OnEvent`

**Contract** — handles the two network events every object understands: a hit and a
destroy order. Other events are ignored here and handled by subclasses.

```text
FUNCTION on_event(packet, kind)
  CASE hit, hit-with-statistics:
    hit = decode(packet)
    hit.attacker = lookup(hit.attacker_id)      # may be absent on a client; logged, not fatal
    hit.weapon   = lookup(hit.weapon_id)
    IF kind is hit-with-statistics AND this is multiplayer THEN
      hand the hit to the weapon-usage statistics for its server-side check
    record_hit_info(attacker, weapon, bone, position_in_bone_space, direction)
    apply_hit(hit)
    IF multiplayer THEN close the statistics check

  CASE destroy:
    IF I have a parent THEN
      # the container owns my destruction; a direct order here would leave a dangling child
      log and ignore
    ELSE
      raise the destroy flag
```

**Invariants** — the hit-info record is set *before* the hit is applied, because the hit
handler may kill the object and the death handler reads who did it.

## `spawn_supplies`

**Contract** — creates the entities an authored "spawn" block in the object's per-instance
config asks for, as children of this object. Runs once, during spawn. Skipped entirely
when the alife simulation is running, because then the *simulation* owns entity creation
and duplicating it here would double every container's contents on every level load.

```text
FUNCTION spawn_supplies()
  IF no per-instance config OR the alife simulation is active THEN RETURN
  IF the config has no "spawn" section THEN RETURN

  FOR EACH (section, arguments) line IN the "spawn" section
    IF the named section does not exist THEN CONTINUE   # tolerate broken mod data
    count = first argument, default 1
    probability = "prob=" argument, default 1 (and 1 if it parses to zero)
    condition = "cond=" argument, default 1
    attach_scope    = the word "scope" appears
    attach_silencer = the word "silencer" appears
    attach_launcher = the word "launcher" appears

    REPEAT count TIMES
      IF random() >= probability THEN CONTINUE
      child = create a server object record for section, at my position and
              navigation vertex, parented to me
      IF child is an inventory item THEN child.condition = condition
      IF child is a weapon THEN
        FOR EACH of scope, silencer, launcher
          IF that addon is attachable on this weapon THEN set its flag from the request
      send the spawn record onto the wire as a reliable message
      release the record          # the record is consumed by the spawn path, not kept
```

**Notes**

- The arguments are parsed out of a free-form value string by substring search, which is
  why the words are position-independent and why `prob=` and `cond=` look like an
  afterthought. They are: the format grew. A rebuild should parse it as a key/value list
  and accept the shipped spelling.
- The probability roll is per copy, not per line, so `count=3 prob=0.5` yields zero to
  three items rather than three or none.
- Supplies are created by *sending a spawn message*, even in single player, because the
  server side is what owns entity creation and the single-player process runs both sides.

## `H_SetParent`, `OnH_B_Chield`, `OnH_A_Chield`, `OnH_B_Independent`, `OnH_A_Independent`

**Contract** — attach to or detach from a container, returning the previous parent. The
four hooks are the before/after pairs for each direction, so that a subclass can act on
either side of the registry change.

**Invariants** — you may not reparent directly from one container to another: detach to
no parent first. The code asserts this, because the before/after hooks assume exactly one
transition is happening.

```text
FUNCTION set_parent(new_parent, just_before_destroy) -> previous parent
  IF new_parent is the current parent THEN RETURN it
  REQUIRE new_parent is none OR current parent is none

  IF attaching THEN before_attach()  ELSE before_detach(just_before_destroy)
  IF attaching THEN spatial_unregister()  ELSE spatial_register()
  parent = new_parent
  IF attaching THEN after_attach()  ELSE after_detach()
  mark me for a per-frame update this frame
  RETURN previous parent
```

The base behaviours are minimal and exactly express "contained things are invisible and
have no independent place in the world": attaching hides the object; detaching shows it;
detaching first copies the container's navigation position onto the object so it does not
appear at the graph vertex it last occupied hours ago.

## `processing_activate` / `processing_deactivate`

**Contract** — request or release a per-frame update. A **nesting count**, not a flag: the
object is added to the level's active set when the count goes from zero to one and removed
when it returns to zero. Unbalanced calls are a fatal error in debug builds, in both
directions.

**Notes** — the counter exists because several independent subsystems may each need an
object updated (it is on fire, it is being carried, a script asked), and none of them can
know whether another still needs it.

## `MakeMeCrow`

**Contract** — marks the object for a per-frame update this frame. Idempotent within a
frame and safe to call from several threads at once; the first caller in a frame wins and
the rest return without touching the level's list.

**Invariants** — the level's per-frame list is appended to without a lock, so exactly one
thread may append per object per frame. That is what the atomic frame stamp buys.

```text
FUNCTION mark_for_frame_update()
  IF already marked THEN RETURN
  IF per-frame updates are not requested THEN RETURN
  ATOMICALLY
    IF my frame stamp == current frame THEN RETURN      # somebody already did it
    claim the current frame by compare-and-swap
    IF the claim failed THEN RETURN
  append me to the level's per-frame update list
  set the marked flag
```

**Notes** — "crow" is the codebase's own word for "needs a per-frame update this frame",
distinct from "registered with the scheduler". The scheduler runs an object at a degraded
rate; this list runs it exactly once this frame. A rebuild is free to rename it but will
need both concepts.

## `UpdateCL`

**Contract** — the per-frame update. Refreshes the object's place in the space partition,
decides whether it deserves another per-frame update next frame, and notifies the
navigation-obstacle layer when the object has actually moved.

```text
FUNCTION update_per_frame()
  REQUIRE the transform is finite
  REQUIRE this is the first call this frame
  REQUIRE I am not both parented and spatially registered
  REQUIRE I am not collidable without a collision model

  spatial_update(coarse position epsilon, coarse radius epsilon)

  # decide whether to keep getting per-frame updates
  IF I am carried by the view entity, or I always qualify THEN
    mark_for_frame_update()
  ELSE
    distance = camera to me
    IF distance < near radius THEN mark_for_frame_update()
    ELSE IF I was visible within the last two frames AND distance < far radius THEN
      mark_for_frame_update()

  IF I have a parent THEN RETURN                   # a contained object has no world motion
  IF my transform is unchanged since last frame THEN RETURN
  on_matrix_change(previous)                       # tells the obstacle layer to re-stamp the mesh
  remember the transform
```

**Notes** — the two-radius rule is the load-bearing decision: near the camera everything
updates, further out only things the visibility pass actually drew in the last two frames.
This is what lets a level hold hundreds of entities and update a few dozen. The tolerance
used here is five times coarser than the one the scheduler uses, so a slowly drifting
object re-registers in the space partition on its scheduled tick rather than every frame.

## `spatial_update`, `spatial_move`, `spatial_register`, `spatial_unregister`

**Contract** — keep the object's bounding sphere in the space partition current, and
maintain a short history of where it has been.

```text
FUNCTION spatial_update(position_epsilon, radius_epsilon)
  IF the position history is empty OR the newest entry differs from my position
     by more than position_epsilon THEN
    push a new (time, position) entry, keeping at most four — oldest dropped
    spatial_move()
    RETURN
  refresh the newest entry's timestamp        # I am effectively stationary
  IF I am registered THEN
    IF my radius changed beyond radius_epsilon THEN spatial_move()
    ELSE IF my bounding centre moved beyond position_epsilon THEN spatial_move()
```

`spatial_move` is where the navigation position is refreshed: a parented object copies its
container's, an independent one re-derives its own from its bounding centre. The bounding
centre is used rather than the origin, but with the origin's horizontal coordinates —
so the vertex lookup is taken at the object's feet's ground plane at the height of its
middle, which is what keeps a tall object on the vertex it is standing on rather than one
it leans over.

**Notes** — the four-entry history exists for client-side interpolation and for hit
rewind in multiplayer; the timestamps are converted to server time when read out.

## `set_visual` (`cNameVisual_set`)

**Contract** — attaches, replaces or clears the object's model. Replacing carries the
skeleton's pose-update callback across from the old model to the new one, so that a model
swap does not silently stop the game layer's per-pose hooks. Clearing releases the model
through the renderer. Always ends by notifying the object that its visual changed.

**Invariants** — setting the same name twice is a no-op; this matters because the spawn
path sets it from the record and then again from the section.

## `Load`

**Contract** — reads the class's configuration section: the object's name and section are
both set to the section name, the model is taken from the section (path lowercased,
extension stripped, because the model store is keyed by a normalized path), and the object
starts invisible. It also clears "reacts to sound" in the spatial type, so that only
objects that explicitly opt in are considered by the sound sense.

## `reinit` / `reload`

**Contract** — the two halves of per-life reset. `reload` resolves the class identifier to
its script-visible class index; `reinit` clears the per-life state: the pose callbacks, the
navigation position and every subscribed script callback. Called in that order on spawn
and again whenever an object is recycled.

**Notes** — clearing script callbacks on reinit is what prevents a recycled entity from
firing a previous owner's script; it is the subtle half of object reuse.

## `net_Save` / `net_Load` / `save` / `load` / `net_SaveRelevant`

**Contract** — serialize the object into a save game. The engine-side `save`/`load` pair is
empty at this level — subclasses fill it — but the *framing* is here and is load-bearing:
the whole record is wrapped in a length-prefixed chunk, with the script binder's own blob
written inside it, after the engine's. A reader that does not know a subclass can therefore
still skip the record. Whether an object is saved at all is delegated to the script binder.

## `lua_game_object`

**Contract** — returns the script-visible facade, creating it on first use. The facade
lives as long as the object and is destroyed with it. Asking for it after destruction is a
programming error and is reported.

**Notes** — the lazy creation is why an entity that no script ever touches costs nothing on
the script side. The facade holds a back-pointer to the object; that is the reason
destruction must release it explicitly rather than letting the script garbage collector
decide.

## `callback` / `clear_callbacks`

**Contract** — the script callback table: one named callback list per callback kind, looked
up (and created empty on first request) by kind. Scripts subscribe; the engine fires.
`clear_callbacks` empties every list. The set of kinds and the order in which they fire is
frozen by conformance criterion 10.

## `use`

**Contract** — the "player pressed use on me" entry point. Refuses when the object is a door
that is currently blocked in either direction; otherwise fires the use callback with this
object and the user, and reports success. Returning false is how the engine tells the
player's use action that nothing happened.

## `DestroyObject` / `NeedToDestroyObject` / `setDestroy`

**Contract** — the destruction request path, kept separate from the teardown. `setDestroy`
raises the flag and enqueues the object on the level's end-of-frame destruction list;
asking for destruction while already queued is ignored. `DestroyObject` is the *authoritative*
request: only the owning host issues it, and it does so by sending a destroy event, so that
every host agrees. `NeedToDestroyObject` is the per-tick question a subclass overrides to
retire itself (a timed-out grenade, a fully-looted corpse); the base answer is never.

**Invariants** — destruction is requested at most once per object; the removed flag latches.

## `add_visual_callback` / `remove_visual_callback`

**Contract** — subscribe to the skeleton's per-pose callback. The object registers itself
with the skeleton only while at least one subscriber exists, and unregisters when the last
one leaves, so that an object with no interest in bone poses costs nothing per frame.
Double-subscribing the same callback is a programming error.

## `create_anim_mov_ctrl` / `destroy_anim_mov_ctrl` / `update_animation_movement_controller`

**Contract** — animation-driven movement: while active, the object's world transform is
driven by the root motion baked into a playing animation rather than by the game's own
movement code. Creating one while another is active hands the new animation to the existing
controller so the motion is continuous across a blend change. The controller requires an
explicit starting pose — without one there is no frame to interpret the root motion
relative to, and the object teleports. Changing the model destroys the controller, because
the root bone it was tracking no longer exists.

The per-frame tick advances an active controller and destroys one that has finished.

## `spatial` / `render` / `collision` accessors

**Contract** — the object's bounding sphere centre, radius and box all come from the model's
own precomputed visibility data transformed by the object's matrix; an object with no model
cannot answer them. Rendering submits the model with the object's transform and stamps the
model with the current frame, which is the stamp the per-frame-update decision above reads
back.

## `setEnabled` / `setVisible`

**Contract** — the two flags each do double duty: they gate behaviour *and* they add or
remove the corresponding capability bit from the object's spatial registration, so that a
hidden object is not considered by the visibility pass and a disabled one is not considered
by collision queries. An object can only become renderable if it actually has a model, and
only collidable if it actually has a collision model.

## Navigation-position helpers

**Contract** — `UsedAI_Locations` answers whether this object participates in graph
positions at all (a server flag on the spawn record decides). `validate_ai_locations`
re-derives the level vertex from the current position and then the game vertex from the
level-to-game cross table, skipping the work when the object has not changed vertex.
`setup_parent_ai_locations` copies a container's position and graph vertices onto a
contained object, falling back to re-derivation when the container itself has none.

**Invariants** — the game vertex is always derived from the level vertex through the
cross table rather than stored independently, so the two can never disagree.

## `ef_*` type queries

**Contract** — six queries the AI's evaluation functions ask of any object: creature type,
equipment type, main weapon type, anomaly type, weapon type, detector type. The base
implementations all fail loudly: reaching one means the AI asked a question of an object
whose class never declared an answer, and silently returning a default would produce
mysteriously wrong target selection rather than a crash. The class identifier is printed
so the missing override can be found.

## `u_EventGen` / `u_EventSend`

**Contract** — build and send a game event message: a header carrying the current server
time, the event kind and the destination entity identifier, then the caller's payload,
sent reliably by default. This is the sole path by which an object asks the authoritative
side to do something to an entity.

## `OnRender` / `dbg_DrawSkeleton` (debug builds)

**Contract** — debug visualization. The skeleton draw renders each bone's collision shape
in its own colour by shape kind. The object draw renders both the tight oriented box over
the visible bones and a second box grown by half a navigation cell on every axis — the
second is what the obstacle layer actually stamps into the navigation mesh, so seeing both
is how an obstacle that is too large or too small is diagnosed. The minimum-volume box over
the bone corners is computed rather than the axis-aligned union, because a diagonally-posed
creature's axis-aligned box would block a corridor it is standing beside.

## Could not recover

- `get_new_local_point_on_mesh` returns a random direction scaled by 0.7 with no bone
  assigned; the constant is unexplained and the base implementation appears to be a
  placeholder that subclasses with real skeletons override.
- The near and far per-frame-update radii are engine-wide constants defined elsewhere;
  their values were tuned by eye.
