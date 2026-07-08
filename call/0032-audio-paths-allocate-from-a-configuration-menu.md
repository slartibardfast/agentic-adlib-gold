# Digital-audio paths allocate from a configuration menu, and full duplex returns to scope

- Status: accepted
- Scope: software
- Date: 2026-07-08

## Context and Problem Statement

`call/0031` descoped full duplex on the ground that the GoldLib wires one DRQ/DACK pair, so
the second-DMA design (`plan/0010`) has nothing to drive. That wiring finding stands. What it
does not rule out is a register-driven transport: the MMA FIFO data port (reg 0Bh) streams
under CPU control, channel 1 has its own address/data port pair (38Eh/38Fh) so PIO on it
cannot disturb channel 0's DMA handshake, and the driver already sustains a heavier PIO
workload than duplex needs (the 16-bit mono render fill at 44.1 kHz). The status decode even
reserved a channel-1 helper for this. The operator reframed the design space: transports,
channels, and service clocks are not fixed pairings to choose between; they are a
configuration menu the driver reads at stream open.

The MMA also carries three interrupt timers and a base counter (SDK 7-40 to 7-42): Timer 0
counts 16 bits at 1.89 us resolution, Timers 1 and 2 tick on the 12-bit base counter, Timer 2
is readable through a latch, and reg 08h holds per-timer masks and start bits. A timer is
therefore an allocatable notification clock, decoupled from the FIFO thresholds that move
data.

## Decision

Wave streams acquire their resources from a menu, resolved by a pure, tested allocator
(state and request in; channel set, transport, and service clock out; or refuse):

| Already open | New request | Assignment |
|---|---|---|
| nothing | stereo render | ch0+ch1, ILV, DMA (the validated path) |
| nothing | mono render 8-bit | ch0, DMA |
| nothing | mono render 16-bit | ch0, PIO (the dither path) |
| nothing | capture | ch0, DMA |
| capture (ch0, DMA) | mono render | ch1, PIO |
| mono render 8-bit (ch0, DMA) | capture | ch1, PIO |
| mono render 16-bit (ch0, PIO) | capture | ch1, DMA (the line is free) |
| stereo render | capture | refuse: stereo owns both channels |
| capture | stereo render | refuse: stereo owns both channels |
| any | same direction | refuse (as today) |

Full duplex thereby returns to scope with no second DMA line: the second direction takes the
free channel over PIO, and when the open render is the 16-bit PIO path the allocator hands
capture the idle DMA line instead. `call/0031` is superseded; its board-wiring finding
becomes a menu constraint (one DMA line) rather than a feature veto.

The service clock is on the menu too. A DMA stream's only interrupt need is the WaveCyclic
notification, so it takes Timer 0 at the notification interval and masks its FIFO interrupt
(MSK gates the IRQ line only; the AIL reference runs ENB=1 with MSK=1, so DRQ is unaffected).
That divides the DMA-path interrupt rate by roughly ten and makes it independent of sample
rate. A PIO stream keeps its FIFO threshold unmasked, since that interrupt is what moves its
data.

One consequence is immediate rather than gated: reg 08h's timer masks are an interrupt
source with the same undocumented power-on state as the FIFO masks `call/0029` guards, and a
running unmasked timer asserts the same level-sensitive line the ISR does not clear. The
pre-AEN insurance therefore extends to reg 08h (mask all three timers, timers and base
counter stopped) and ships before the pending hardware session, as alpha.13.

## Consequences

- `plan/0014` carries the duplex work, gated behind the `plan/0013` retest: every PIO path
  is hardware-unproven until PCM is audible at all.
- Recorded warts: duplex is mono plus mono (stereo interleave owns both channels); which
  input side capture hears depends on open order (channel 0's ADC side when capture opens
  first, channel 1's when it opens second); KMixer must fall back to mono when the second
  direction restricts formats, which needs hardware validation.
- The allocator table above is the contract: each row is a test case, and the behaviour rule
  enters `wave.allium` with its obligations dispositioned.
- `plan/0010` stays withdrawn; this design replaces it rather than reviving it.
