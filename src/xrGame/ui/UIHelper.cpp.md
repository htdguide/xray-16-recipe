# src/xrGame/ui/UIHelper.cpp

> One shape, repeated sixteen times: construct a widget of a known type, configure it from a named layout element, adopt it into a parent — and, when the element is declared optional, return nothing instead of failing.

**Needs** — [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) · [`xrUICore/XML/UIXmlInitBase.h`](../../xrUICore/XML/UIXmlInitBase.h.md)
**Used by** — [`UIHelper.h`](UIHelper.h.md)
**Tier floor** — T3.

## Purpose

Chapter 15's layout reader takes *a widget the caller already constructed* and configures it.
That is the right factoring — it is what makes the element vocabulary closed — but it makes
every screen in this chapter write the same four lines per element. This file is those four
lines, once per control type.

Its one real decision is **optionality**, and it is the reason the file is worth a page
rather than a sentence.

## The shape

```text
FUNCTION create(kind, document, element_path, index, parent, critical) -> optional<Widget>
  IF NOT critical AND the document has no element at (path, index)
    THEN RETURN none                       # the element is legitimately absent
  widget := a new widget of that kind
  IF configure(document, path, index, widget, critical) failed AND NOT critical
    THEN destroy widget                    # the element exists but is malformed
  IF widget exists AND parent given
    parent.adopt(widget); widget is owned by the parent
  RETURN widget
```

**Invariants**

- **Critical is the default.** A screen asks for an element and gets it or the game stops.
  Optionality is opt-in, per element, at the call site.
- A widget returned with a parent is owned by that parent and must not be deleted by the
  caller. A widget returned without a parent is the caller's.
- The element *path* is used as the widget's debug name, and for indexed forms the index is
  appended. Widget names are therefore automatically distinct and automatically meaningful in
  the inspector, at no cost to the screen.

## Why optionality is the point

Three games' worth of UI data are loaded by one executable, and a fourth generation of it is
loaded from mods. A screen that is shared between them needs elements that exist in one
game's layout and not another's — the faction-war page's two alternative backgrounds, the
PDA's per-game panels, every element added after a game shipped.

Without the optional form each such element needs its own presence check at the call site,
and the shipped code would be a thicket of them. With it, a screen reads as a flat list of
elements with a flag on the ones that may be missing, which is what every screen in this
chapter actually looks like.

The two-stage test — *element absent* versus *element present but rejected by the reader* —
is deliberate: a missing element is normal, a malformed one is a data error, and only the
first is silent even in the optional case.

## Exported units

One creator per control type. Each is the shape above; nothing else is worth saying about
them individually.

| Creator | Notes |
|---|---|
| plain window | the only one whose type has no configuration beyond the window attributes |
| picture/label | indexed form available |
| scroll view, edit box, list box, check button | — |
| progress bar, progress shape | **the optional flag is not passed to the reader** — see below |
| frame line (indexed), frame window | — |
| three-state button | indexed form available |
| tooltip | **not adopted**: it is returned parentless and self-deleting, because a tooltip is positioned in screen space and must not be clipped by a parent |
| cell board, quick-use bar | **the optional flag is not passed to the reader** — see below |

**Notes** — four of the creators accept a `critical` flag, honour it for the *presence*
test, and then call a reader that has no such parameter. A malformed — as opposed to absent —
progress bar, progress shape or cell board therefore faults even when the caller declared it
optional. The source marks this as known. A rebuild gives every reader the same two-valued
contract and the inconsistency disappears.

The indexed creators exist for the repeated-element idiom described in
[`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md): the layout vocabulary has no repetition, so
N instances of one element are read by index.
