# src/xrGame/alife_registry_container_inline.h

> The type-keyed selector that picks one registry out of the bundle, checked at compile time.

**Needs** — [`alife_registry_container.h`](alife_registry_container.h.md)
**Used by** — [`alife_registry_container.h`](alife_registry_container.h.md)
**Tier floor** — T3: a lookup resolved before the program runs

## Purpose

Holds the two bodies (readable and writable) of the container's selector. Substance of the
container is in [`alife_registry_container.cpp`](alife_registry_container.cpp.md).

## The selector

**Contract** — given a registry type, yields that registry out of the container. Cannot
fail at run time: a type that is not in the registry list is rejected before the program
exists, with the diagnostic *there is no specified registry in the registry container*.

```text
FUNCTION select<RegistryType>() -> ref RegistryType
  STATIC REQUIRE RegistryType is a member of the registry list
  RETURN this viewed as RegistryType
```

**Notes** — this is the one place the "adding a registry means editing exactly one file"
property is enforced. The membership check is not defensive programming: without it, the
view would still compile for any base class the container happens to have, silently
handing back the wrong store. A rebuild using a type-indexed map gets the same guarantee
from the map's key type; a rebuild using named fields gets it for free and can delete this
page entirely.

The original's spelling — passing a null pointer of the desired type as the argument, so
that the argument's *type* selects the overload while its *value* is never read — is pure
C++ idiom with no counterpart worth preserving.
