# src/xrGame/flare.h

> Declares the hand-held burning flare implemented in [`flare.cpp`](flare.cpp.md).

**Needs** — [`hud_item_object.h`](hud_item_object.h.md)
**Used by** — [`flare.cpp`](flare.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the flare: an inventory item whose whole behaviour is a timed light source
attached to the first-person hands. Substance is in [`flare.cpp`](flare.cpp.md).

The load-bearing content of the declaration is the five-state machine, because the states
are the item's entire contract with the animation layer and the order of transitions is what
the implementation enforces:

```text
ENUM FlareState
  hidden     # not in hand, no light, not being processed per frame
  showing    # raise animation playing; light already lit
  idle       # burning in hand; the only state in which the flare is "active"
  hiding     # lower animation playing
  dropping   # throw animation playing; ends by releasing the item into the world
```

Exported units:

- `CFlare` — the item. Holds the light-animation curve it reads colour from, the light it
  created in the renderer, the particle effect at its tip, and the burn duration in seconds.
- `Load` — reads the burn duration from the item's section.
- `net_Spawn` / `net_Destroy` — enter the hidden state with no particle effect.
- `OnStateSwitch` / `OnAnimationEnd` — the state machine: which animation each state plays,
  and what each animation's end transitions to.
- `UpdateCL` — the per-frame burn: brightness, condition, light and particle placement, the
  auto-drop two seconds before burnout, and extinction at burnout.
- `UpdateXForm` — deliberately does nothing; see the note in the implementation twin.
- `ActivateFlare` / `DropFlare` / `IsFlareActive` — the surface the actor's item handling
  drives it through.
- `SwitchOn` / `SwitchOff` — create and destroy the light and the particle effect.
- `FirePoint` / `ParticlesMatrix` — where the flame is, resolved differently depending on
  whether the item is drawn as a first-person model or as a world object.
