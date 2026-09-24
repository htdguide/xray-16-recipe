# src/xrGame/cta_game_artefact.cpp

> The artefact that is the objective in capture-the-artefact: it refuses to be used by the wrong team and re-anchors itself at its home point when carried back to base.

**Needs** — [`cta_game_artefact.h`](cta_game_artefact.h.md) · [`cta_game_artefact_activation.h`](cta_game_artefact_activation.h.md) · [`Artefact.h`](Artefact.h.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`game_base.h`](game_base.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`cta_game_artefact.h`](cta_game_artefact.h.md); callers name that, not this file.
**Tier floor** — T2: a subclass whose decisions are game-rule tests; the only hard edge is the event packet it composes

## Purpose

In capture-the-artefact, two artefacts exist — one per team — and each is the other team's
prize. The base artefact class does not know about teams, so this subclass adds the three
rule tests the mode needs: *may I be activated at all right now*, *am I my own team's
artefact or the one I stole*, and *have I been brought home*.

The last of those is the scoring event, and it is detected here rather than by the game
rules because only the artefact knows its own position every frame.

## State

```text
RECORD CtaGameArtefact EXTENDS Artefact
  game            : optional<CaptureTheArtefactClient>  # the mode's client rules, or none in another mode
  artefact_rpoint : optional<point>                     # this artefact's home position, borrowed from the mode
  my_team         : Team                                # which team owns it; starts as "spectators" meaning unknown
```

Invariants:

- **`artefact_rpoint` and `my_team` are resolved together, lazily, and never separately.**
  Until the mode has told the client which artefact identifier belongs to which team, both
  are unknown; the per-frame update retries until they are known.
- **The home point is borrowed from the mode**, not copied. It is authored level data with
  the level's lifetime, and the artefact is not the right owner.
- **`game` being absent is legal.** The same class is instantiated when the object's class
  identifier appears in a level loaded under another mode; every rule test then falls
  through to permissive behaviour.

## `UpdateCLChild`

**Contract** — the per-frame hook the base artefact leaves for subclasses. Keeps a carried
artefact glued to its carrier, resolves the team and home point if they are not yet known,
and — the load-bearing part — detects arrival at the home point and settles the artefact
there.

```text
FUNCTION update_child()
  base.update_child()
  IF carried: transform = carrier.transform        # see Notes

  IF artefact_rpoint is none: initialize_rpoint()
  IF artefact_rpoint is none: RETURN               # the mode has not synchronised yet

  IF game exists
     AND transform.position is within game.base_radius of artefact_rpoint
     AND a physics body is active
    move_to(artefact_rpoint)                       # snap exactly onto the point
    deactivate_physics_body()                      # and stop simulating it
```

**Invariants** — after settling, the artefact is at the home point exactly and is no longer
physically simulated, so it cannot be nudged off the point by a grenade or a body.

**Notes** — the arrival test is a radius test against the *base radius*, a mode-wide tuning
value, not a per-artefact one. So "home" is the base, and the artefact teleports the last
short distance to a canonical point. That teleport is what makes the visual state
unambiguous for every client: an artefact at the base is at exactly one position, so no
client sees it resting at a slightly different angle.

Copying the carrier's transform wholesale — rotation included — rather than placing the
artefact in a hand bone is a deliberate simplification for this mode: the objective is
carried visibly at the carrier's origin, not held. It also means the artefact's transform is
never stale relative to its carrier, which matters because the arrival test reads it.

When the home point is not yet known the update simply returns, every frame, until the
mode's synchronisation message arrives. The artefact is inert but present in the meantime,
which is correct — it must not settle at a point it has not been told about.

## `InitializeArtefactRPoint`

**Contract** — private. Compares this artefact's entity identifier against the two the mode
publishes, and on a match borrows the matching team's home point and records the team.
Leaves both unset when the mode has not yet published them.

**Notes** — the artefact discovers its own team from the mode rather than being told at
spawn, because in multiplayer the spawn record arrives before the mode's team assignment
message. Identity by entity identifier is the only key available at that point.

## `Action`

**Contract** — intercepts the player's use command. Blocks activation — the artefact's
consume-itself behaviour — when the mode forbids activation at this moment, or when this is
**not** the player's own team's artefact. Everything else falls through to the base
artefact.

```text
FUNCTION action(command, flags) -> handled
  IF game exists AND command is fire AND flags say "pressed"
    IF NOT game.can_activate_artefact(): RETURN handled     # swallow, do nothing
    IF NOT is_my_team_artefact():        RETURN handled
  RETURN base.action(command, flags)
```

**Notes** — the rule reads backwards until you know the mode. You may destroy *your own*
team's artefact (the mode allows it as a tactical option, under its own timing rules) but
you may not destroy the one you stole, because that would be a way to deny the enemy their
objective without ever returning it. The blocked command is *swallowed*, not rejected: the
player gets no error, the weapon simply does not fire.

## `IsMyTeamArtefact`

**Contract** — private. Reads the carrier's player record from the mode, takes that player's
team, and compares this artefact's identifier against the mode's artefact identifier for
that team. Requires a carrier — it is only ever asked about a held artefact. Yields true
when there is no mode at all, which makes every rule test permissive outside the mode.

## `CreateArtefactActivation`

**Contract** — overrides how the base artefact begins consuming itself. On the authoritative
side only, it emits an ownership-rejection event naming the carrier, the artefact, and the
artefact's **home point** as the drop position, then moves the artefact there itself.

```text
FUNCTION create_artefact_activation()
  IF NOT authoritative: RETURN
  event = new event(OWNERSHIP_REJECT, destination = carrier.id)
  event.write_entity_id(self.id)
  event.write_byte(0)                       # see Notes
  event.write_vector(artefact_rpoint)
  send(event)
  move_to(artefact_rpoint)                  # so the server's own state matches what it just sent
```

**Notes** — activating the artefact in this mode does not spawn an anomaly where it is used,
as it would in the single-player game; it *returns the artefact to its base*. The base
class's activation path is therefore repurposed into a teleport, and the anomaly-spawning
half is suppressed by the activation subclass
([`cta_game_artefact_activation.cpp`](cta_game_artefact_activation.cpp.md)).

The zero byte after the artefact identifier is a flag in the ownership-rejection event's
frozen layout, and this call site always sends it clear. The source gives it no name here. A
rebuild must emit the byte to keep the wire format, and can discover its meaning only from
the event's reader.

The server moves itself *after* sending, with a comment noting it is for the server's own
subsequent import of physics state. The ordering is not load-bearing — no reader observes
the intermediate state — but the pairing is: the authoritative side must apply what it
broadcasts, or its next physics export contradicts the event it just sent.

## `OnAnimationEnd`

**Contract** — guards the base class's animation-completion handling against a carrier that
has gone away mid-animation, logging and returning instead. This happens when the carrier is
killed during the activation animation, which is a legal and common race in multiplayer.

## `CanTake` / `PH_A_CrPr` / `OnStateSwitch`

**Contract** — `CanTake` is the base class's answer unchanged. `OnStateSwitch` is the base
class's behaviour unchanged. `PH_A_CrPr`, the pre-physics-step hook, is deliberately empty.

**Notes** — all three are overrides that add nothing, and two of them wrap substantial
disabled code. `OnStateSwitch` once short-circuited the activating state by firing the
animation-end callback immediately, because no first-person animation for artefact
activation existed; that shortcut is commented out, so the mode now relies on the base
class's state machine and whatever animation the data supplies. `PH_A_CrPr` once fixed the
artefact's physics body in place on the first frame after spawn and told it to ignore static
geometry; that too is disabled, which is why the settling logic in the per-frame update has
to deactivate the physics body itself.

A rebuild should implement the *live* behaviour and treat these three as places where the
original left a door open. The disabled network-export body in the same file is a fourth:
it is a complete quantised physics-state serialiser (position, normalised quaternion at
eight bits per component, angular velocity clamped to ten turns per second, linear velocity
clamped to plus or minus thirty-two units per second, with null-velocity flags) that the
mode ended up not using, falling back to the base inventory item's export. It is recorded
here because those clamp ranges are the same ones the live inventory-item export uses, and
seeing them written out is the clearest statement of that wire format in the codebase.
