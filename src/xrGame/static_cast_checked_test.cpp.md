# src/xrGame/static_cast_checked_test.cpp

> Not built: a scratch file recording, by example, what the checked downcast must accept and
> what it must reject.

**Needs** — [`static_cast_checked.hpp`](static_cast_checked.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it is about a language's conversion rules.

## Purpose

This file is **not compiled into anything**. It is a set of worked examples left beside
[`static_cast_checked.hpp`](static_cast_checked.hpp.md), including two cases per hierarchy
that the author annotated as *deliberately failing to build* — and it is a specification, not
a test: nothing runs it and nothing reports on it.

What it specifies is a single property of the conversion, stated over two hierarchies (one
without runtime type information, one with, since the operation behaves differently in each):
the conversion must preserve read-only-ness. Converting a modifiable handle yields a
modifiable one; converting a read-only handle yields a read-only one; and attempting to
launder a read-only handle into a modifiable one must fail at build time rather than
silently succeeding.

The two failing lines in each hierarchy are the specification's teeth. They are present in
the source, so the file cannot ever have been part of the build — which is the fact worth
recording.

## State

Stateless.

## Notes

A rebuild reproduces this as an actual test in whatever its language offers, or discards it
entirely if the language's conversion rules make the property automatic. What must not be
discarded is the property itself: a downcast helper that loses read-only-ness is a hole in
every guarantee built on it.
