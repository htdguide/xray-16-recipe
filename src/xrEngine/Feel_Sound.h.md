# src/xrEngine/Feel_Sound.h

> The interface by which an entity is told that a sound it could hear has been emitted.

**Needs** — [`xrSound/Sound.h`](../xrSound/Sound.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`IGame_Level.cpp`](IGame_Level.cpp.md) · [`CustomMonster.h`](../xrGame/CustomMonster.h.md) · [`base_client_classes_script.cpp`](../xrGame/base_client_classes_script.cpp.md)
**Tier floor** — T3: one callback signature.

## Purpose

Sound in this engine is a perception channel as well as a playback channel. Every emitted sound carries, besides its audio, a *type*, a *power* and a payload naming its source; the audio layer distributes those to every entity that registered as a listener within range. That is how a creature hears a gunshot behind it, and it is why the sound library is below the AI in the build order and not beside it.

This header is one third of the `feel` family — touch, vision and sound — each of which is a small interface an entity inherits to be fed one kind of perception.

## `Sound`

**Contract** — One entry point, invoked by the audio layer when a sound reaches this listener: who emitted it, what kind of sound it is, an opaque payload the game attaches to the emitter, the world position it was emitted from, and its perceived power at this listener. The default does nothing, so inheriting the interface without implementing it is legal and means "register me but ignore it".

**Notes** — Position and power are what the audio layer computed, not what the emitter authored: the power has already been attenuated by distance and the position is the emission point. The listener therefore does not need the audio model, only the result. The sound's *type* and the payload are pure game vocabulary and the engine never interprets them.

**Notes** — This interface has no registration function of its own. Registration happens in the audio layer; an entity becomes a listener by being added there. That split is why this file is three lines and why a rebuild might reasonably merge it into the audio module's own interface set.
