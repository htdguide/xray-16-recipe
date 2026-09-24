# src/xrGame/Car.cpp

> The drivable vehicle: an engine model that turns a pedal into wheel torque through a gearbox, a body made of physics joints whose wheels and doors take damage individually, and a seat the player rides in.

**Needs** — [`Car.h`](Car.h.md) · [`Entity.h`](Entity.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`DamagableItem.h`](DamagableItem.h.md) · [`DelayedActionFuse.h`](DelayedActionFuse.h.md) · [`Explosive.h`](Explosive.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`PHCollisionDamageReceiver.h`](PHCollisionDamageReceiver.h.md) · [`CarLights.h`](CarLights.h.md) · [`CarDamageParticles.h`](CarDamageParticles.h.md) · [`CarWeapon.h`](CarWeapon.h.md) · [`car_memory.h`](car_memory.h.md) · [`script_entity.h`](script_entity.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`Inventory.h`](Inventory.h.md) · [`Actor.h`](Actor.h.md) · [`CameraLook.h`](CameraLook.h.md) · [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`Level.h`](Level.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/PHUpdateObject.h`](../xrPhysics/PHUpdateObject.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`Car.h`](Car.h.md); callers name that, not this file.
**Tier floor** — T1: it runs inside the physics step's own callbacks, on a fixed sub-frame cadence, and feeds joint motor targets that must be written before the solver reads them

## Purpose

A vehicle is the most heavily multiple-natured object in the game: it is an entity with
health, a physics assembly with joints, a holder the actor sits inside, a destructible, an
explosive, a damage receiver, a script-controllable entity and an inventory container, all
at once. This file is where those natures are ordered against each other.

Two ideas carry the whole file.

**The drivetrain is a one-dimensional simulation that meets the rigid-body world only at
the wheel joints.** Engine speed, engine power and gear ratio are plain numbers advanced
here; the only thing handed to the physics library is, per driving wheel, a target angular
velocity and a maximum torque on its drive axis. Everything else — traction, weight
transfer, the car rolling over — is the rigid-body solver's business. A rebuild may
replace the whole engine model and still drive, as long as it produces that pair of
numbers per wheel.

**The vehicle's parts are individually damageable.** Wheels, doors and the body each carry
their own health and their own damage-level table, and a hit is routed to a part by the
*bone* it struck. This is why the collision proxy has to be per-bone and why the damage
system is addressed by bone identity rather than by object.

The car's tuning does **not** come from its configuration section. It comes from a
configuration file embedded in the *model* — the same place the bone hierarchy lives —
because every number here is about a specific set of bones. That is a real design decision
and a rebuild must keep the data where the bones are.

## State

```text
RECORD EngineModel
  max_power            : real   # peak power; the authored value is scaled on load (see Notes)
  max_rpm, min_rpm     : real   # redline and idle, stored as angular rate, not revolutions
  power_rpm            : real   # the speed at which peak power occurs
  torque_rpm           : real   # the speed at which peak torque occurs
  a, b, c              : real   # the three coefficients of the power curve, derived from the above
  current_rpm          : real   # smoothed toward the speed the wheels imply
  current_engine_power : real   # smoothed toward the curve's value at current_rpm
  power_increment_factor, power_decrement_factor : real   # smoothing rates, rising vs falling
  rpm_increment_factor,  rpm_decrement_factor    : real
  power_neutral_factor : real   # invariant: strictly between 0.1 and 1; power with no pedal
  axle_friction        : real

RECORD Transmission
  gear_ratios          : list<(ratio, downshift_rpm, upshift_rpm)>
                         # index 0 is reverse, with a negated ratio; 1..n are forward gears
  current_gear         : int
  current_gear_ratio   : real
  auto_switch          : bool
  switching            : bool   # a shift is in progress; torque is zero until it ends

RECORD DriverInputState          # one flag per held control, not per event
  rsp, lsp, fwp, bkp, brp : bool # right, left, forward, back, handbrake

RECORD CarParts
  wheels        : map<bone, Wheel>          # every wheel, by the bone it is attached to
  driving       : list<DriveWheel>          # the subset that receives engine torque
  steering      : list<SteerWheel>          # the subset whose steer axis is driven
  breaking      : list<BrakeWheel>          # the subset the brakes act on
  doors         : map<bone, Door>
  doors_update  : list<Door>                # only doors mid-motion; see Notes
  exhausts      : list<Exhaust>
  lights        : CarLights

RECORD CarBody
  fuel, fuel_tank, fuel_consumption : real
  engine_on, clutch, starting, stalling, breaks : bool
  break_start, break_time, breaks_to_back_rate  : real
  steer_angle       : real      # the visual steering-wheel angle, heavily smoothed
  bone_steer        : bone      # the steering wheel's bone, driven by a pose callback
  exploded          : bool
  sits_transforms   : list<transform>   # index 0 is the driver's place
  camera_position   : vector            # first-person eye point, in car space
  exit_position     : vector            # filled by whichever door agreed to let the actor out
  async_calls       : set<deferred sound/particle action>
```

Invariants:

- `bone_map` — the bone-to-physics-element table — is **shared by every car in the
  process**, rebuilt from scratch at the start of each car's definition parse and consumed
  before the next car is built. It is a scratch buffer with a misleading lifetime. A
  rebuild should make it a local passed down through the build, and must keep the
  build-then-consume ordering either way.
- every entry in `driving`, `steering` and `breaking` points at an entry in `wheels`; a
  wheel may appear in all three.
- gear 0 is reverse. Every gear-shifting operation refuses to act when the current gear is
  0, so the only way out of reverse is an explicit request.
- `doors_update` holds only doors that are opening, closing or newly opened. A door that
  reaches rest removes itself. The list is compacted in place during the same walk that
  updates it.

## `net_Spawn` and `SpawnInitPhysics`

**Contract** — build a live vehicle from its spawn record. Returns whether the spawn
succeeded. Reads the model's embedded configuration, creates the physics assembly, and
restores saved door and wheel state.

The order is the substance:

```text
FUNCTION spawn_physics(record)
  ParseDefinitions()      # read the model's embedded configuration into the part lists
  CreateSkeleton(record)  # build the physics assembly, filling the shared bone table
  recompute the pose from the bones, forcing the pose callbacks to run
  Init()                  # give every part the physics handle it now has
  SetDefaultNetState(record)   # only if the record carries no saved state
  activate the per-physics-step callback
```

**Invariants** — the pose must be recomputed *twice* around the assembly build, once
before and once after, because the physics elements are positioned from the bind pose and
the bones are then positioned from the physics elements. Skipping either leaves the visual
and the physical vehicle a frame apart on the first frame, which the player sees.

Health comes from the spawn record, not from the section. A car remembered as wrecked
spawns already marked exploded so that it does not explode again on sight.

Three optional subsystems are enabled purely by the *presence* of a section in the
model's configuration: a destroyed-state model, a mounted weapon, and a visual memory
(which lets a scripted, driverless car see). Absence is the off switch; there is no flag.

## The engine model

### `InitParabola` and `Parabola`

**Contract** — the power curve. `InitParabola` derives three coefficients from the four
authored numbers; `Parabola` evaluates power at a given engine speed. Never returns
negative power.

```text
FUNCTION init_power_curve()
  a = exp((power_rpm - torque_rpm) / (2 * power_rpm)) * max_power / power_rpm
  b = torque_rpm
  c = sqrt(2 * power_rpm * (power_rpm - torque_rpm))

FUNCTION power_at(rpm) -> real
  x = (rpm - b) / c
  value = a * exp(-x*x) * rpm
  IF value < 0 THEN RETURN 0
  IF the drivetrain is in neutral THEN value = value * power_neutral_factor
  RETURN value
```

**Notes** — the curve is *torque shaped as a Gaussian in engine speed, times speed*, which
gives power peaking above the torque peak, as a real engine does. The function is still
named for the polynomial it used to be; a commented-out quartic sits above it. The three
coefficients are chosen so that the peak lands at the authored torque speed with the
authored maximum power — but `c` is not the width that makes those two constraints exactly
consistent, and no derivation for it survives. Reproduce the expressions.

Running in neutral scales the whole curve rather than clamping the throttle: an engine
with no load still revs, and the factor is what keeps it from revving as hard as one under
power.

### `EngineDriveSpeed` and `EnginePower`

**Contract** — advance the two smoothed engine numbers by one step. Both are
exponentially smoothed with *different rates for rising and falling*, which is what makes
an engine sound and feel different accelerating than it does lifting off.

```text
FUNCTION target_engine_speed() -> real
  IF a shift is in progress THEN
    target = max_rpm                       # the engine is held at the redline during a shift
    IF current_rpm has climbed past power_rpm THEN the shift is over
  ELSE
    target = abs(mean driving-wheel angular rate * current_gear_ratio)
    IF the clutch is out AND target < min_rpm THEN target = min_rpm   # idle floor
    clamp target to max_rpm
  RETURN smooth(current_rpm -> target, rising rate or falling rate)

FUNCTION engine_power() -> real
  value = power_at(current_rpm)
  IF the starter is engaged THEN
    IF current_rpm < min_rpm THEN value = power_at(min_rpm)   # the starter fakes idle power
    ELSE IF more than one second has passed since the starter engaged THEN disengage it
  RETURN smooth(current_engine_power -> value, rising rate or falling rate)
```

**Invariants** — engine speed is *derived from the wheels*, never integrated on its own.
The engine has no inertia of its own in this model; what looks like inertia is the
smoothing. That is why the clutch matters only as an idle floor, and why stalling is
detected by the smoothed speed falling below idle rather than by any torque balance.

**Notes** — the one-second starter window is the whole starting model: for one second
after a start request the engine is given idle power regardless of how slowly it is
turning, which is enough to get the car rolling, after which the real curve takes over.

### `UpdatePower` and the automatic gearbox

**Contract** — recompute both engine numbers, shift if the gearbox is automatic, then push
the resulting torque and speed limit into every driving wheel.

```text
FUNCTION update_power()
  current_rpm          = target_engine_speed()
  current_engine_power = engine_power()
  IF automatic AND no shift in progress THEN
    IF current_rpm < this gear's downshift speed THEN shift down
    IF current_rpm > this gear's upshift speed   THEN shift up
  FOR EACH driving wheel: push the new torque and speed limit
```

**Notes** — the shift thresholds are per gear and ship in the data, as the second and
third components of each gear's authored triple. Both tests run in the same pass, so a
gear whose two thresholds overlap will shift twice; the shipped data does not do that.

### `Transmission`, `TransmissionUp`, `TransmissionDown`, `CircleSwitchTransmission`

**Contract** — select a gear. A change queues the transmission sound, latches the shifting
flag (which zeroes wheel torque until the engine reaches its power speed again) and
re-applies drive.

**Invariants** — all three relative operations refuse to act while in reverse. The circular
switch skips gear 0 on wrap-around, so cycling through the gears never lands in reverse by
accident.

## The driver controls

### `PressForward`, `PressBack`, `ReleaseForward`, `ReleaseBack`

**Contract** — the accelerator and the reverse control, as a small state machine over the
two held flags. Pressing one while the other is held does **not** do the obvious thing.

```text
FUNCTION press_forward()
  IF back is held THEN declutch and go to neutral     # opposing inputs cancel, they do not fight
  ELSE drive forward: clutch in, select gear 1 if in reverse, engage the starter, apply drive
  mark forward held

FUNCTION press_back()
  IF forward is held THEN declutch and go to neutral
  ELSE declutch, go to neutral, and START BRAKING     # not reverse: brake first
  mark back held
```

**Invariants** — **the reverse control brakes before it reverses.** Holding it applies an
increasing brake, and only when the car's forward velocity has actually reached zero does
the car engage reverse (see `UpdateBack`). This is the reason a vehicle in this game cannot
be slammed from forward into reverse, and it is a deliberate substitute for a separate
brake control.

Releasing one control while the other is still held hands the car to *that* one, so a
driver who holds both and releases one gets an immediate direction change with no
intermediate neutral.

### `UpdateBack`

**Contract** — called every physics step while the reverse control is held. Ramps the brake
in over the configured brake time, then flips to reverse once the car is no longer moving
forward.

```text
FUNCTION update_back()
  IF not braking THEN RETURN
  k = clamp(elapsed since braking began / break_time, 0, 1)
  apply brake torque scaled by k to every braking wheel
  IF the body's velocity along its own forward axis is no longer positive THEN
    stop braking
    drive in reverse
```

**Notes** — the ramp exists so that the brake does not lock the wheels instantly at speed;
the same ramp is what makes a long press feel like braking and a short one like a tap.

### `PressRight`, `PressLeft`, `ReleaseRight`, `ReleaseLeft`, `Steer`, `LimitWheels`

**Contract** — steering is three-valued (left, centre, right), not continuous, on keyboard
input; the analogue path in [`CarInput.cpp`](CarInput.cpp.md) bypasses these and calls
`Steer` with a real value. Holding both directions centres the wheel, unless the
accelerator is also held, in which case the current steer is kept.

`LimitWheels` re-imposes the steer-axis stops on wheels that are not being actively
steered, once per physics step, and is suppressed for the step in which a steer happened.

**Notes** — the "both held, accelerator also held" exception is not obviously intentional
and no comment explains it. It has the observable effect that a driver mashing both
steering controls while accelerating keeps whatever lock they had, rather than snapping
straight.

## `PhDataUpdate` — the per-physics-step hook

**Contract** — runs inside the physics step, between collision detection and the constraint
solve, once per sub-step. This is where every continuous vehicle quantity is advanced.

```text
FUNCTION on_physics_step(step)
  IF repairing THEN apply the righting force
  LimitWheels()
  UpdateFuel(step)
  UpdatePower()
  IF the engine is on, the starter is disengaged, and speed is below idle THEN Stall()
  IF the reverse control is held THEN UpdateBack()
  IF the handbrake control is held THEN HandBreak()

  walk doors_update, dropping doors that have come to rest and updating the rest
  steer_angle = 10% of the steering wheels' actual angle + 90% of the previous value
```

**Invariants** — this must run at the physics cadence, not the frame cadence. Fuel is
consumed per step; the engine's smoothing rates are per step; the brake ramp reads real
time but is sampled per step. A rebuild that moves this to the frame loop changes the
vehicle's behaviour with frame rate.

The visual steering-wheel angle is smoothed at a tenth per step *because the joint's
actual angle oscillates* under the solver, and an unsmoothed steering wheel visibly
judders.

### `PhTune`

**Contract** — runs before the solve. Applies, to every element of the assembly, an upward
force equal to that element's mass times the gravity difference.

**Notes** — this is how the car gets **half gravity while driven**. `EffectiveGravity`
halves gravity whenever the per-step hook is active, and this function injects the
difference back as a force so that only the vehicle, and not the world it collides with,
experiences it. The reason is handling: the shipped vehicles are unstable and roll over
under full gravity at the speeds the engine model reaches. Impulses from hits are scaled by
the square root of the same ratio so that a grenade does not throw a half-weight car twice
as far.

## Damage, death and explosion

### `Hit`, `ChangeCondition`, `ApplyDamage`, `PHHit`

**Contract** — a hit is routed three ways before it touches the vehicle's own health: to a
wheel, to a door, and to the physics assembly as an impulse. Only then is it scaled by the
per-bone damage table and the per-type immunity table and applied to the body.

```text
FUNCTION hit(descriptor)
  WheelHit(damage, bone, type)     # the wheel on that bone takes it
  DoorHit(damage, bone, type)      # the door on that bone takes it
  IF the type is not a melee strike THEN look up the per-bone hit and wound scales
  damage = damage * immunity(type) * hit scale
  apply to the body's health
  IF the explosion fuse is not already lit THEN test the new health against its threshold
  run the damage-level effect
```

**Invariants** — the steering-wheel bone is excluded from physical impulses, because it is
a joint driven by a pose callback and pushing it desynchronizes the visual from the
physical.

The vehicle's damage level is a three-step ladder, and each step does something different:
level 1 starts the light smoke particles, level 2 starts the heavy smoke **and lights the
explosion fuse**, level 3 empties the fuel tank. So a badly damaged car dies of fuel
starvation even if it is never destroyed.

### `CarExplode`

**Contract** — destroy the vehicle. Idempotent: a second call does nothing.

```text
FUNCTION explode()
  IF already exploded THEN RETURN
  mark the skeleton as not needing to be saved      # a wreck is not restored from a save
  deactivate the mounted weapon; turn the headlights off
  mark exploded; emit the explosion event

  IF an actor is riding THEN
    compute an exit position from the first door, or the car's own position if there are none
    eject the actor
    IF the actor is dead THEN destroy their character controller
  IF the destroyed-state model exists THEN switch to it
```

**Invariants** — the actor is ejected *before* the body is replaced by its destroyed form,
because the destroyed form has no seat and no doors. The exit position is taken from a
door even though the door may be on the wrong side or blocked — a player who would
otherwise be left inside an exploding wreck is worth a bad exit.

**Notes** — the fuse (`CDelayedActionFuse`) is a delay between reaching damage level 2 and
exploding, configured per vehicle and defaulting to two minutes. It gives the player time
to get out of a burning car. Scripts can set and read it.

`CanRemoveObject` refuses to let the object be released until the explosion has both
happened and finished making noise, because the sound outlives the body.

## The actor as passenger

### `attach_Actor`, `detach_Actor`

**Contract** — seat or eject the actor. Attaching fails if the vehicle already has a rider
or has been destroyed.

```text
FUNCTION attach(actor)
  IF occupied or destroyed THEN RETURN false
  take ownership of the actor
  IF the model names a driver place THEN seat transform = that bone's transform
  ELSE hide the actor entirely and seat them at the root bone
  switch to the first-person camera
  wake the physics assembly and install the actor-obstacle contact filter
  request unconditional per-frame updates
  release the handbrake
```

**Invariants** — a model with no authored driver place hides the rider rather than placing
them badly. Detaching reverses every step, and additionally puts the car in neutral,
disengages the clutch, clears every held control, drops the engine to idle and **applies
the handbrake** — a car the player steps out of must not roll away.

The actor-obstacle contact filter is installed only while occupied. It forces collisions
with surfaces marked as actor obstacles that the vehicle would otherwise drive through,
because those surfaces exist to stop a walking player and the vehicle is large enough to
find the gaps.

### `Use`

**Contract** — the single "press use" entry point, which means three different things
depending on where the player is looking and whether they are already inside.

```text
FUNCTION use(eye position, look direction, foot position) -> bool
  IF nobody is riding AND some door will admit us THEN RETURN true   # enter
  cast a short ray against this vehicle's own collision proxies
  FOR EACH hit, nearest first
    IF the hit element is a working door THEN
      front = is this door on the side we are approaching from
      IF (riding AND not front) OR (not riding AND front) THEN operate the door
      RETURN false
  IF riding THEN RETURN whether some door will let us out
  RETURN false
```

**Notes** — the front/back asymmetry is the decision worth keeping: from outside you may
only work the door you are facing; from inside you may only work the doors you are *not*
facing, because the one you are facing is the one you would exit through and the exit test
owns that gesture.

`Enter` averages the eye position and the foot position before testing, so that a player
standing close and looking down still presents a sensible point to the door's area test.

## `UpdateCL`, `UpdateEx`, `VisualUpdate`

**Contract** — the per-frame half. `UpdateCL` is the unconditional per-frame update;
`UpdateEx` is the variant the holder calls when the actor is riding and supplies the
current field of view.

```text
FUNCTION visual_update(fov)
  read the interpolated assembly transform into the object's transform
  update the engine sound
  IF occupied AND the assembly is awake THEN place the rider at the driver seat transform
  update the exhaust particle emitters
  update the lights
```

**Invariants** — the transform comes from the physics assembly's *interpolated* pose, not
its stepped pose, because the physics runs at a fixed rate and the frame does not.

The rider is re-seated only while the assembly is awake. A sleeping car does not move, and
re-seating against a sleeping body every frame fights the actor's own position.

An unoccupied car is still updated per frame, at a fixed field of view of 90, purely so
that its sound, exhaust and lights keep working while the player watches from outside.

### `ASCUpdate` — deferred effects

**Contract** — a three-bit set of actions requested from inside the physics step and
performed at the top of the next frame: switch the transmission sound, stop the engine
sound, stop the exhaust particles.

**Notes** — this exists because the sound and particle systems may not be touched from
inside a physics callback. The set is a set, not a queue: requesting the same action twice
in one step performs it once. A rebuild whose audio and particle layers are safe to call
from anywhere deletes this; one that defers work out of a physics step needs the same idea
under some name.

## Fuel

### `UpdateFuel`, `AddFuel`, `ChangefFuel`

**Contract** — consume fuel per physics step in proportion to how far above idle the engine
is turning, with idle itself still costing fuel. Stops the engine when the tank empties.
`AddFuel` returns how much was actually taken.

```text
FUNCTION update_fuel(step)
  IF the engine is off THEN RETURN
  IF current_rpm > min_rpm THEN fuel = fuel - step * (current_rpm - min_rpm) * consumption
  ELSE                          fuel = fuel - step * min_rpm * consumption
  IF fuel is spent THEN StopEngine()
```

**Notes** — the authored consumption figure is divided by one hundred thousand on load. The
scale factor has no stated justification; it exists to make an authored number in
convenient units come out right against engine speeds expressed as angular rates. Keep it.

## Save and restore

### `SaveNetState`, `RestoreNetState`, `SetDefaultNetState`

**Contract** — a vehicle's saved state is: the skeleton's physics state, its position and
orientation, every door's open-state and health, every wheel's health, and the body's
health. Restore walks the saved lists **in parallel with the live part maps**, by position.

**Invariants** — this is a positional correspondence, not a keyed one. The saved door list
and the live door map must have the same length and the same order, which holds only
because both are built from the same model in the same way. A rebuild should key by bone
identity instead; the format is frozen against itself, not against the shipped data, so
this is one of the few places a save format may legitimately be improved.

A record with no saved state gets the default: every door forced closed, which also
deactivates its joint.

**Notes** — a large commented-out block in the restore path attempted to re-place a
restored car by growing an activation shape and pushing it out of whatever it had been
saved intersecting. It is disabled, so a car saved inside geometry is restored inside
geometry and the solver resolves it however it likes.

## `OnEvent` — the boot

**Contract** — a vehicle is an inventory container. Two ownership events are handled: an
item offered to it is taken if the container will accept it and explicitly rejected back to
the sender if not; an item rejected out of it is dropped, carrying a flag that says whether
the drop is happening because the item is about to be destroyed.

**Notes** — the inventory is created with slots disabled: a car has no equipment slots,
only a volume. The rejection message is the mechanism that keeps the sender and the
container in agreement when a transfer fails, and it must exist in any rebuild that moves
items by message rather than by call.

## Parts and definitions

### `ParseDefinitions`

**Contract** — read every authored number out of the model's embedded configuration and
build the four part collections. Fails hard if the model carries no configuration at all.

Unit conversions applied on load, all of them load-bearing:

- **power** is multiplied by 800 — the authored figure is in some convenient unit and the
  simulation wants another. The factor is written as `0.8 * 1000` and is not explained.
- **every engine speed** is converted from revolutions per minute to radians per second.
  The engine model works entirely in angular rate, which is what the wheel joints speak.
- **reverse's gear ratio is negated** and every ratio is multiplied by the final drive
  ratio, so the gearbox afterwards is a single multiply.
- **fuel consumption** is divided by 100000.

### `Init`

**Contract** — after the physics assembly exists, give every part its joint handle and its
health, install the steering-wheel pose callback, and set the starting state: handbrake on,
first gear.

**Notes** — a wheel gets 100 health and a two-level damage ladder; a door gets 100 and a
one-level ladder; the body's ladder has three levels. The `damage_items` section overrides
a specific wheel's or door's health by bone name, and rejects any bone that is neither.

### `fill_wheel_vector`, `fill_exhaust_vector`, `fill_doors_map`

**Contract** — turn a space-separated list of bone names into part records, registering each
bone in the shared bone table so the physics builder knows to make an element for it.

**Invariants** — a wheel named in more than one of the three lists yields **one** wheel
record referenced three times. The check is "is this bone already in the shared table",
which is why the table has to be cleared before each car and why the driving list must be
parsed before the steering and braking lists that may reuse its wheels.

## Smaller exported units

- **`cb_Steer`** — the pose callback that rotates the steering-wheel bone by the smoothed
  steer angle. It asserts the resulting transform is still a rotation, because a
  degenerate bone transform propagates into the skinning and produces a collapsed model.
- **`Starter`, `StartEngine`, `StopEngine`, `Stall`, `SwitchEngine`** — the engine's on/off
  path. Starting refuses with an empty tank. Stopping and stalling are the same code with
  different sounds; both go to neutral and zero the engine speed.
- **`Clutch`, `Unclutch`, `ReleasePedals`, `ResetKeys`** — the clutch is a single flag whose
  only effect is whether the idle floor applies.
- **`HandBreak`, `ReleaseHandBreak`, `StartBreaking`, `StopBreaking`, `PressBreaks`,
  `ReleaseBreaks`** — two independent brake paths: the handbrake, which is a full-torque
  lock, and the ramped service brake the reverse control drives.
- **`Revert`** — pushes the car upward at 1.5 times its own weight, used while "repairing"
  to right an overturned vehicle.
- **`Initiator`** — who is blamed for damage this car causes: the rider if there is one and
  the car is alive, otherwise the car itself. This is what makes running someone over count
  against the player.
- **`AlwaysTheCrow`** — the vehicle demands a per-frame update whenever its mounted weapon
  is active, overriding the scheduler's distance-based rate reduction, because a firing
  turret cannot be updated lazily.
- **`RefWheelMaxSpeed`, `EngineCurTorque`, `RefWheelCurTorque`, `DriveWheelsMeanAngleRate`,
  `EffectiveGravity`, `AntiGravityAccel`, `GravityFactorImpulse`, `ExitVelocity`,
  `CurrentVel`** — derived quantities. Torque during a shift is zero.
- **`GetfFuel`, `SetfFuel`, `GetfFuelTank`, `SetfFuelTank`, `GetfFuelConsumption`,
  `SetfFuelConsumption`, `ChangefHealth`, `isActiveEngine`, `GetRPM`, `SetRPM`,
  `PlayDamageParticles`, `StopDamageParticles`** — the script surface, exported in
  [`CarScript.cpp`](CarScript.cpp.md). `ChangefHealth` clamps health to the range zero to
  one, which is the only place in this file that treats health as normalized.

## Notes

`net_SaveRelevant` unconditionally reports that a car is worth saving. The condition it
replaced — do not save a car that is mid-explosion, exploded, or destroyed — is commented
out directly above it, so wrecks now persist in saves. Whether that is a fix or a
regression is not recoverable; it is the shipped behaviour.

`net_Export` and `net_Import` add nothing of their own. A vehicle's network state is
whatever its base entity sends, which means a driven car is not properly replicated. Given
that the shipped build has no working transport at all (see the networking seam), this is
consistent with multiplayer being unfinished rather than broken.
