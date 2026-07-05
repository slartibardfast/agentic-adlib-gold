# 0010 Full-duplex over the second DMA channel

The card records and plays digital audio, but not at the same time: `NewStream` rejects the
second direction while one is open (`STATUS_INVALID_DEVICE_REQUEST`). That was the right
interim behaviour while the driver had one DMA channel, but the hardware has two. The YMZ263
carries two sampling channels, and the control chip has a second DMA select-and-enable
register (reg 14h, `DEN1` + `DMA SEL 1`; the DOS toolkit drives it through
`CtSelectDMA1ChannelSampChan`). This milestone lets a render stream and a capture stream run
together over the two DMA channels.

## Who

- **Casey** can use voice chat and record-while-monitoring: playback and capture at once, the
  thing a half-duplex driver cannot do.
- **Piotr** gets the two-DMA path exercised by a spec obligation and the resource claim made
  explicit in the INF, so a later change that breaks the pairing fails a check.

## The constraint: full-duplex is mono plus mono

Stereo digital audio already uses both MMA channels (channel 0 and channel 1 interleaved,
`call/0017`). Full-duplex also needs both channels, one per direction. The two cannot both
have both channels, so full-duplex is mono render plus mono capture: the render stream owns
MMA channel 0 over DMA channel 0, and the capture stream owns MMA channel 1 over DMA channel
1. A stereo stream and a second direction cannot coexist; when a stereo stream is open, the
second direction is still refused, and a full-duplex pair negotiates mono on both sides.

## The rule for this milestone

The milestone keeps the no-hollow-green discipline. The direction-to-channel mapping and the
reg-14h DMA-select derivation are pure, tested helpers. The behaviour rule (a render and a
capture stream may coexist, each on its own DMA channel) goes into `wave.allium`, and the
resource claim (two DMA channels) is explicit in the INF and attested on the GoldLib, which
has the two-DMA control chip. The build stays byte-reproducible.

## Build sequence

### Claim a second DMA channel in the INF {#claim-second-dma}

- verify: attested operator
- inputs: software/adlib_gold/main/adlibgold.inf

Add a logical configuration that claims two DMA channels (two `DMAConfig` lines within one
`LogConfig`), offered at a lower priority than the single-DMA configuration, so an operator
who wants full-duplex assigns a second channel while a half-duplex install still works with
one. CRLF and ASCII stay intact.

### Allocate the second DMA channel {#allocate-second-dma}

- depends: #claim-second-dma
- verify: cargo-free; attested operator

When the resource list carries two DMA channels, the adapter allocates both and hands the
wave miniport a second `PDMACHANNELSLAVE` alongside the first. With one channel present the
miniport keeps its current single-channel behaviour, so a half-duplex card is unaffected.

### Let a render and a capture stream coexist {#dual-stream}

- depends: #allocate-second-dma
- verify: cc -Werror the wave test and run it

Replace the single active-stream slot and the full-duplex rejection with a render slot and a
capture slot. `NewStream` admits the second direction when the second DMA channel exists and
both formats are mono; it still refuses a second direction when only one DMA channel is
present or when either side is stereo. The direction-to-channel mapping (render to channel 0,
capture to channel 1) is a pure, tested helper.

### Program the second DMA channel and its MMA channel {#second-channel}

- depends: #dual-stream
- verify: attested operator

The capture stream programs MMA channel 1 (via `WriteMMA1`) and control register 14h with its
DMA-select bits (derived in a pure helper mirroring `WaveDmaSelectBits` for the first channel)
and the enable bit, and starts its own DMA channel. The render stream keeps channel 0. The
shared sampling interrupt is decoded per channel from the MMA status, so each stream is
serviced from its own FIFO ready flag.

### Service both streams from the shared interrupt {#service-both}

- depends: #second-channel
- verify: attested operator

The ISR advances whichever direction its channel's status flags, and notifies each stream's
service group independently. Then the milestone is complete: mono render and mono capture run
together over the two DMA channels, attested on the GoldLib.

## Depends on

Milestone `0007` (the single-DMA wave path this extends) and `call/0017` (the stereo path that
sets the mono-only constraint). The second-DMA registers are documented in the driver's
`manual/src/ch07-low-level.md` (reg 14h) and the toolkit's DMA-channel API.
