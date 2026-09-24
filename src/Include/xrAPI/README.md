# src/Include/xrAPI

> One record, one global instance: the engine's service locator, and its deliberate cycle-breaker.

## What this module is responsible for

A single header declaring a single mutable record and the one instance of it that exists per process. Each module, as it initializes, installs its own service into a field; every module reads whatever it needs from the same record.

It is chapter 4's smallest part and chapter 5's reason for existing — [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order) names the global environment as one of the engine's three structural cycles, and notes that it is a cycle broken on purpose.

## Where it sits

It rests on nothing but forward declarations. The renderer interfaces it names are in the sibling directory; the audio, AI, script and UI types it names are declared in modules that come much later in the build and are never included here. That is the whole trick: the record names *nothing it needs the definition of*, so every module can see the record while seeing almost none of its peers.

## The load-bearing ideas

**Installation order is a contract, not an accident.** Audio, then AI, then the selected renderer as a group of five fields, then the script engine, then the game's UI root. Teardown is the reverse. Reading a field before its owner has installed itself is the most common startup crash in this engine and nothing in the type prevents it.

**There is no synchronization, and that is only safe because of an unwritten rule**: every write happens before the worker threads that read exist and after they are joined. A rebuild that starts workers earlier must add real synchronization or make the record immutable after startup.

**The dedicated-server flag gates entire subsystems.** With it set, no renderer and no audio system are ever installed, and the corresponding fields stay absent for the process's whole life. Code that must run in both configurations checks the flag rather than the field.

**A rebuild should replace this with explicit dependency injection.** The recipe notes at each use site what is actually being reached for, so the replacement can proceed one field at a time rather than as one large change.

## The twins

| File | Role |
|---|---|
| [`xrAPI.h`](xrAPI.h.md) | The global environment record: nine service fields plus the dedicated-server flag, with the installation order and lifetime rules |
