# src/xrCore/Threading/ThreadUtil.cpp

> Gives a thread a name a debugger will show, and maps the engine's priority names onto whatever the host calls them.

**Needs** — [`ThreadUtil.h`](ThreadUtil.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Profiler and GPU debugging](../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — [`ThreadUtil.h`](ThreadUtil.h.md)
**Tier floor** — T3 in substance — every function is a call into the host — with one T1 corner where a thread name is delivered by raising a debugger-specific exception.

## Purpose

Thread naming and priority, per platform. Naming is not cosmetic here: the engine runs a dozen long-lived threads plus a worker per core, and a crash report or a profile capture that lists them all as anonymous is nearly useless. Priority exists mostly so the engine can *read* it; almost nothing sets it.

## `SetCurrentThreadName`

**Contract** — names the calling thread. Never fails in a way the caller must handle; a platform that refuses gets a log line. Also forwards the name to the profiler when one is compiled in, so the two views agree.

```text
FUNCTION set_current_thread_name(name)
  full <- "X-Ray " + name                  # prefix, so engine threads are greppable
                                           #   among the host's and the drivers'
  IF the host has a thread-naming call
    use it
  ELSE
    fall back to the debugger-notification convention:
      raise a specific exception carrying a (type, name, thread, flags) record
      and immediately continue execution — the debugger reads it in passing,
      and with no debugger attached nothing happens
  forward `name` to the profiler, if present
```

**Notes** — the fallback is the genuinely strange part and is worth understanding rather than copying: on that platform, older runtimes had no naming call, so the convention was to *throw* a magic exception whose payload the debugger inspects. The handler continues execution at the faulting instruction, so the raise is a no-op when nobody is listening. A rebuild targeting only modern hosts uses the direct call and deletes this.

The name length is bounded by the buffer it is composed into; a longer name is truncated silently.

Some hosts limit a thread name to a short length and will simply refuse a longer one, which is why the failure path logs instead of asserting.

## Priority

**Contract** — four functions read and write the calling thread's priority level and the process's priority class, translating between the engine's seven- and six-value vocabularies and the host's. On hosts where the concept does not map, the readers report *normal* and the writers do nothing.

**Notes** — the thread-priority *setter* on the one platform that implements it ignores its argument and always sets time-critical. That is a bug, not a decision: it computes the correct value into a local and then passes a constant. A rebuild should pass the computed value. Nothing in the engine currently calls it with anything but the highest priority, which is why it has survived.

Raising a process's priority class from inside the process is a request the host may refuse, and refusing is not reported. Treat both setters as advisory.

The read/write pairs exist so that a subsystem can raise priority for a phase and restore what it found, rather than assuming normal — the level is read before, not defaulted after. A rebuild should keep that save/restore shape even where the operations are no-ops.
