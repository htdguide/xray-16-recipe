# src/xrGame/CustomDetector.cpp

> The artefact detector: a held item that senses nearby artefacts through the touch sense, drives a small readout rendered on its own model, and hides itself automatically whenever the other hand needs to do something incompatible.

**Needs** — [`CustomDetector.h`](CustomDetector.h.md) · [`ui/ArtefactDetectorUI.h`](ui/ArtefactDetectorUI.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Weapon.h`](Weapon.h.md) · [`player_hud.h`](player_hud.h.md) · [`Artefact.h`](Artefact.h.md) · [`HUDManager.h`](HUDManager.h.md)
**Used by** — reached through its declarations in [`CustomDetector.h`](CustomDetector.h.md); callers name that, not this file.
**Tier floor** — T2: proximity queries and a small state machine over the held-item lifecycle

## Purpose

A detector is a held item with an unusual property: it occupies the *left* hand, alongside
whatever the right hand is holding, and it therefore has to negotiate with the other item
continuously. Most of this file is that negotiation. The sensing itself is three lines —
ask the touch sense which artefacts are inside a radius — and everything else answers
"may the detector be visible right now, and if not, what should happen instead".

The negotiation rule is the load-bearing decision: a detector may be out only while the
right hand holds nothing demanding — a pistol, a knife, a bolt, or nothing. A rifle, a
reload, a weapon being raised to the eye or swapped: the detector holsters itself, fast,
and *remembers that it wants to come back*. When the obstruction clears it returns on its
own. The player never puts a detector away; the game does it for them.

Two other decisions matter:

- **artefacts are found by the touch sense, not by a query.** The detector registers a
  sphere with the senses system and is told when something enters or leaves it, so the
  cost is paid by the sense system's own spatial structure and the detector keeps a stable
  per-artefact record it can attach a sound timer and a particle effect to.
- **a detector has a rank and refuses to see above it.** Each detector model declares which
  artefact ranks it can perceive; an artefact above that rank is not merely undrawn, it is
  never admitted to the contact set. This is what makes the better detector a real upgrade
  rather than a cosmetic one.

## State

```text
RECORD Detector
  working          : bool     # sensing and drawing
  need_activation  : bool     # was forced away; wants to return when the obstruction clears
  fast_anim_mode   : bool     # use the abbreviated draw/holster animation
  detect_radius    : real     # sensing sphere, default 30
  vis_radius       : real     # inside this, an invisible artefact is forced visible, default 2
  decay_rate       : real     # condition consumed per draw, default 0 (no wear)
  artefacts        : DetectList<Artefact>   # see CustomDetector.h
  readout          : optional<DetectorUI>   # created on first activation, owned for life
```

Invariants: `working` and the readout's existence are coupled — turning the detector on
creates the readout if there is none, and the readout is never destroyed until the item
is. A detector that is not the current view entity's child never senses, so a detector in
a non-player character's inventory costs nothing.

## The held-item state machine

**Contract** — a detector moves through hidden, showing, idle and hiding, driven by
animation completion rather than by timers. The order is load-bearing:

```text
hidden  --ToggleDetector--> showing : attach to the first-person rig, play the draw
                                      sound and animation, mark the item busy,
                                      turn sensing ON
showing --animation ends--> idle    : play the loop animation, mark the item free,
                                      charge the draw against the item's condition
idle    --ToggleDetector--> hiding  : play the holster sound and animation, mark busy
hiding  --animation ends--> hidden  : turn sensing OFF, detach from the rig
```

Two asymmetries are deliberate. Sensing starts when the draw *begins* and stops when the
holster *ends*, so the readout is live for the whole time the device is on screen.
Condition is charged when the draw *completes*, so an interrupted draw is free — which
matters because the automatic hide/show cycle can draw and holster the device many times
during a firefight.

## `ToggleDetector`

**Contract** — the single entry point for both directions. From hidden it checks whether
the currently active item permits a detector; from idle it starts the holster. From any
other state — mid-animation — it does nothing, which is what makes the automatic cycle
safe to call every frame.

The interesting branch is what happens when the active item is *not* compatible but
something compatible could be activated instead: rather than refusing, the detector
switches the other hand to an acceptable item and sets its pending flag, so the draw
happens on a later frame once the swap completes.

```text
FUNCTION toggle(fast)
  need_activation = false
  fast_anim_mode  = fast

  IF state is hidden THEN
    active = the inventory's currently active item
    IF compatible(active, and find a fallback slot) THEN
      IF a fallback slot was chosen THEN
        inventory.activate(fallback slot)
        need_activation = true            # try again once the swap lands
      ELSE
        switch to showing
        turn sensing on
  ELSE IF state is idle THEN
    switch to hiding
```

## `CheckCompatibilityInt` — the negotiation rule

**Contract** — answers whether a given held item permits the detector to be out, and
optionally chooses a slot to switch the other hand to if it does not. Holding nothing is
always compatible.

```text
FUNCTION compatible(item, out fallback_slot) -> bool
  IF item is none THEN RETURN true

  ok = item's home slot is the sidearm, the knife, or the bolt

  IF NOT ok AND a fallback was requested THEN
    # prefer, in ascending order of preference: bolt, knife, then anything
    # currently occupying a slot that is not its own home
    fallback = none
    IF the bolt slot is occupied            THEN fallback = bolt slot
    IF the knife slot is occupied           THEN fallback = knife slot
    IF slot 3 holds an item whose home is not slot 3 THEN fallback = slot 3
    IF slot 2 holds an item whose home is not slot 3 THEN fallback = slot 2
    ok = fallback is not none

  # an item in the middle of doing something cannot be interrupted,
  # unless it is itself still being drawn
  IF item is not currently being drawn THEN ok = ok AND item is not busy

  IF ok AND item is a weapon THEN
    ok = ok AND weapon is not idling-with-flourish
             AND not reloading
             AND not switching fire mode
             AND not aimed down sights

  RETURN ok
```

**Notes** — the last two clauses of the fallback search test for an item sitting in a slot
that is not its home. That is how a weapon *already holstered into the pistol slot* is
recognized as something the player is willing to have in hand. The overlapping conditions
(slot 2 tested against slot 3's home) look like a transcription slip in the original and
are preserved because changing them changes which weapon the game picks for you.

## `CheckCompatibility`

**Contract** — the outward-facing form, asked by the item system before allowing another
item to be activated. Delegates to the base and then to the rule above; when the rule
says no, it *also* holsters the detector on the spot and answers no. So a single call both
answers the question and performs the consequence.

## `UpdateVisibility` — the automatic cycle

**Contract** — run every frame while the detector's owner is the view entity. Decides in
both directions: hide when something incompatible starts, and show again when it stops.
Climbing a ladder counts as incompatible, separately from anything held.

```text
FUNCTION update_visibility()
  IF a first-person item is attached AND this detector has rig data THEN
    IF the owner is climbing THEN
      hide(fast); need_activation = true
    ELSE IF the attached item is a weapon
            AND (it is aimed, reloading, or switching) THEN
      hide(fast); need_activation = true

  ELSE IF need_activation THEN
    IF the owner is not climbing
       AND (nothing is held OR the held item is compatible) THEN
      show(fast)
```

**Invariants** — the automatic path always uses the fast animation. The deliberate,
player-initiated path uses the full one. That is how the player can tell a forced holster
from their own.

**Notes** — the two branches are mutually exclusive on whether a first-person item is
attached at all, which is why a forced hide cannot immediately un-hide itself: during the
holster animation the detector is still attached, so only the first branch runs, and only
once the holster completes and detaches does the second become reachable.

## `shedule_Update`

**Contract** — the scheduled (rate-degraded) update. Does nothing unless the detector is
working. Snaps the item's position to its carrier's, then asks the touch sense to refresh
the contact set against a sphere of the detect radius. A detector whose condition has
fallen to nothing stops sensing but is otherwise still drawn — a broken detector shows an
empty readout rather than disappearing.

**Notes** — the position snap exists because a held item's transform is otherwise driven
by the first-person rig, which is a screen-space construct with no meaningful world
position. The sense sphere needs a world position, so the item borrows its carrier's.

## `UpdateCL`

**Contract** — the per-frame update. Returns immediately unless the carrier is the entity
the player is currently controlling, so detectors held by anyone else cost nothing. Then
runs the visibility negotiation and, if working, the readout refresh.

## `IsWorking`

**Contract** — true only when sensing is enabled *and* the carrier is the current view
entity. Both halves matter: the first is the device's own switch, the second scopes the
whole subsystem to the one character the player is looking through.

## `UpfateWork`

**Contract** — the per-frame readout refresh: let the subclass translate contacts into
drawable marks, then tick the readout widget. Split from `UpdateCL` so that a subclass can
extend what is drawn without re-deriving when it is drawn.

## `Load`

**Contract** — reads the sensing radius, the forced-visibility radius, the per-draw
condition cost, the detectable artefact classes and their audio, and the draw/holster
sounds. Fixes the animation slot this item occupies on the first-person rig.

**Notes** — every radius has a compiled-in default, so a detector section can be minimal.
The per-draw condition cost defaults to zero, meaning the shipped detectors do not wear
out unless a section says so.

## `OnH_B_Independent` / `OnMoveToRuck`

**Contract** — the two ways a detector leaves the player's hand. Dropping it into the
world clears the contact set and, if it was out, stops sensing and forces the state to
hidden without playing anything. Moving it to the backpack from a slot additionally
detaches it from the first-person rig and cancels the running animation *without* letting
the animation's completion callback fire — because that callback would otherwise advance
the state machine of an item that is no longer held.

**Invariants** — this is the conformance invariant about destroyed and removed entities
being unreferenced: the contact set holds pointers to artefacts, and a detector that
leaves the world with a populated contact set leaves the senses system holding it.

## `~CCustomDetector` / `net_Spawn`

**Contract** — destruction releases the detect list's audio, stops sensing and destroys the
readout. Spawning explicitly turns sensing *off* before the base spawns, so a detector
that appears in the world — dropped, or spawned into a container — is never born switched
on.

## `CAfList::feel_touch_contact` — the rank filter

**Contract** — the senses system's admission test for one candidate artefact. Admits it
only if its configuration section appears in this detector's table of detectable classes
*and* its rank does not exceed the detector's. Rejecting here rather than at draw time is
what makes rank a real capability: an artefact above the detector's rank never enters the
contact set, so it produces no sound, no mark and no forced visibility.

## `TurnDetectorInternal`

**Contract** — sets the working flag and lazily creates the readout on first switch-on.
The readout is never destroyed on switch-off, only on item destruction, so repeated
holster/draw cycles do not rebuild a widget tree.

## `UpdateNightVisionMode`

**Contract** — empty. The per-artefact record carries a particle effect field intended for
a night-vision display mode that is not implemented in this branch; the field is
constructed, destroyed and never set.

## Could not recover

- The per-contact record carries a sound timer, a current sensing period, a zone map
  location and a night-vision particle name. The detect list loads a frequency *range*
  per detectable class and a per-class sound set, and none of it is consulted by any
  detector here: no detector in this branch clicks. The data and the plumbing for a
  proximity-driven click rate are complete and unused, which suggests the feature was
  moved into the readout widgets and the audio path was never removed.
- `OnActiveItem`, `OnHiddenItem`, `OnH_A_Chield`, `OnMoveToSlot` and `UpdateXForm` are all
  overridden to do nothing beyond the base, or to do nothing at all. The first two
  deliberately suppress base behaviour — a detector is never the "active item" in the
  sense the base means — but nothing records why.
