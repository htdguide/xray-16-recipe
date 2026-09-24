# src/xrGame/inventory_upgrade_property_inline.h

> The read-only accessors of an upgrade property descriptor.

**Needs** — [`inventory_upgrade_property.h`](inventory_upgrade_property.h.md)
**Used by** — [`inventory_upgrade_property.h`](inventory_upgrade_property.h.md)
**Tier floor** — T3: field reads

## Purpose

Carries the property descriptor's accessors out of its declaration, following the
namespace's file convention. A rebuild folds them into the declaration.

## State

`Stateless.`

## Accessors of `Property`

**Contract** — `id` and `id_str` are the property's configuration section name, which is
also its identity in the manager's table; `name` is the already-localized display string;
`icon_name` and `icon_color` are the icon and its tint in the upgrade screen;
`functor_params` is the authored list of item parameter names this property describes.
