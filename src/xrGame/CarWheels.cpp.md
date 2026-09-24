# src/xrGame/CarWheels.cpp

> The vehicle's wheels — how a skeleton bone with a wheel joint becomes a driven,
> steered, braked and damageable axle, and how damage is expressed as joint softening.

**Needs** — [`Car.h`](Car.h.md) · [`DamagableItem.h`](DamagableItem.h.md) · [`CarDamageParticles.h`](CarDamageParticles.h.md) · [`Hit.h`](Hit.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes contact parameters from inside the solver's collision
callback, on the solver's own contact records.

## Purpose

A car in this engine is not a special-cased vehicle controller. It is an ordinary jointed
rigid-body assembly built from the model's skeleton, and a "wheel" is just a bone whose
joint is a two-axis wheel joint — one axis to steer about, one to spin about. This file is
the four roles that assembly plays: the wheel itself, the driven wheel, the steered wheel
and the braked wheel. They are separate records because a given wheel may be any
combination of the three, and the shipped vehicles use every combination.

The design consequence worth carrying into a rebuild: **there is no tyre model**. Grip is
whatever the solver's friction produces at the contact, adjusted by a per-wheel multiplier.
Everything that would be a tyre model elsewhere is expressed as joint motors and contact
parameter scaling.

## State

```text
RECORD WheelCollisionParams
  spring_factor  : real = 1     # multipliers applied to the contact's
  damping_factor : real = 1     # spring/damper pair — the suspension's compliance
  mu_factor      : real = 1     # friction multiplier — the tyre's grip

RECORD Wheel                     # also a damageable health item
  bone_id           : bone id
  inited            : bool
  radius            : real       # taken from the collision shape, not from configuration
  joint             : wheel joint
  car               : Car
  collision_params  : WheelCollisionParams

RECORD WheelDrive                # a wheel the engine turns
  wheel       : Wheel
  gear_factor : real             # this wheel's radius / the car's reference wheel radius
  pos_fvd     : real             # +1 or -1: which way this wheel's spin axis points

RECORD WheelSteer                # a wheel the steering turns
  wheel       : Wheel
  pos_right   : real             # +1 or -1: which side of the car this wheel is on
  lo_limit    : real             # the joint's authored steering travel
  hi_limit    : real
  limited     : bool             # steering travel currently pinned shut at centre

RECORD WheelBreak                # a wheel the brakes act on
  wheel             : Wheel
  break_torque      : real       # already scaled by this wheel's radius ratio
  hand_break_torque : real
```

**Invariants** — a wheel's `radius` and the joint it drives are discovered from the built
physics shell, so `init` may only run after the shell exists and the bone-to-element map is
filled. Initialization is idempotent and guarded by `inited`, because a wheel can be
referenced from all three role records and each of them initializes it.

`gear_factor` and the two brake torques are both expressed relative to a **reference wheel
radius** declared on the car. This is the trick that lets the whole drivetrain be tuned in
one set of units regardless of wheel size: the engine's output is defined at the reference
wheel, and each real wheel divides by its own ratio.

## initialization

**Contract** — bind a wheel record to the physics element and joint that the skeleton's bone
produced, and set the wheel's invariant physical properties. Fails loudly if the model's
bone has no collision element or no wheel joint — a car whose wheels are not wheels cannot
be silently degraded into something drivable.

```text
FUNCTION Wheel.init()
  IF inited THEN RETURN
  element = physics element for bone_id                 # must exist
  joint   = joint      for bone_id                      # must exist and be a wheel joint
  raise the element's dynamic velocity limits           # angular limit x100: see Notes
  radius = element's shape radius
  register this wheel's back-reference in the joint     # so the joint can null us on death
  apply zero drive velocity and zero drive torque       # free-spinning until the engine acts
  attach the contact callback, with collision_params as its payload
  disable air resistance on the element
  inited = true
```

**Notes** — the angular velocity limit is raised by a factor of a hundred above the default
used by ordinary debris. The default exists to stop a physics object spinning itself into
numerical nonsense; a wheel at speed legitimately exceeds it by an order of magnitude, and
clamping it would silently cap the car's top speed.

Air resistance is switched off per wheel. The car's aerodynamic drag is modelled once on the
body; leaving it on the wheels as well would apply it four extra times.

## contact parameters

**Contract** — a collision callback the solver invokes for every contact a wheel makes,
before the constraint solve. It scales the contact's friction and its spring/damper
compliance by the wheel's own multipliers. Both sides of the contact are consulted, so a
wheel-versus-wheel contact gets both wheels' factors.

```text
FUNCTION on_wheel_contact(contact, material_a, material_b)
  FOR EACH side IN (first shape, second shape)
      ud = side's user data
      IF ud declares a wheel-collision callback
          p = ud's payload as WheelCollisionParams
          contact.friction = contact.friction * p.mu_factor
          scale contact's spring and damper by p.spring_factor, p.damping_factor
```

**Invariants** — this *scales* what the material system already decided; it never replaces
it. Driving onto ice must still be slippery. The wheel's factor is the vehicle's own
character — a truck's tyres grip more than a car's on the same surface.

**Notes** — the spring and damper are not scaled independently. They are converted to the
solver's constraint-softness pair, scaled together and converted back, because scaling the
softness terms directly would change the effective stiffness and the damping ratio in a
coupled way that no tuner could reason about. The suspension *is* the contact here: there is
no separate spring between hub and body.

## `load`

**Contract** — read the three collision factors from the model's embedded configuration,
preferring a per-wheel section and falling back to a shared `wheels_params` section. A model
with neither keeps the neutral defaults of one.

Brake torques come from `car_definition` (`break_torque`, and `hand_break_torque` defaulting
to it), optionally overridden per wheel. The hand brake defaulting to the service brake means
a vehicle that does not distinguish them needs only one number.

## driving a wheel

**Contract** — the driven wheel converts the car's reference-wheel figures into this wheel's
own units and applies them as a motor on the spin axis.

```text
FUNCTION WheelDrive.init()
  wheel.init()
  gear_factor = wheel.radius / car.reference_wheel_radius
  pos_fvd = sign of the wheel element's spin-axis component  ->  -1 if positive else +1

FUNCTION WheelDrive.drive()         set spin motor velocity to
                                      pos_fvd * car.reference_wheel_max_speed / gear_factor
FUNCTION WheelDrive.update_power()  set spin motor torque   to
                                      car.reference_wheel_current_torque / gear_factor
FUNCTION WheelDrive.neutral()       set spin motor velocity 0, torque = car.axle_friction
FUNCTION WheelDrive.angular_speed() spin-axis rate * pos_fvd     # signed, forward positive
```

**Invariants** — a wheel is driven by commanding a target *velocity* with a torque *ceiling*,
not by applying a torque. The ceiling is the engine's available torque at the current gear
and revolutions; the target velocity is the speed that gear could reach. The solver then
produces exactly as much torque as the wheel can transmit, which is why the car naturally
bogs down on a hill and slips when the torque ceiling exceeds what friction can hold. This
is the whole drivetrain model and it is worth preserving verbatim: a rebuild that applies
torque directly will need a slip model it does not otherwise have.

Neutral is the same mechanism with a zero target and a torque ceiling of the car's axle
friction — engine braking, expressed as a very weak motor pulling toward a stop.

**Notes** — `pos_fvd` exists because a modeller may build the left and right wheels mirrored,
which flips the sign of their joint axes. Rather than demand a convention from the art, the
code reads the built axis and normalizes it. The mapping is deliberately *inverted*
(a positive axis component yields −1), which is only correct in company with the joint axis
convention the shell builder uses; the two must be changed together or not at all.

## steering a wheel

**Contract** — steering is not a position command. The steering motor always runs at a fixed
speed toward one side, and the *joint's travel limit* is moved to where the wheel should
stop. Angle is an input in the range −1…+1, a fraction of the authored travel.

```text
FUNCTION WheelSteer.init()
  wheel.init()
  (lo_limit, hi_limit) = the joint's authored steering travel
  pos_right = sign of the wheel element's lateral axis  ->  -1 if positive else +1
  apply the car's steering torque as the steer motor's torque ceiling
  set the joint's fudge factor to 0.005 / steering_torque        # see Notes
  set steer motor velocity to zero;  limited = false

FUNCTION WheelSteer.steer(angle)
  limited = (angle == 0)
  pick the side to travel toward from (sign of angle, or the car's steer state when centring)
  combined with pos_right, and then either
      raise the high limit to hi_limit * |angle| and run the motor positive
  or
      lower the low  limit to lo_limit * |angle| and run the motor negative

FUNCTION WheelSteer.limit()                         # called while returning to centre
  IF NOT limited AND |current steer angle| < one degree
      pin both travel limits to zero; stop the motor; limited = true
  car.wheels_limited = car.wheels_limited AND limited

FUNCTION WheelSteer.steer_angle()  -> -pos_right * current steer-axis angle
```

**Invariants** — only *one* of the two limits is moved per command; the other stays where it
was. The wheel is therefore free to travel from wherever it is toward the commanded side and
is stopped by the limit, and the opposite limit continues to hold it from the other
direction. Commanding the centre runs the motor toward zero and then `limit` slams both
limits shut once the wheel is within a degree, which is what actually centres the steering —
a motor alone would hunt around zero forever.

`limited` starts as "is this a centring command" and is then refined by `limit`; the car ORs
all its steered wheels together to know when the steering has finished returning.

**Notes** — the fudge factor is the solver's correction for a joint driven far past what one
step can satisfy. It is set inversely proportional to the steering torque so that a stronger
steering motor gets a proportionally smaller correction — the product, and therefore the
per-step correction magnitude, stays constant across vehicles with very different steering
strengths. The constant itself is a tuning value with no derivation available in the source.

`pos_right` mirrors the command for wheels on the far side of the car, so a single steering
input turns both front wheels the same way in world terms even though their joint axes point
opposite ways.

## braking a wheel

**Contract** — braking is the drive motor commanded to a stop with a very large torque
ceiling.

```text
FUNCTION WheelBreak.init()   wheel.init()
                             scale break_torque and hand_break_torque by
                               wheel.radius / car.reference_wheel_radius
FUNCTION WheelBreak.break(k) set spin motor velocity 0, torque = 100000 * break_torque * k
FUNCTION WheelBreak.hand()   set spin motor velocity 0, torque = 100000 * hand_break_torque
FUNCTION WheelBreak.neutral()set spin motor velocity 0, torque = car.axle_friction
```

**Notes** — the hundred-thousand multiplier is a unit bridge, not a physical quantity: the
authored brake numbers are in a convenient small range and the solver wants torque in the
same units as the engine's. It means an authored value of 1 is already a locking brake, and
the fractional pedal input `k` is what makes it progressive.

Brake torques scale with radius for the same reason the drive does: a bigger wheel needs
proportionally more torque at the hub for the same force at the contact patch.

## wheel damage

**Contract** — a wheel is a damageable health item; a hit on a bone that belongs to a wheel
is routed to that wheel's health rather than to the car's. Two damage levels have effects,
and they are cumulative in the sense that level two is reached through level one.

```text
FUNCTION Car.wheel_hit(power, bone) -> bool
  IF bone names a wheel THEN apply power to that wheel's health; RETURN true
  RETURN false                                  # not a wheel; the car handles it

FUNCTION Wheel.apply_damage(level)
  CASE 1: soften the joint  (spring / 20, damper * 4);  play the level-one damage particles
  CASE 2: perturb the spin axis by a small equal offset on all three components and
          renormalize;  soften further (spring / 30, damper * 8);
          play the level-two damage particles
```

**Notes** — a damaged wheel is modelled entirely as a *softer, sloppier joint*. Level one
makes the suspension mushy; level two additionally tilts the spin axis a little, which is
what produces the visible wobble and the pull to one side. Perturbing all three components
equally is a cheap way to tilt an axis in an unknown orientation without caring which way it
originally pointed — the direction of the resulting tilt is arbitrary and that is acceptable,
because a bent wheel's lean is arbitrary too.

The spring and damper are divided and multiplied rather than set, so the effect compounds
correctly on a wheel that was already tuned unusually.

## wheel persistence

**Contract** — a wheel writes exactly its health into the network/save stream, and restores
from it by setting health and replaying whatever damage effect that health implies. Nothing
else about a wheel is state: its joint softening and its particle effect are both derived
from health, so restoring health restores the wheel completely.
