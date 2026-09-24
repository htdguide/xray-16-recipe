# src/xrUICore/Callbacks — binding notifications to handlers

> A screen becomes a message target by mixing this in, and then says "when *that* widget sends
> *this* message, call *this*".

Part of [chapter 15](../README.md).

## What this directory is responsible for

The notification channel described in the [chapter opener](../README.md) delivers a
`(sender, message-id, payload)` triple to a window's message target. This directory is how a
target turns those triples into handler calls without writing a dispatch switch.

A screen mixes in the callback holder and registers bindings. A binding names the sender —
either a widget by identity or a widget by *name* — the message id, and the handler. Incoming
events are matched against the flat list and the **first match wins**.

## The load-bearing ideas

**Two ways to name a sender, and they are not equivalent.** By identity, the binding refers to
one widget the screen built. By name, it matches any sender whose name is the given one, which
is how a screen binds to widgets that the XML reader created and that the screen never held a
reference to. Names are not required to be unique, so a name binding can match more than one
widget — which is sometimes the point.

**First-wins matching is the dispatch rule.** Registration order is therefore significant, and
a later binding for the same (sender, message) pair is dead. A rebuild that uses a map keyed
by the pair changes behaviour for any screen that registered a duplicate.

**Handlers come in two shapes** — one that receives the sender and payload, one that receives
nothing — so a screen does not have to declare parameters it will not read.

## The twins

| Twin | Role |
|---|---|
| [`UIWndCallback.cpp`](UIWndCallback.cpp.md) | The flat binding list and its first-wins match on each incoming event |
| [`UIWndCallback.h`](UIWndCallback.h.md) | The mixin that turns a screen into a message target |
| [`callback_info.h`](callback_info.h.md) | The binding record and the predicate that matches an event against it |
