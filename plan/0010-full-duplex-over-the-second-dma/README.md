# 0010 Full-duplex over the second DMA channel

**Withdrawn**, superseded by `call/0030`. The Gold 1000 class (the GoldLib included) cannot
select DMA 0, and the machine's remaining 8-bit lines belong to the floppy controller and the
parallel port, so the second simultaneous audio DMA line this milestone needs does not exist
on the target. The build sequence was removed from the task graph before any task started (no
receipts); this file's prior revision in git history preserves it for a hypothetical Gold
2000 port. The half-duplex `NewStream` rejection is the permanent contract.

The design context below is kept as the record of what was intended and why.

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

## Depends on

Milestone `0007` (the single-DMA wave path this extends) and `call/0017` (the stereo path that
sets the mono-only constraint). The second-DMA registers are documented in the driver's
`manual/src/ch07-low-level.md` (reg 14h) and the toolkit's DMA-channel API.
