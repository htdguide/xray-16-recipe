# src/xrGame/ui/UIInvUpgrade.cpp

> One node of the upgrade tree drawn as four stacked layers, whose appearance is a pure function of two independent state machines — what the game says about this upgrade, and what the pointer is doing to it.

**Needs** — [`UIInvUpgrade.h`](UIInvUpgrade.h.md) · [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`inventory_upgrade.h`](../inventory_upgrade.h.md) · [`inventory_upgrade_manager.h`](../inventory_upgrade_manager.h.md) · [`alife_simulator.h`](../alife_simulator.h.md)
**Used by** — [`UIInvUpgrade.h`](UIInvUpgrade.h.md)
**Tier floor** — T3.

## Purpose

The upgrade bench shows a tree of upgrades for one item. This file is one node of it. The
node is not a button: it is a **stack of four pictures whose textures are chosen by a
computed view state**, and the view state is the combination of a *verdict* the game returns
about this upgrade and a *pointer state* the widget tracks itself.

Keeping those two separate is the file's whole structure, and collapsing them — the obvious
simplification — is what makes an upgrade that cannot be bought stop showing its hover
highlight.

## State

```text
RECORD UpgradeNode EXTENDS Window
  upgrade_id   : text                 # names an upgrade in the game's upgrade registry
  scheme_index : (column, row)        # where in the authored tree this node sits
  view_state   : ViewState
  prev_state   : ViewState            # so textures are re-chosen only on a change
  button_state : { free, focused, pressed, double_pressed }
  state_locked : bool                 # the verdict forbids pointer state from changing view
  layers : item picture, colour picture, optional border, optional ink, optional point

ENUM ViewState
  enabled                  # may be installed now
  focused                  # ... and the pointer is over it
  touched                  # ... and the pointer is pressed on it
  selected                 # already installed
  unknown                  # the player has not learned this upgrade exists
  disabled_parent          # a prerequisite upgrade is missing
  disabled_group           # a mutually exclusive sibling is installed
  disabled_precond_money   # the player cannot pay
  disabled_precond_quest   # a story condition is unmet
  disabled_focused         # any disabled state, with the pointer over it
```

**Invariants**

- The `selected`, `unknown` and `disabled_group` verdicts **lock** the view state: the
  pointer cannot change how they look, because there is nothing the player can do to them.
  Every other verdict leaves the node responsive.
- Textures are re-chosen **only when the view state changes**. Every node in a tree updates
  every frame and most of them change nothing.
- The node draws nothing unless its upgrade resolves in the registry.

## The layers

**Contract** — five named layers, of which two are always present:

| Layer | What it carries |
|---|---|
| item | the upgrade's own icon |
| colour | the state tint, a texture chosen by view state |
| border | an optional frame around the cell |
| ink | an optional fill shown while the node's hierarchy is highlighted |
| point | an optional marker at an authored offset inside the cell |

**Notes** — the cell may be authored *with* a border or without, and the two shapes lay out
differently: with a border the colour layer covers the whole cell, without one it is a
narrow strip inset by a few units — the layout's two visual idioms for "this node's state".
The inset differs between 4:3 and widescreen by one unit, which is the non-uniform canvas
stretch showing up yet again, here as a hand-tuned nudge rather than a factor.

The strip's size — five by thirty-eight units — and the point marker's size are literal in
code rather than authored. Recorded as unrecovered: they match the shipped textures and
nothing explains the choice.

## The two state machines

**Contract** — the verdict is recomputed whenever the item changes; the pointer state every
frame; the view state is derived from both.

```text
FUNCTION apply_verdict(item)
  result := the upgrade registry's "can this be installed on this item" answer
  dim the item icon                          # the default
  CASE result OF
    ok                  -> icon full bright; view := enabled;                lock := false
    unknown             ->                   view := unknown;                lock := true
    already installed   -> icon full bright; view := selected;               lock := true
    missing prerequisite->                   view := disabled_parent;        lock := false
    exclusive sibling   -> icon full bright; view := disabled_group;         lock := true
    cannot afford       ->                   view := disabled_precond_money; lock := false
    story condition     ->                   view := disabled_precond_quest; lock := false

FUNCTION derive_view_state()
  IF the pointer is over this node or its point marker
    THEN IF button_state is not pressed THEN button_state := focused
         AND tell the window to show this upgrade's description
    ELSE button_state := free
  IF state_locked THEN RETURN
  CASE button_state OF
    free           -> view := enabled if view was enabled/focused, else disabled_focused
    focused        -> view := focused if view was enabled/focused, else disabled_focused
    pressed/double -> view := touched if view was enabled/focused   # otherwise unchanged
```

**Notes** — three things a rebuild will get wrong.

*The icon's brightness is set by the verdict, not by the view state.* Three verdicts brighten
it — installable, installed, and blocked by an exclusive sibling — and the rest dim it. The
sibling case is brightened because the player *could* have had it and chose otherwise; it is
information, not absence.

*A disabled node still has a hover state*, `disabled_focused`, distinct from its verdict.
That is why the pointer state cannot simply be folded into the verdict: the tree must show
that a blocked node is under the pointer so its description appears.

*The pressed state is not cleared by a release on the node.* It is cleared by the pointer
leaving, or by a release anywhere. Pressing and dragging off therefore leaves the node in its
pressed appearance until the button comes up, which is the shipped behaviour.

## Interaction

**Contract** — a left press asks the parent window to confirm installing this upgrade, with a
localized prompt naming it; a right press and a double click do nothing but clear the
description and set the button state. Any of the three re-highlights the node's hierarchy.
Losing pointer focus clears the description and the highlight.

**Notes** — **the node does not install anything.** It asks the window, which asks the
player, which asks the game. Three hops for one click, and the reason is that the install is a
transaction with a price: the confirmation must be able to fail, and the node has no standing
to spend the player's money. This is the chapter's event-not-call rule in its clearest form.

Hovering highlights the node's whole **hierarchy** — its prerequisites and its dependants —
by asking the window to mark them, and the ink layer of every marked node becomes visible.
The highlight is computed by the window because it needs the tree; a node knows only itself.

## The point marker

**Contract** — a small picture placed at an authored offset inside the cell, shown only while
the node's hierarchy is highlighted, and forwarding all of its pointer events to the node it
belongs to. Its horizontal offset is scaled by 0.8 and its width changes from 14 to 11 units
on a widescreen display.

**Notes** — the marker is a second, smaller hit target for the same node, which is how the
connecting lines drawn between tree nodes become clickable without being widgets. Its
existence is why every "is the pointer over this node" test in the file is really "over the
node **or** its marker".

**The marker never records which node it belongs to.** Its constructor takes the owning node,
asserts it is not empty, and discards it; every one of its handlers then reaches through an
unset reference. Any pointer interaction with a marker is therefore a fault. Since the shipped
games' upgrade layouts define markers, this is a live defect rather than a dormant one — a
rebuild must simply store the reference the constructor is given.
