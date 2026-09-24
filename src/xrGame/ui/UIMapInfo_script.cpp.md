# src/xrGame/ui/UIMapInfo_script.cpp

> Exports the map description panel to the script layer under its own name, deriving from the
> base window type.

**Needs** — [`UIMapInfo.h`](UIMapInfo.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIMapInfo.h`](UIMapInfo.h.md)
**Tier floor** — T3.

## Purpose

A separate file for one reason: the script registration is compiled against the script
engine's own precompiled header, and the panel's implementation is not. The split is a build
artefact; the *contract* is that this panel is script-visible.

## `script_register`

**Contract** — register the panel as a script class named `CUIMapInfo`, derived from the base
window class, with a default constructor and two methods: the placement call exposed as `Init`
and the content rebuild exposed as `InitMap`. The long-description accessor is **not**
exported.

**Notes** — the exported method name `Init` differs from the internal one. Conformance
criterion 10 freezes the exported name, not the internal one, so a rebuild may name the
implementation anything and must export exactly these two names on this class.
