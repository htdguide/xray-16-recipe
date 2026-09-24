# src/xrGame/Missile.cpp

> The thrown item: a held weapon whose "shot" is a second entity it spawns, charges up, and then releases into the world with a velocity.

**Needs** — [`Missile.h`](Missile.h.md) · [`hud_item_object.h`](hud_item_object.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level.h`](Level.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`xrUICore/ProgressBar/UIProgressShape.h`](../xrUICore/ProgressBar/UIProgressShape.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Missile.h`](Missile.h.md)
**Tier floor** — T2: a state machine over animation events, with one bone-transform computation per frame

## Purpose

A grenade is not a bullet. It is an object that exists in the world before, during and after
the throw, it has a fuse that keeps running while it is held, and the player can hold the
throw at a charged-up force and change their mind. None of that fits the weapon model, so
thrown items get their own base class.

The central decision — and the one that explains every peculiarity below — is that **the
thing that flies is not the thing that was held.** During the wind-up the held item spawns a
duplicate of itself, parented to itself, called the fake missile. At the release instant the
held item hands the fake missile a launch frame and a force and then *rejects ownership of
it*, which makes it an independent object, which activates its physics with that velocity.
The held item never leaves the hand; the inventory removes it separately.

## State

```text
RECORD Missile                          # on top of the held-item base
  state_time         : int              # ms in the current state; reset on every switch
  throw              : bool             # the release has been requested
  motion_marks_available : bool         # the throw animation carries a release marker
  destroy_time       : int              # absolute time this object removes itself
  destroy_time_max   : int              # the fuse length, from configuration
  throw_direction    : vec3             # the launch direction, in world space
  throw_matrix       : frame            # the full launch frame: orientation and origin
  fake_missile       : Missile or none  # the duplicate that will actually fly

  force_min / force_const / force_max / force_grow_speed : real
  const_power        : bool             # true = fixed-force throw, false = charged throw
  throw_force        : real             # the charging force

  throw_point        : vec3             # launch origin offset, in the owner's frame
  throw_dir          : vec3             # launch direction, in the owner's frame
  ef_weapon_type     : int              # the planner's coarse weapon classification

ENUM MissileState                       # extends the held-item states
  ThrowStart · Ready · Throw · ThrowEnd
```

**Invariants**

- `fake_missile` is non-empty from the end of the wind-up animation until the release
  completes. `Throw` writes through it unconditionally, so calling `Throw` before the wind-up
  animation has ended is a crash, not a misbehaviour.
- `throw_force` is clamped to `[force_min, force_max]` while charging and reset to `force_min`
  after each release, so a second throw starts from the bottom of the range.
- The fake missile is flagged **not savable**. It exists for a fraction of a second between
  spawn and launch, and a save taken in that window must not resurrect it.

## The throw sequence

This is the file. Everything else serves it.

```text
Idle
  │ fire pressed (constant force) ─────────────────► ThrowStart, throw = true
  │ alternate pressed (charged)   ─────────────────► ThrowStart, throw = false
  ▼
ThrowStart            wind-up animation; force starts at force_min
  │ animation ends: SPAWN THE FAKE MISSILE (parented to this item)
  │   throw requested? ──yes──► Throw
  │                    ──no───► Ready
  ▼
Ready                 the charged hold; force grows every frame toward force_max
  │ alternate released, or throw otherwise requested ──► Throw
  ▼
Throw                 the release animation
  │ the animation's release MARKER fires ──► Throw()   ← the actual launch
  │ (no marker in this animation? launch when the animation ENDS instead)
  ▼
ThrowEnd  ──► Showing ──► Idle          the hand comes back up with the next one
```

**Invariants** — the launch is driven by an *animation marker*, not by a timer or by the state
change. The grenade leaves the hand on the frame the hand opens, and retuning the animation
retunes the throw with no code change. The fallback to the animation's end exists for
animations that were authored without a marker, and it is decided once when the throw
animation starts by inspecting whether that clip carries any markers.

## `Throw`

**Contract** — the launch. Computes the launch frame from the owner, copies it and the force
onto the fake missile, resets the charging force, and sends an ownership-reject event that
detaches the fake missile from this item. Requires an owner and a fake missile. Only the
authoritative side sends the event.

```text
FUNCTION Throw()
  REQUIRE the parent is an entity
  setup_throw_params()                        # fills throw_matrix and throw_direction
  fake_missile.throw_direction = throw_direction
  fake_missile.throw_matrix    = throw_matrix

  owner = the parent as an inventory owner
  IF owner uses the default throw force THEN
    fake_missile.throw_force = const_power ? force_const : throw_force
  ELSE
    fake_missile.throw_force = owner.missile_throw_force    # a creature's own tuning
  END IF
  throw_force = force_min                     # rearm for the next throw

  IF this side is authoritative AND there is a parent THEN
    send OWNERSHIP_REJECT(this item, operand: fake_missile.id)
  END IF
```

**Notes** — the two force modes are the two input gestures: the primary action throws at a
fixed configured force with no wind-up hold, the secondary action charges. A creature that is
not the player supplies its own force instead, because a stalker's throw is aimed by the
planner and must land where the planner computed.

The launch is *not* performed by calling the fake missile. It is performed by sending it a
detach event, which reaches it through the ordinary event path and triggers
`OnH_B_Independent`, which activates the physics. A rebuild must keep that indirection or
accept that a single-player launch and a networked launch take different paths.

## `setup_throw_params`

**Contract** — builds the launch frame. If this item is the owner's active item it asks the
owner's firing geometry for a position and direction (the same source a weapon's muzzle uses);
otherwise it uses the item's own transform. Builds an orthonormal frame whose forward axis is
the launch direction.

**Notes** — using the owner's fire parameters rather than the item's own transform means a
grenade launches from where the owner is aiming, not from where the animated hand happens to
be. The two differ by several centimetres and the difference matters when throwing through a
doorway.

## `spawn_fake_missile`

**Contract** — spawns a duplicate of this item, parented to this item, at this item's position
and navigation vertex, flagged not savable, and sends it as a spawn message. Authoritative side
only; skipped if this item is already being destroyed.

**Invariants** — the duplicate is the *same configuration section*, so it has the same mass,
model and fuse. It must be parented to this item at spawn, because the later ownership-reject
is what launches it and a reject without a prior take is meaningless.

**Notes** — this is the mechanism's cost: every throw spawns and destroys an entity, consuming
an entity identifier, going through the full spawn lifecycle, and reaching every connected
client. A rebuild could instead convert the held item into the flying one, but then the hand
would be empty during the release animation and the inventory would have to be told mid-animation.
The engine chose the duplicate.

## `OnEvent`

**Contract** — handles the two ownership events on top of the base item behaviour. A take
records the incoming object as this item's fake missile and snaps it to this item's position;
a reject clears the reference, detaches the object, and — on a non-authoritative side only —
starts the fuse.

```text
FUNCTION OnEvent(message, kind)
  inherited OnEvent
  SELECT kind
    CASE OWNERSHIP_TAKE
      id = read
      fake_missile = find object(id) as a missile
      fake_missile.parent = this
      fake_missile.position = this.position
    CASE OWNERSHIP_REJECT
      id = read
      was_our_fake = (fake_missile exists AND id == fake_missile.id)
      IF was_our_fake THEN fake_missile = none
      object = find object(id) as a missile
      IF object is none THEN RETURN
      object.detach(active: message has more bytes AND that byte is non-zero)
      IF was_our_fake AND this is NOT the authority THEN
        object.destroy_time = now + destroy_time_max     # start the fuse locally
      END IF
  END SELECT
```

**Invariants** — the fuse is armed on the *client* side here and on the server side elsewhere,
because the two sides learn about the launch at different moments. The condition is the
authority check, not the object's locality.

## `activate_physic_shell`

**Contract** — brings the physics body up. Two entirely different behaviours depending on
whether this object is a *fake missile* (its parent is another missile) or an ordinary dropped
item.

```text
FUNCTION activate_physic_shell()
  IF the parent is NOT a missile THEN
    # An ordinary drop: the base class's behaviour, plus — in multiplayer only — the
    # contact filter that stops it colliding with whoever dropped it.
    inherited activate_physic_shell()
    IF the body is active AND multiplayer THEN
      install ExitContactCallback with the root owner as its data
    END IF
    RETURN
  END IF

  # --- the launch ---
  linear = normalize(throw_direction) * throw_force
  IF the owner uses throw randomness THEN
    angular = a uniformly random direction on the sphere, magnitude in [2pi, 3pi] rad/s
  ELSE
    angular = zero
  END IF
  transform = throw_matrix
  IF the root owner is a living entity THEN
    linear = linear + that entity's character velocity    # a thrown object inherits motion
  END IF
  REQUIRE no body exists yet
  create the body
  activate it at throw_matrix with (linear, angular)
  mark EVERY collision shape as swept                     # see below
  install ExitContactCallback with the thrower as its data
  set air resistance to zero; set dynamic scales to unity
  force a bone recalculation so the visual matches the body immediately
```

**Invariants**

- Every shape is marked as *swept* (continuously traced between steps). A grenade leaves the
  hand fast enough to pass through a wall in one fixed timestep, and the ordinary discrete
  collision test would miss it. This is the one place in the item hierarchy where that is
  turned on, and it is not optional.
- Air resistance is explicitly zero. The throw is tuned against a pure ballistic arc, and the
  configured throw forces are meaningless with drag applied.
- The thrower's own velocity is added to the launch velocity, so a grenade thrown from a moving
  vehicle or while running goes where it looks like it should.

**Notes** — the random spin is a presentation decision with a real gameplay consequence: a
grenade that spins bounces unpredictably. It is per-owner tunable so that a scripted throw can
be made deterministic.

## `ExitContactCallback`

**Contract** — a contact filter installed on a thrown or dropped missile. Given a contact
between two shapes, it suppresses the contact when one shape belongs to the object recorded as
this missile's thrower. Static; takes the contact by reference and writes the decision through
its first argument.

```text
FUNCTION ExitContactCallback(out should_collide, first_is_ours, contact, ...)
  ours, other = the two shapes' user data, ordered by first_is_ours
  IF ours.callback_data == other.owning object THEN should_collide = false
```

**Invariants** — the filter is removed and its data cleared when the thrower is released
(`net_Relcase`), because it holds a bare reference to an object that can be destroyed while the
grenade is still in the air.

**Notes** — without this, a grenade thrown while running would immediately collide with the
thrower's own body and drop at their feet. It is not a gameplay concession; it is the fix for
the launch origin being inside the thrower's collision volume.

## `UpdateCL`

**Contract** — the per-frame update. Advances the state timer, charges the throw force while
the wind-up is being held, releases when a release has been requested, and idles into the
"bored" animation after twenty seconds of a motionless owner holding it.

```text
FUNCTION UpdateCL()
  state_time = state_time + frame time
  inherited UpdateCL()

  IF the owner is the player AND is not moving AND this is their active item THEN
    IF not tuning the first-person view AND state is Idle
       AND this substate has lasted more than 20 seconds THEN
      switch to the bored animation; reset the substate timer
    END IF
  END IF

  IF state is Ready THEN
    IF a release was requested THEN switch to Throw
    ELSE IF the owner is the player THEN
      throw_force = throw_force + force_grow_speed * frame_time_in_seconds
      clamp throw_force to [force_min, force_max]
    END IF
  END IF
```

**Notes** — the force only grows for the player. A creature's throw force comes from its own
tuning and is applied at release, so charging is a player-facing mechanic only.

## `shedule_Update`

**Contract** — the low-rate update. When the object is loose in the world with a physics body
and its fuse has expired, it destroys itself. Runs at the scheduler's degraded rate, which is
adequate: the fuse is measured in seconds.

**Invariants** — the fuse is compared against *server* time, not the frame clock, so that a
grenade's lifetime is the same on every machine.

## `OnH_B_Independent`

**Contract** — called when this object stops having an owner. If this is a real detachment
rather than a teardown, it zeroes air resistance and dynamic scales and — if the release
animation was already running — performs the launch immediately. If the fuse was never armed
and this side is authoritative, the object destroys itself.

**Notes** — the "throw on reject" branch covers the case where the item is taken out of the
owner's hands mid-animation (death, a script, a container). The grenade must still be thrown,
because the fake missile has already been spawned and would otherwise be orphaned.

## `OnH_A_Chield`

**Contract** — called when this object gains an owner. Pure delegation; the fake-missile spawn
that once lived here now happens at the end of the wind-up animation instead.

## `State` · `OnStateSwitch` · `OnAnimationEnd` · `OnMotionMark`

**Contract** — the animation binding. `State` plays the clip for each state and sets or clears
the "an action is pending" flag; `OnStateSwitch` resets the state timer and then runs `State`;
`OnAnimationEnd` advances the machine at each clip's end; `OnMotionMark` performs the launch
when the release marker fires during the throw clip.

**Invariants** — the pending flag is set for every state that must not be interrupted (showing,
hiding, wind-up, release) and cleared for the ones that may be (idle, hidden). The inventory
consults it before switching items.

The hidden state stops the current clip *without running its end callback*, so that hiding
mid-animation does not advance the state machine.

## `UpdateXForm`

**Contract** — places the item in the owner's hands, once per frame. Builds a frame from the two
hand bones: the forward axis runs from the right hand toward the left, the remaining axes are
made orthonormal against the right hand's own orientation, and the origin is the right hand.
Skipped entirely if the owner has attached the item elsewhere on their body, or if the owner has
no weapon bones.

**Notes** — deriving the orientation from the *vector between two bones* rather than from one
bone's transform is how the item stays correctly oriented across animations that were authored
for different weapon shapes. It is the same trick the weapon hierarchy uses.

The frame guard means this recomputes at most once per frame however many callers ask.

## `UpdateFireDependencies_internal`

**Contract** — recomputes the launch direction once per frame. In the third-person case it
rotates the configured direction by the owner's transform. The first-person case is
unimplemented and asserts.

**Notes** — the first-person branch is a live hole: it is unreachable in the shipped
configuration because thrown items do not reach it, but a rebuild that routes first-person
items through this path must implement it (take the direction from the first-person model's
own frame).

## `PH_A_CrPr`

**Contract** — the one-shot fixup run on the frame after a spawn: pulls the object's transform
out of its physics body, recomputes its bones and re-registers it with the spatial index.
Recomputes the bones first if the body is not yet fully active, because the body's transform is
derived from them.

**Notes** — this exists because a spawned item's transform and its physics body are established
by different code at different moments, and one frame of disagreement is visible as the item
appearing in the wrong place. Every physics-bearing item has an equivalent.

## `Action`

**Contract** — the input binding. The primary action starts a fixed-force throw; the secondary
action starts a charged one and releases on key-up.

```text
FUNCTION Action(command, flags) -> bool          # true = consumed
  IF inherited Action consumed it THEN RETURN true
  SELECT command
    CASE FIRE
      const_power = true
      IF pressed AND state is Idle THEN throw = true; switch to ThrowStart
      RETURN true
    CASE ZOOM                                    # the charged throw
      const_power = false
      IF pressed THEN
        throw = false
        IF state is Idle  THEN switch to ThrowStart
        IF state is Ready THEN throw = true      # already charged: release now
      ELSE IF state is Ready, ThrowStart or Idle THEN
        throw = true
        IF state is Ready THEN switch to Throw
      END IF
      RETURN true
  END SELECT
  RETURN false
```

**Invariants** — a release requested during the wind-up sets the flag rather than switching
state, and the wind-up's end then goes straight to the release. That is what lets a player tap
the charge key and get a minimum-force throw.

## `render_item_ui_query` · `render_item_ui`

**Contract** — the charge meter. Drawn only when this is the player's active item, the state is
the charged hold, and no release has been requested. Draws a shaped progress indicator at the
fraction of the force range currently charged, creating that indicator from its layout file on
first use.

**Notes** — the indicator is a single process-wide instance created lazily and never released.
That is a leak, and a rebuild should own it alongside the rest of the head-up display.

## `create_physic_shell` · `setup_physic_shell`

**Contract** — build the physics body from the item's own collision description, and bring it up
at the item's current transform with no velocity. `setup_physic_shell` is the "it is lying
there" path; `activate_physic_shell` is the "it is being thrown" path.

## `net_Relcase`

**Contract** — called when any object is about to be released. If the released object is this
missile's recorded thrower, the contact filter is removed and its data cleared. This is the
only thing standing between a grenade in flight and a reference to a destroyed entity.

## `Destroy` · `net_Spawn` · `net_Destroy` · `reinit` · `Load`

**Contract** — the lifecycle. `reinit` resets the throw state and starts hidden with no fake
missile and a fuse of "never". `Load` reads the four force values, the fuse length, the launch
point and direction offsets and the planner's weapon classification. `net_Spawn` invalidates the
transform cache and sets a default upward launch direction. `net_Destroy` drops the fake missile
reference. `Destroy` removes the object, authoritative side only.

## `ef_weapon_type` · `GetBriefInfo` · `AlwaysTheCrow`

**Contract** — the planner's coarse weapon classification (asserted to have been configured);
the item's short name for the inventory tooltip; and a declaration that this item is always
rendered even when far away.

## `OnActiveItem` · `OnHiddenItem`

**Contract** — selection and deselection. Selection plays the show animation and settles on idle;
deselection plays the hide animation in single player and skips straight to hidden in
multiplayer, because a multiplayer client cannot afford to wait an animation before the item is
gone.
