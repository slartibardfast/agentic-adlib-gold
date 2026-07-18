# PCem is a validation oracle only, walled off from the driver codebase

- Status: accepted
- Scope: software
- Date: 2026-07-19

## Context and Problem Statement

PCem (GPL-2.0, `sarah-walker-pcem/pcem`) ships a register-level emulation of the
Ad Lib Gold in `src/sound/sound_adlibgold.c`: the OPL3 (with a NukedOPL option),
the MCS-1175 sample and DMA path with its FIFO and threshold interrupt, the
service timer the driver and `spec/NotifyLiveness.tla` both use, the YM7128
surround, and the TDA8425-style mix. The timer cadence and the FIFO, DMA, and
interrupt surface are the same ones the driver programs and the specula lane
models, so an emulator run is a plausible stand-in for the card when real
hardware is scarce.

PCem is GPLv2; the driver is not. Its expression (structure, sequence,
organization, or verbatim code) cannot cross into `adlib_gold/` without forcing
copyleft onto the driver. The useful thing about PCem is running it; the
forbidden thing is mining it for the driver's implementation.

## Decision

Use PCem as a validation oracle only, behind a clean-room wall.

- **Oracle, not source.** PCem is run; the driver is exercised inside it and
  observed (does PCM play through, is MIDI clean). No PCem source enters the
  driver codebase or this host, not vendored, not paraphrased, not adapted. The
  test harness drives PCem as a subprocess against an externally-supplied binary
  and captured audio output, never linked and never copied into the tree.
- **Spec authority stays the hardware.** The `.allium` and `.tla` specs derive
  from the Ad Lib Gold datasheet and SDK and from real-card observation, not
  from PCem's source. Behavioral facts observed by running PCem may stand as a
  test oracle; they do not become the spec's source.
- **Agent recusal.** The clean-room reader/writer separation applies to AI
  agents, not only to humans. An agent that has ingested PCem source is on the
  reader side and may build and run the harness, author specs from the
  datasheet, and work in the host; it may not author the driver's
  hardware-interface code, the ISR, the DMA setup, the register, FIFO, and
  timer programming. A hardware-interface edit is routed to a clean agent that
  has not read PCem source. The boundary is auditable: which files a session
  read is on the record.

## Consequences

- Good: the driver's license position stays clean and the boundary is auditable
  (no PCem-derived file in the driver tree), while PCem's fidelity on the
  digital PCM path and the OPL3 register sequence is still usable as an oracle.
- Cost: a hardware-interface edit must be routed to a clean agent, a
  coordination overhead; any session that has read PCem source is scoped to
  validation and host work for the rest of its thread.
- Residue: PCem cannot validate the physical delivery of the selected interrupt
  line (the `call/0034` residue) since it hardwires the card's interrupt, and
  its analog modeling (surround, bass and treble) is approximate; both stay
  real-hardware questions for the operator session.
