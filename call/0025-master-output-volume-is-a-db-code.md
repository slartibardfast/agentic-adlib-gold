# The control-chip master output volume is a dB code, not a linear level

- Status: accepted
- Scope: software
- Date: 2026-07-05

## Context and Problem Statement

On Windows 2000 the driver loaded and ran, detected the card, and correctly programmed the OPL3
FM synth and the MMA digital-audio engine (a boot trace confirms note-on events and repeated PCM
stream programming), yet produced no audible output on any path: PCM playback, MIDI through the
FM synth, and both the DirectSound and DirectMusic tests in dxdiag were all silent. Three
independent output paths being silent points at the one stage they share: the control chip's
Final Output Volume (reg 04h/05h), which every source sums into before line-out.

A code review against the SDK found that register mis-programmed. The SDK (Control Chip register
reference) documents reg 04h/05h as a dB code: bits D6 and D7 are forced to 1, and the field
D5-D0 is a dB value, running from +6 dB at 0x3F down to -64 dB at 0x1C, and any value below 0x1C
gives -80 dB (off). So the audible byte range is 0xDC to 0xFF, and 0 dB is 0xFC. The driver instead
treated this register as if it used the linear 128-to-255 encoding of the FM and sampling
volume registers (reg 09h-0Ch): the hardcoded default was 0xD8, whose D5-D0 field is 0x18, below
the audible floor, so -80 dB off; and the topology master node advertised a minimum of 0xC0,
also inside the off band. Every path was therefore muted at the shared master.

Two related gaps compounded it. The MMA digital-audio output volume (reg 0Ah, per channel,
0 = minimum and 0xFF = maximum) was defined but never written, leaving the sampling path at the
chip's undocumented power-on default. And the mixer registry restore replayed a saved value
verbatim, so a stale master value saved from the old buggy default would survive a default
change unless the operator deleted the settings key.

## Decision

Four fixes.

- The master output-volume default becomes 0xFC (0 dB), replacing 0xD8 (which was off). The
  default now sits in the audible band.
- The topology master node's minimum register value becomes 0xDC (0x1C in the field, the -64 dB
  floor of the audible range), replacing 0xC0. The whole slider range is now audible, and the
  taper is confined to the register's real dB range.
- The MMA output volume (reg 0Ah) is written to 0xFF (maximum) at PCM start, on both channels for
  a stereo stream and on channel 0 for a mono stream, matching the SDK's per-channel register.
- The mixer registry restore clamps a restored master value that is below 0xDC back up to 0 dB
  (0xFC), so a stale or corrupt saved value can never silence the card, and no manual registry
  deletion is needed on an already-installed machine.

## Consequences

- Good: the shared master node is audible by default, the master slider stays inside the audible
  band, the sampling path sets its own output volume, and a stale registry value self-heals.
- Not all volume registers on this chip share an encoding. The master output volume (reg 04h/05h)
  is a dB code; the FM and sampling source volumes (reg 09h-0Ch) are linear 128-to-255, and those
  were already correct (default 0xC0, floor 0x80). The fix touches only the dB-coded master and
  the unwritten MMA output volume; it does not change the source-volume encoding.
- Unverifiable on this host: the driver is untestable here, so the fixes are confirmed by the SDK
  cross-check, the code review, and a byte-reproducible compile; audible output is validated by
  the operator on the GoldLib. The boot trace already proved the digital stages upstream are
  correct, so the mixer output stage was the only remaining silent node.
- Ships as v1.0.0-alpha.10.
