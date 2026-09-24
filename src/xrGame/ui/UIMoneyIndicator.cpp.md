# src/xrGame/ui/UIMoneyIndicator.cpp

> The multiplayer money readout: a persistent total beside a change notice that fades itself
> out, over a short list of the awards that caused it.

**Needs** — [`UIMoneyIndicator.h`](UIMoneyIndicator.h.md) · [`UIGameLog.h`](UIGameLog.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md)
**Used by** — [`UIMoneyIndicator.h`](UIMoneyIndicator.h.md)
**Tier floor** — T3.

## Purpose

A thin panel with one decision in it: the *change* is a separate widget from the *total*, and
it is animated rather than cleared. A player who earns money sees the delta appear and fade
while the total stays put.

## State

```text
RECORD MoneyReadout EXTENDS Window
  back         : Static     # element "money_wnd:money_indicator"
  total        : Static     # element "…:money_indicator:total_money"; a child of the plate
  change       : Static     # element "money_wnd:money_change"; starts invisible, fades
  bonus_list   : GameLog    # element "money_wnd:money_bonus_list"
```

**Invariants** — the total is a child of the backing plate while the change notice and the
bonus list are siblings of it. So moving the plate moves the total and nothing else, which is
what the shipped layout wants.

## `InitFromXML`

**Contract** — build from the `money_wnd` element tree, declining if the document has no such
element. Configure the four widgets and the bonus list's font, hide the change notice, and arm
it with a named alpha-and-text-colour animation.

**Notes** — the decline is the feature: a game kind whose overlay document omits the money
panel gets no money readout and needs no flag anywhere else. The change notice is *armed* at
construction and *restarted* at each use, so the animation's definition lives in data and the
code only re-triggers it.

## `SetMoneyChange`

**Contract** — set the change text and restart its fade from the beginning.

**Notes** — restarting rather than showing is what makes a second award while the first is
still fading read as one continuous notice rather than two overlapping ones. The widget's
visibility is never set again after construction hid it: the animation drives the alpha, so a
"hidden" widget with a running animation is exactly a visible one, and an idle one is
invisible. A rebuild must not "fix" the initial hide.

## `SetMoneyAmount` and `AddBonusMoney`

**Contract** — pass straight through: the total's text, and one structured award appended to
the bonus list. The caller formats the number; this file does not know the currency.
