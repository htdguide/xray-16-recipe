# src/xrMaterialSystem/GameMtlLib_Engine.cpp

> Reads a pair record and turns its five comma-separated name lists into live audio, particle and decal resources — the only part of the module that touches a device.

**Needs** — [`GameMtlLib.h`](GameMtlLib.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md) · [`Include/xrRender/WallMarkArray.h`](../Include/xrRender/WallMarkArray.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it acquires audio-device handles and renderer-created decal arrays whose release order against device teardown is fixed.

## Purpose

This file exists as a separate compilation unit for one reason: it is the half of the
material system that needs a sound device and a renderer. The authoring tool links
[`GameMtlLib.cpp`](GameMtlLib.cpp.md) and substitutes its own version of this half, so
that editing a library does not require bringing up a game. That is the whole of the
split's meaning; a rebuild with one build target should merge the two.

The substance is a single decision repeated five times: a pair's media are authored as one
string per kind, holding several names separated by commas, and loading a pair means
splitting each string and asking the corresponding subsystem for a handle per name.

## State

```text
Stateless. The records it fills are defined in GameMtlLib.h.
```

## `load_pair(reader)`

**Contract** — fills one pair record from one chunk sequence. Every chunk it reads is
required; a pair with no step sounds still carries an empty step-sound string, so a
missing chunk means a corrupt file and fails hard. Acquires audio-device handles for up to
eighteen sounds and asks the renderer for up to four decal shaders, so it must run after
both devices exist. Allocates.

```text
FUNCTION load_pair(reader)
  REQUIRE chunk PAIR
    material_0 <- int32 ; material_1 <- int32
    id         <- int32 ; parent_id  <- int32
    own_props  <- int32                    # kept, never consulted while playing

  REQUIRE chunk BREAKING ; breaking_sounds <- sounds_from(string)
  REQUIRE chunk STEP     ; step_sounds     <- sounds_from(string)
  REQUIRE chunk COLLIDE
    collide_sounds    <- sounds_from(string)
    collide_particles <- names_from(string)
    collide_marks     <- decals_from(string)
```

**Invariants** — the three strings inside the collide chunk are read in that order and
nothing separates them but their terminators, so a reader that skips one desynchronizes
the other two.

**Notes** — the parent link and the own-property bits are authoring state: the tool
resolves inheritance into each pair before writing, so by the time the game reads a pair
it is already complete. A read-only rebuild keeps the four bytes each occupies and ignores
both.

## Sub-item lists

**Contract** — each of the five media fields is one string holding zero or more names
separated by commas. Splitting yields the list; an empty string yields an empty list,
which is the normal way to say "this combination makes no sound".

```text
FUNCTION sounds_from(text) -> list<Sound>
  names <- split text on ','
  REQUIRE count(names) <= 6
  FOR EACH name IN names
    acquire a sound handle for name, as an effect, with the source's own game type

FUNCTION names_from(text) -> list<text>          # particle effects, resolved at use
  names <- split text on ','
  REQUIRE count(names) <= 4

FUNCTION decals_from(text) -> Wallmarks          # renderer-owned
  names <- split text on ','
  REQUIRE count(names) <= 4
  FOR EACH name IN names
    append a decal shader for name to the array
```

**Invariants** — the caps are asserted, not clamped: a library that exceeds them is
rejected rather than truncated. Four is the authored maximum for every kind; sound lists
are allowed two extra because the authoring format grew a few over-full entries that ship
in retail data, and rejecting them would make the retail library unloadable.

**Notes** — the multiplicity is the point of the whole arrangement. Every consumer picks
uniformly at random from the list, so a footstep on gravel is one of several recordings
rather than the same one repeated. Footsteps go further and refuse to repeat: the step
player remembers which entry it used last and, while the pair has not changed, advances to
a different one. That anti-repetition rule lives with the consumer, not here; what this
file guarantees is that the list is short, indexable and in authored order.

Particle effects are kept as names and resolved when one is spawned, because a particle
effect is cheap to name and expensive to instantiate, and most pairs never collide.
Sounds and decals are acquired eagerly, because a sound acquired during a collision
callback would stall the physics step and a decal shader is a renderer resource that
cannot be created off the render thread.

The decal array is not a plain list but an object the renderer creates, because the
renderer needs to hand back a shader without allocating at the call site — the decal is
chosen inside the frame, once per bullet hole.

## Pair teardown

**Contract** — releases every sound handle the pair holds, in the three lists that hold
them. Particle names and the decal array need no explicit release. Must run while the
audio device is still alive, which is why unloading the library is an explicit step in
the game module's shutdown rather than something that happens when the process exits.

**Notes** — the teardown of the particle-name list is a no-op that exists so that all
four kinds are released through the same shape; a rebuild deletes it. What does not
delete is the ordering constraint above: audio handles outliving their device is the
failure this explicit release prevents.
