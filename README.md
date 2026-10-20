# cinder-aac

AAC Bluetooth transmit for the Sony NW-A50 series. Out of the box the player can *receive* AAC but
only *sends* SBC, aptX, aptX HD and LDAC. This project adds AAC as a send codec, for headphones
that have none of the others (AirPods, Beats, most budget pairs) and would otherwise fall back
to SBC.

It is kept apart from [Cinder](https://github.com/superwilso) on purpose: it touches Sony's
Bluetooth stack, so it lives in a repo of its own that Cinder does not depend on.

## How it works

- The player already ships Fraunhofer's AAC encoder (`/system/lib/libFraunhoferAAC.so`,
  `aacEncOpen` / `aacEncEncode`), licensed with the device. Nothing in Sony's firmware calls it.
- This project hooks the existing stack at run time so that it advertises an AAC source endpoint
  and encodes with that on-device library.
- It ships no encoder and no Sony code.

## Rules for this repo

1. **No Sony bytes.** No binaries, no decompiled listings, no copied tables or strings beyond what
   a hook needs to recognise its target (a hash, an offset).
2. **Notes in our own words.** Describe what a function does; don't paste what Ghidra printed.
3. Everything runs against the copy of the firmware already on the owner's player.

Reverse engineering here is for interoperability: making an independent program (this one) work
with the player's Bluetooth stack (UK CDPA 1988 s.50B / s.50BA, EU Directive 2009/24/EC art. 6,
US 17 U.S.C. 1201(f)).

## Status

Research. Nothing here is safe to install yet.
