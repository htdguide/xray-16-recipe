# src/xrSound/OpenALDeviceList.h

> Declares the enumerated audio devices and what each was found to support.

**Needs** — [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`OpenALDeviceList.cpp`](OpenALDeviceList.cpp.md) · [`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md)
**Tier floor** — T1: the capability record is a packed bit field.

## Purpose

Declares the device list built in [`OpenALDeviceList.cpp`](OpenALDeviceList.cpp.md).

## Exported units

- **`DeviceDescription`** — one device: its own reported name, its specification major and minor
  version, and a capability record holding whether it is selected, which reverb-extension
  generation it claims (0 for none, else 2 to 5), and whether it offers the effects extension.
- **`DeviceList`** — the enumerated set, the system default's name, the count and per-index
  accessors, and the selection policy.

## Notes

The capability record is a packed bit field overlaid on a 16-bit word, which is an artefact of
wanting to copy it as one value; a rebuild should use ordinary fields. What matters is that the
reverb generation is a *number*, not a flag — the extension had four incompatible generations and
the device reports the newest it supports.
