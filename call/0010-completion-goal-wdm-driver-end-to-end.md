# The completion goal: a WDM driver, end-to-end, that Casey can hear

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The milestones so far bring up one sound path (the 8-bit PCM tone) on one target. The
project needs a stated definition of done that the milestones ladder up to, so a
milestone earns its place by moving the driver toward completion rather than sideways.

## Decision

Complete the Ad Lib Gold WDM driver end-to-end across three subsystems, each brought
up on the test target (Windows 98SE on the GoldLib recreation, `call/0006`) and proven
by repeatable dual-hosted builds (`call/0007`, `call/0009`):

- **PCM** (the WaveCyclic wave path): 8-bit playback first (plan/0002), then 16-bit
  with high-fidelity downsampling (`call/0011`).
- **Mixer** (the Control Chip topology): the volume, tone, and mute controls, plus the
  SP2 special-effects module for every mode the SDK documents (`call/0012`), as KS nodes
  that sndvol32 drives.
- **FM synth** (the OPL3): the card's signature voice, driven by the driver's own
  `fmsynth` miniport rather than a stock synth (`call/0014`), under spec and in the whole.

Each subsystem is specified before it is trusted. Because the driver starts from a
skeleton, its behaviour is specified as `.allium` distilled from the code (allium
`distill`) and then leads the remaining work. The chips' timing is specified as `.tla`
and model-checked (`call/0013`), since the driver runs on far faster CPUs than the
hardware assumes. Both live with the software and run in its CI, and the host plan
references them by pin. The SDK manual is the authority for every chip specific,
consulted as each figure is needed.

The driver is complete when its KS filters enumerate on the target, each subsystem
produces or routes sound as its spec says, and both build lanes reproduce the artifact
byte-for-byte.

## Consequences

- Good: every milestone has a north star. Casey hears PCM and FM and sets levels in the
  mixer; Piotr builds it reproducibly; Quill keeps the specs and the manual in step.
- Lacuna: the driver also carries a MIDI UART miniport (the external MIDI port,
  midi.cpp) that this goal does not name. plan/0007 dispositions it as in-scope or
  deferred, so it is not left silent.
- The goal is revisited if the intent shifts; this record is then superseded, not edited.
