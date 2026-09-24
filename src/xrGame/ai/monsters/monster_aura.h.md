# src/xrGame/ai/monsters/monster_aura.h

> Declares one named aura: a proximity field around a creature that drives a screen effect, a looping sound and a Geiger-like detector tick on the player.

**Needs** — [`monster_aura.cpp`](monster_aura.cpp.md) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_aura.cpp`](monster_aura.cpp.md)
**Tier floor** — T2: holds sound handles whose release order matters

## Purpose

Declares the surface implemented in [`monster_aura.cpp`](monster_aura.cpp.md). A creature owns
one aura per distinct field it projects — a psi field, a radiation field, a fear field — each
constructed with a short name that becomes the prefix of every configuration key it reads, so
several auras can be authored in one section without collision.

## Exported units

- construction with a creature and a name — the name is the configuration key prefix and is
  copied into a fixed buffer.
- `load_from_ini` — reads the whole parameter set, prefix by prefix, all optional.
- `strength` — the field's power at the player's current distance.
- `update_schedule` — the per-tick step: keep the looping sound alive, attach or detach the
  screen effect.
- `play_detector_sound` — the discrete tick whose period shortens as the player closes.
- `on_monster_death` — stop both sounds.

Teardown detaches the screen effect from the player.
