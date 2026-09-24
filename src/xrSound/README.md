# src/xrSound — 3D audio

Chapter 11 of the build order.

## What this module is responsible for

This module turns "play this sound here" into audio coming out of a speaker, and into a stimulus a
creature can react to. It owns four things: the **emitter** model and its lifetime; the **ranking**
that decides which handful of emitters get the scarce hardware voices; the **streaming** pipeline
that keeps those voices fed from compressed assets on disk; and the **environment** model —
geometry-driven reverb regions and raycast occlusion — that makes a sound belong to the place it is
played in.

What it deliberately does *not* own is the mixing. Positional attenuation, panning, per-source gain
and reverb are the audio device's job, bought behind [Seam: Audio
device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device); decoding is bought behind [Seam: Audio and
video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs). Almost every decision in
this chapter is therefore **policy**, and policy is what a rebuild must reproduce: which sounds are
audible, how loud, how they fade in and out, what a room does to them, and what an NPC hears.

## Where it sits

It rests on the core (virtual filesystem, strings, threading, the task scheduler), on the math
layer, and — less obviously — on **[`xrCDB`](../xrCDB/README.md), the static collision database**,
because both occlusion and reverb regions are ray queries against level geometry. It is built
before the engine and the game, and is reached through one interface, so a build with sound
disabled fills that interface with nothing and nothing above notices.

## The load-bearing ideas

Six ideas, named here once so the twins can be terse.

**1. Playing and being heard are different states.** An emitter exists from the moment the game asks
for a sound until the sound's natural end, whether or not it is ever audible. When it loses its
voice it *simulates*: its clock and its stream cursor keep advancing in silence, and it can be
promoted back mid-sound because the cursor is recomputed from wall-clock time, not from bytes
decoded. This is what lets a looping ambient bed survive a firefight without restarting, and it is
the single behaviour whose absence a player would notice immediately.

**2. Voices are a fixed, small pool and are ranked every frame.** Thirty-two by default, allocated
once at startup, never grown. Every emitter computes one number — smoothed volume × distance
attenuation × an importance scale — and a voice carries the number of whoever holds it. A free
voice carries −1, below every real rank, so "is a voice free" and "is the weakest voice weak
enough" are the same comparison. Taking a voice *demotes* its previous holder rather than stopping
it.

**3. Four independent smoothings stop the ranking from flickering.** A one-pole filter on volume
(≈10% per frame) so rank moves slowly; a linear fade ramp (full travel in 0.1 s) so a voice never
starts or stops on a discontinuity; an approach on occlusion (1.0 per second) so a doorway does not
snap; and a strictly-less-than comparison so two equal emitters do not trade a voice every frame.
A rebuild that compares instantaneous volumes will hear the difference as chatter at the edge of
audibility.

**4. Streaming is a ring of tenth-of-a-second blocks with two depths.** Ten blocks decoded ahead on
a worker thread (one second of lookahead, so a burst of new sounds cannot starve the old ones);
four blocks queued at the device (400 ms of slack, so a frame hitch is inaudible). A starved stream
is not a dropped sound: the device runs dry and stops, the next frame notices a stopped voice with
buffers still queued, and restarts it.

**5. The environment is authored as floors, and the listener's region is one downward ray.** The
level ships a mesh whose triangles each name two reverb presets — one for the side above, one for
below. "Which room am I in" is a ray straight down, and the dot of the ray against the hit
triangle's normal picks the side. Crossing a boundary blends the two presets linearly toward the
new one. Reverb is **global to the listener**, not per-source: one preset for the whole mix.

**6. Occlusion is one dithered ray per audible emitter, against two databases.** The ray is aimed
at a random point on a 0.2 m sphere around the source rather than at the source, so an edge is
crossed stochastically and the occlusion smoothing turns that into a gradient. The level's
collision model gives a binary answer scaled by one global constant; a dedicated **sound occlusion
mesh** gives a graded per-face answer, and the two compose multiplicatively. Each emitter caches the
triangle that blocked it last frame and tests that single triangle before querying the tree — which
is what makes the whole thing affordable.

### The sidecar every sound asset carries

Frozen game data, and the reason the AI can hear. Inside each audio file's comment field sits a
small binary record, currently at version 3:

| field | meaning |
|---|---|
| `version` | 1, 2 or 3; earlier versions omit later fields and default them |
| `min_distance` | full volume inside this radius |
| `max_distance` | inaudible to the **player** beyond this |
| `base_volume` | the authored level for this asset (version 2+) |
| `game_type` | AI classification: a bit set, an actor class OR'd with an event class |
| `max_ai_distance` | how far this sound carries to a **creature** (version 3) |

The last two are the AI-perception attributes. `max_ai_distance` exists separately from
`max_distance` because how far a sound carries as a *stimulus* is a design decision independent of
how far the player hears it — a silenced pistol is audible at thirty metres and perceptible at five.
`game_type` is what makes a sound *mean* something: a weapon shooting, a monster stepping, an item
being picked up. A playing emitter announces itself to the scene's hearing queue roughly twice a
second (with ±30 ms of jitter, so a firefight does not deliver every event in the same frame), at a
range scaled by its instance volume but **not** by occlusion — the AI hears through walls that the
player does not.

### The three-way split

The chapter is deliberately cut into an abstract interface, a target-agnostic core, and a
device-specific backend:

- [`Sound.h`](Sound.h.md) — everything the rest of the engine sees: a handle, a scene, a manager,
  a parameter record. No mixer appears in it.
- `SoundRender_Core*`, `SoundRender_Emitter*`, `SoundRender_Scene`, `SoundRender_Source`,
  `SoundRender_Environment`, `SoundRender_Target` — the policy, identical under any mixer.
- `SoundRender_CoreA`, `SoundRender_TargetA`, `SoundRender_EffectsA_EAX`, `OpenALDeviceList` — the
  backend. **Four files.** A rebuild changing mixers replaces exactly these.

## The twins

| file | role |
|---|---|
| [`Sound.h`](Sound.h.md) | The public interface: handle, scene, manager, parameters. Substantive. |
| [`Sound.cpp`](Sound.cpp.md) | Module lifetime and the reverb preset library. |
| [`SoundRender.h`](SoundRender.h.md) | The six constants the chapter's timing is built on. |
| [`SoundRender_Core.h`](SoundRender_Core.h.md) | Declares the target-agnostic manager. |
| [`SoundRender_Core.cpp`](SoundRender_Core.cpp.md) | Source cache, scenes, clocks, tunables, listener reverb blend. |
| [`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) | The update and render passes; the one-advance-per-frame epoch. |
| [`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md) | **Voice allocation** — the ranking test and the steal. |
| [`SoundRender_Core_SourceManager.cpp`](SoundRender_Core_SourceManager.cpp.md) | The shared-source cache and its never-evict policy. |
| [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) | Declares one playing instance. |
| [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) | **The state machine, the volume model, the ranking function.** |
| [`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md) | The streaming ring, the cursor and asset swap, the AI hearing event. |
| [`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md) | Lifetime: start, stop, rewind, nested pause, eviction. |
| [`SoundRender_Scene.h`](SoundRender_Scene.h.md) | Declares one sound world. |
| [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md) | **Occlusion, reverb regions, the play entry points, the hearing queue.** |
| [`SoundRender_Source.h`](SoundRender_Source.h.md) | Declares an asset's description. |
| [`SoundRender_Source.cpp`](SoundRender_Source.cpp.md) | **The sidecar format**, asset resolution, and decode at an offset. |
| [`SoundRender_Environment.h`](SoundRender_Environment.h.md) | Declares a reverb preset and its library. |
| [`SoundRender_Environment.cpp`](SoundRender_Environment.cpp.md) | The twelve preset parameters, the blend, the frozen preset file. |
| [`SoundRender_Effects.h`](SoundRender_Effects.h.md) | The reverb seam: four operations. Substantive. |
| [`SoundRender_Target.h`](SoundRender_Target.h.md) | Declares a hardware voice. |
| [`SoundRender_Target.cpp`](SoundRender_Target.cpp.md) | The device-independent half of a voice; the rank = −1 convention. |
| [`SoundRender_CoreA.h`](SoundRender_CoreA.h.md) | Declares the device backend. |
| [`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md) | Open the device, size the voice pool, listener handedness. |
| [`SoundRender_TargetA.h`](SoundRender_TargetA.h.md) | Declares the mixer-backed voice. |
| [`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md) | Buffer queue servicing, underrun recovery, parameter push. |
| [`SoundRender_EffectsA_EAX.h`](SoundRender_EffectsA_EAX.h.md) | Declares the vendor reverb backend. |
| [`SoundRender_EffectsA_EAX.cpp`](SoundRender_EffectsA_EAX.cpp.md) | Capability probing by round-trip; the twelve-property push. |
| [`OpenALDeviceList.h`](OpenALDeviceList.h.md) | Declares an enumerated device and its capabilities. |
| [`OpenALDeviceList.cpp`](OpenALDeviceList.cpp.md) | Probe every device; choose the system default at its newest version. |
| [`MusicStream.h`](MusicStream.h.md) · [`MusicStream.cpp`](MusicStream.cpp.md) | Dead: a slot table over the pre-Vorbis music streamers. Not compiled. |
| [`xr_streamsnd.h`](xr_streamsnd.h.md) · [`xr_streamsnd.cpp`](xr_streamsnd.cpp.md) | Dead: the pre-Vorbis music streamer. Not compiled. |
| [`xr_cda.h`](xr_cda.h.md) · [`xr_cda.cpp`](xr_cda.cpp.md) | Dead: CD-audio playback. Not compiled. |
| [`stdafx.h`](stdafx.h.md) · [`stdafx.cpp`](stdafx.cpp.md) | Build scaffolding. Delete in a rebuild. |
| [`guids.cpp`](guids.cpp.md) | Build scaffolding: one unit emitting the reverb extension's identifiers. |

## Conformance

Nothing in the system requirements' acceptance list tests sound directly, which understates the
chapter: criterion 10 (every shipped script runs unmodified) depends on the sound interface's exact
surface, and criterion 12 (NPCs behave) depends on the hearing events being raised at the right
range with the right classification. Two behavioural checks worth adding to a rebuild:

- Stand a looping ambient sound next to a firefight. It must not restart when it regains a voice.
- Walk a listener slowly through a doorway between two authored reverb regions. The preset must
  cross over a fraction of a second, not snap, and the occlusion of a source behind the door frame
  must fade rather than step.

## What could not be recovered

- **Why the reverb blend uses the raw frame delta as its interpolation factor.** It makes the blend
  rate frame-rate dependent — at 120 fps a boundary crossing takes twice as many frames as at 60,
  but the same wall time only by coincidence. It is applied identically to the listener's blend and
  each emitter's. There is no comment and no tuning constant; it may simply be a long-standing
  approximation nobody revisited.
- **Why the AI announcement range is scaled by instance volume but not by occlusion**, when the
  occluded volume is computed and available two lines away. Either a deliberate "creatures hear
  through walls" rule or an oversight; the source says nothing.
- **The magnitude of the occlusion scale (0.5) and the cull volume (0.01)** are plausible but
  unjustified anywhere. They are console variables, so they were tuned by ear.
- **The `LEVEL_VERSION` constant** is declared and never read. Whatever it once gated now versions
  itself elsewhere.
- **Per-emitter reverb is computed and never used.** Each emitter maintains a blended preset for its
  own position, which the shipped backend — whose reverb is listener-global — cannot apply. Whether
  this is vestigial or preparatory is not recoverable.
- **The two-second slack added to a CD track's measured length** in the dead CD player, and the
  **88 KB / 44 KB buffer sizes** in the dead streamer, have no stated reason. Both files are
  excluded from the build, so it does not matter.
- **Whether the hardware-to-software device substitution is still needed.** The source's own comment
  says the problem was already ancient in 2000 and still observed in 2022, with no measurement
  behind either claim.
