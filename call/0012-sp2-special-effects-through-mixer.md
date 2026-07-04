# The SP2 special-effects module is exposed through the mixer for all modes

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The Ad Lib Gold carries a Yamaha YM7128 (SP2) surround processor, the special-effects
module the SDK documents in `appendix-sp2.md`. The driver already detects it (the
Control Chip surround-present bit in `common.cpp`) and can reach it (register `0x18`, a
bit-serial protocol). But the topology miniport (`algtopo.h`) exposes no SP2 node, so a
user cannot reach the effect through the standard Windows mixer. That is a gap between
what the hardware and the SDK offer and what the driver surfaces.

## Decision

Expose the SP2 special-effects module fully through the KS topology mixer, so the
Windows built-in volume control (sndvol32) drives it, across every mode the SDK
documents. The topology gains SP2 nodes, an enable and a mode selector with any level
the module supports, each mapped onto the register-`0x18` bit-serial protocol, and the
driver ships the documented presets so every supported mode is selectable.

Where the card reports no surround module present, the SP2 nodes are absent rather than
dead, so the mixer reflects the hardware.

## Consequences

- Good: Casey reaches the Gold's surround effect from the same mixer as every other
  control, with no vendor applet, and every documented mode is available.
- New work: the skeleton has no SP2 topology node, so this is an addition to `algtopo`
  and the common object, not a distill of existing code. plan/0005 carries it.
- The `.allium` topology spec covers the SP2 nodes and the mode set, and the tests hold
  the register-`0x18` protocol to the datasheet.
