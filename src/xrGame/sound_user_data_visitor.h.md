# src/xrGame/sound_user_data_visitor.h

> The interface a listener implements to interrogate the AI payload attached to a sound it has heard, without knowing what kind of payload it is.

**Needs** — _(none beyond the sound payload base type it visits)_
**Used by** — [`sound_memory_manager.cpp`](sound_memory_manager.cpp.md) · [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`stalker_sound_data.cpp`](stalker_sound_data.cpp.md) · [`stalker_sound_data_visitor.h`](stalker_sound_data_visitor.h.md)
**Tier floor** — T3: a two-case dispatch over payload kinds

## Purpose

When a sound reaches a creature's hearing sense, the creature needs to know more than
"a noise happened here". Speech carries who is talking and what they are saying; a
footstep does not. The payload attached to a playing sound is therefore polymorphic, and
this interface is how a listener reads it: the listener implements the cases it cares
about, hands itself to the payload, and the payload calls back the case that matches its
own kind.

Only two cases exist in this codebase: the generic payload, and the character-speech
payload. A rebuild in a language with pattern matching over a tagged union should do that
instead — the interface exists because C++ has no cheaper way to ask an object what it is.

## State

`Stateless.` — a visitor holds whatever the implementing listener holds; the interface
itself owns nothing.

## `CSound_UserDataVisitor`

**Contract** — declares one visit case per payload kind. **Every case has an empty default
implementation**, which is the load-bearing part: a listener that only cares about speech
implements only the speech case and is silently unaffected by the others. New payload
kinds may be added without touching existing listeners.

```text
INTERFACE sound_user_data_visitor
  visit(generic_payload)   # default: ignore
  visit(speech_payload)    # default: ignore
```

**Notes** — the do-nothing defaults are a deliberate policy, not laziness: perception is
event-driven and a listener that receives a payload it does not understand must carry on,
not fail. A rebuild that makes the cases mandatory will force every listener to enumerate
every payload kind, which is exactly the coupling this avoids.
