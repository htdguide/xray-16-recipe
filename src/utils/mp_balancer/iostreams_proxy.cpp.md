# src/utils/mp_balancer/iostreams_proxy.cpp

> Allocates the three console sinks declared next door, in the build configuration that does not ship.

**Needs** — [`iostreams_proxy.h`](iostreams_proxy.h.md) · [`pch.h`](pch.h.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T4.

## Purpose

Gives the substitute sinks of [`iostreams_proxy.h`](iostreams_proxy.h.md) somewhere to
live, and defines the line terminator as a carriage return followed by a line feed. Under
every configuration that actually builds, the whole file compiles to nothing.

A rebuild drops it.

## State

Stateless.
