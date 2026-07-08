# Full duplex stays out of scope: the GoldLib does not wire a second DMA path

- Status: accepted
- Scope: software
- Date: 2026-07-08

## Context and Problem Statement

`call/0030` descoped full duplex, but its supporting rationale was wrong on two counts, and
the operator corrected both. The documentation says the original card CAN do it: reg 13h and
reg 14h carry independent DMA enables and selects, reg 14h's table offers lines 1-3 on a
Gold 1000, the DOS toolkit ships `CtSelectDMA0ChannelSampChan` and
`CtSelectDMA1ChannelSampChan` as first-class calls (SDK 5-16), and the Yamaha overview names
a separate DMA per sampling channel as the designed simultaneous record-and-playback method.
And the test machine has BOTH DMA 1 and DMA 3 free, so the line pool was never the obstacle.

The true constraint is board wiring. The GoldLib is built for the 8-bit ISA connector, which
carries DRQ1/2/3 only; DRQ0/DACK0 sit on the 16-bit extension, which is why DMA 0 is a Gold
2000 capability in the first place. The operator attests, from knowledge of the GoldLib's
design, that the second DMA request/acknowledge path is not believed to be wired to the edge
connector on the GoldLib: one DRQ/DACK pair serves the card, and reg 14h's channel-1 select
has nothing to drive. A second simultaneous audio DMA channel therefore cannot reach the bus
on this card, however the registers are programmed.

## Decision

Full duplex remains out of scope, on the corrected ground above. This supersedes
`call/0030`, whose outcome stands and whose rationale (machine DMA contention, a DMA-0
requirement) does not. Everything `call/0030` disposed stays disposed: `plan/0010` remains
withdrawn with its design preserved in git history, and the `NewStream` half-duplex
rejection (`STATUS_INVALID_DEVICE_REQUEST`) remains the permanent contract, with no driver
change.

## Consequences

- The record is falsifiable: the attestation is a design-knowledge belief, and a checked
  build could probe it in one boot (program reg 14h for DMA 3, enable DEN1, arm the channel,
  and report whether the DMA counter moves). Only a future need for duplex justifies running
  that probe; a positive result would supersede this decision the same way it supersedes its
  predecessor.
- A future Gold 2000 target, or a GoldLib revision that wires the second path, reinstates
  the withdrawn `plan/0010` design from history and attests it on that hardware.
- Lesson: a capability has three layers here, the chip, the board, and the machine. The chip
  documents duplex, the machine had the lines, and the board is where it dies. Record which
  layer a constraint lives in before writing the decision.
