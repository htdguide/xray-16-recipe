# src/xrGame/PHSoundPlayer.cpp

> One collision sound at a time per physical object, chosen by the pair of materials that struck.

**Needs** — [`PHSoundPlayer.h`](PHSoundPlayer.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`PHSoundPlayer.h`](PHSoundPlayer.h.md); callers name that, not this file.
**Tier floor** — T2: one sound handle and a gate

## Purpose

A physically simulated object resting on the ground generates collision contacts every step.
Playing a sound for each would be a continuous roar. This object is the gate: **one
collision sound at a time per physical object**, and only while the object is actually
moving.

The sound itself is not chosen here — it comes from the *material pair* the game's material
library holds for "this material struck that one", which is where wood-on-stone differs from
metal-on-metal.

## State

```text
RECORD PHSoundPlayer
  sound  : sound handle      # the one sound currently playing, or none
  object : PhysicsShellHolder
```

**Invariants** — at most one sound handle is live. While it plays, every further collision
on this object is silent. That is the whole throttle, and it is why an object tumbling down
a staircase sounds like a series of separate impacts rather than a continuous scrape.

## `Play`

**Contract** — plays a randomly chosen sound for a material pair at a position, if nothing is
already playing for this object and the object is moving fast enough to be worth hearing.
The sound is cloned from the library's entry so that its position and lifetime belong to this
object rather than to the shared library.

```text
FUNCTION play(material_pair, position)
  IF a sound is already playing for this object
    RETURN
  velocity = the object's current linear velocity
  IF velocity.squared_magnitude <= 0.01        # ~0.1 units per second
    RETURN                                     # resting contacts are silent
  REQUIRE the material pair declares at least one collision sound
  pick one at random; clone it; play it at the given position, attached to
    the object so it follows and stops with it
```

**Invariants** — the speed gate is on the *object's* velocity, not on the impact's relative
speed. An object being pushed slowly along a floor is silent even where the contact is
scraping; an object flying past something and brushing it makes a full impact sound. The
volume is not scaled by speed at all — it is a threshold, not a curve.

**Notes** — the threshold squared is 0.01, meaning one tenth of a unit per second, which is
barely moving. It is there to silence settled objects rather than to model impact energy.

A material pair with no declared collision sound is a hard failure rather than silence,
which makes an incomplete material library a crash at the first collision rather than a
missing sound. That is the right choice for shipped data that must be complete.

## Construction and destruction

**Contract** — binds to one object for life. Destruction stops whatever is playing and drops
the object link, so a sound never outlives the thing making it.
