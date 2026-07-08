# Control-chip bitfields follow the SDK register map figures

- Status: accepted
- Scope: software
- Date: 2026-07-08

## Context and Problem Statement

The alpha.11 hardware retest (dxdiag on the Windows 2000 GoldLib) played FM MIDI audibly but
produced no PCM sound. The DebugView trace showed every software stage healthy: streams reached
RUN, the MMA was programmed with the reference-validated values, and the boot line recorded
`reg13=0xB3` for IRQ 7 / DMA 1. An audit of every control-chip bitfield against the SDK's own
register-map figures (Developer Toolkit PDF, pages 7-3 and 7-9; the OCR text collapses these
tables) found the driver's field layouts shifted by one bit in five registers. The audit was
prompted by one decisive figure: register 13h is `DEN0 | DMA SEL 0 (D6-D4) | AEN (D3) |
INT SEL A (D2-D0)`, while the driver placed DMA SEL at D6-D5 and AEN at D4. The written 0xB3
therefore selected DMA line 3 (PnP granted DMA 1) and left AEN clear, so the card never raised
an interrupt and never asserted DRQ on the programmed channel. PCM starved in silence while FM,
which needs neither, played. The FM-audible/PCM-silent split is exactly this fault's signature.

## Decision

Derive every control-chip bitfield from the PDF register-map figures and fix the five divergent
layouts:

- **Register 13h/14h**: DMA SEL occupies D6-D4 (shift 4) and AEN is D3 (0x08). The boot write
  for IRQ 7 / DMA 1 becomes 0x9B. Register 14h's DMA SEL 1 moves the same way.
- **Register 08h**: ST-MONO occupies D4-D3 (linear stereo = 0x08) and SOURCE occupies D2-D0
  (both channels = 6, right = 4, left = 2). The boot default becomes 0xCE (linear stereo, both
  channels, unmuted); the prior 0xC4 decoded on hardware as forced-mono, right-channel-only.
  Because the registry restore replays saved bytes verbatim and no UI writes these fields, the
  restore also normalizes a saved OutputMode to the fixed layout, preserving only the user's
  mute bit, the same self-heal shape as the master-volume clamp (`call/0025`).
- **Register 00h read**: the option bits are OP0 telephone = D4, OP1 surround = D5, OP2
  SCSI = D6 (set means absent). The GoldLib's 0xD1 therefore reports a surround module present
  and no telephone; the prior layout read it backwards and withheld the SP2 nodes from a card
  that has the module.
- **Register 11h**: FLT0 is D1 and FLT1 is D0 (currently unwritten; latent).
- **Model ID values**: 0 = Gold 2000, 1 = Gold 1000, 2 = Gold 2000MC (names were swapped; only
  a bound check consumes them today).

Two hardening changes ride with the layout fixes, both to keep the next hardware session
conclusive:

- Both MMA format registers are masked before the reg-13h write. AEN has never been set on
  hardware before this fix, and the MMA's power-on mask state is undocumented; masking first
  removes the one path to a level-triggered interrupt storm at StartDevice.
- The stereo and 8-bit-mono render starts prime the channel-0 FIFO with four zero bytes before
  GO. This restores the step the validated Miles AIL sequence performs
  (`plan/0008/stereo-mma-reference.md` :961-964) and the driver had omitted; the omission was
  justified from the "working" 8-bit mono path, which has never actually been heard on hardware.

## Consequences

- PCM playback finally has its interrupt and DMA plumbing programmed as the hardware defines
  it; the stereo image and output effect return to linear/both-channels for every source.
- The SP2 surround nodes install on the GoldLib for the first time. Boot risk is low
  (surround stays disabled until a user enables it), and the preset payloads are a follow-up
  audit before anyone does.
- The checked build gains an ISR counter and a stop-line diagnostic (ISR count, DMA position,
  MMA status at RUN to PAUSE), so if PCM is still silent the same log localizes the fault to
  DRQ, IRQ routing, or the analog path without another blind round.
- The `wavereg` test now pins absolute register bytes, not shift-relative expressions; a
  layout regression fails the suite instead of passing vacuously.
- Lesson recorded: decode register layouts from the PDF figures, never the OCR text, whose
  table columns collapse. This is the same lesson family as `call/0025` (decode each register
  against the SDK), one level down: verify the *figure*, not a transcription of it.
