# Reference: 16-bit stereo on the YMZ263 (MMA), traced from the Miles AIL driver

`call/0016` deferred stereo digital audio because the driver has no channel-1 access and
the manual's channel-1 addressing looked ambiguous. This trace of the Miles Audio
Interface Library AdLib Gold driver (its `pack_type 132` = signed-16-bit stereo path)
resolves that: it shows the exact hardware contract a known-good driver uses. It is the
blueprint for implementing stereo when that work is picked up and verified on hardware.

The whole 16-bit-stereo behaviour reduces to one thing: a stereo stream lights up the
channel-1 FIFO that a mono stream leaves off, and programs channel 1's play and format
registers alongside channel 0's. Everything else is the same as 16-bit mono.

## The one-bit decision

The format table carries a mono row and a stereo row for 16-bit. The only difference
between them is whether channel 1 is enabled. A `host-lint:ignore` block preserves the
traced source citations verbatim.

```host-lint:ignore
pack_modes    db 0,1,128,129,4,132   ; index 5 = 132 = signed-16-bit PCM stereo
;                m8 m4  s8   s4  m16  s16
PRC_1_values  ; [4] m16 = 00000000b  -> channel-1 play control OFF (mono)
PRC_1_values  ; [5] s16 = 00100110b  -> channel-1 play control ON  (stereo)
```

Index 4 (mono) leaves channel 1 dark; index 5 (stereo) turns it on. That single row
difference is the entire mono-versus-stereo decision at 16 bits. `packing` stays 4 for both
(the width-only code for 16-bit); there is no separate stereo engine and no distinct 12-bit
path. The card's 12-bit DAC simply reproduces the top bits of the 16-bit samples.

## The register contract

Stereo programs the play/rate register (reg 9) and the sample-format register (reg 12) on
both channels. Decoding the traced index-5 values against the manual (ch07):

Reg 9 (RST[7] R[6] L[5] FREQ[4:3] PCM[2] P-or-R[1] GO[0]):

| Channel | Value | Meaning |
|---|---|---|
| 0 | `01000110b` | R output on, L off, PCM, playback, GO deferred |
| 1 | `00100110b` | L output on, R off, PCM, playback, GO deferred |

So channel 0 drives the right output and channel 1 drives the left, confirmed by the
volume path below. That routing runs opposite to the manual's default channel naming; the
validation section treats it as the one point to confirm on the card.

Reg 12 (ILV[7] DATAFMT[6:5] FIFOINT[4:2] MSK[1] ENB[0]):

| Channel | Value | Meaning |
|---|---|---|
| 0 | `11000101b` | ILV on, format 2, FIFO interrupt at 96 bytes, interrupt enabled, DMA enabled |
| 1 | `01000011b` | ILV off, format 2, FIFO interrupt at 112 bytes, interrupt masked, DMA enabled |

This matches the manual exactly: ILV is set on channel 0 only, both channels set ENB, and
channel 1 masks its own FIFO interrupt so the shared interrupt is driven by channel 0. The
`DATAFMT = 2` selects the same format-2 byte order the driver already uses, so the on-wire
sample encoding is unchanged from the mono path.

## The programming sequence

The stereo branch of the rate-setup routine does this (traced citations boxed):

```host-lint:ignore
set_sample_rate, ADLIBG branch  (AIL source :919-979)
 :951-953  stereo => shr cx,1              ; halve the app's doubled rate to per-channel
 :958-959  MMA_write 0,9,80h ; MMA_write 1,9,80h    ; reset BOTH FIFOs (RST bit)
 :961-964  MMA_write 0,11,0 x4                       ; prime the FIFO for DMA
 :966-969  MMA_write 0,9, freq_bits OR PRC_0_values  ; channel 0 play/rate
 :971-973  MMA_write 1,9, freq_bits OR PRC_1_values  ; channel 1 play/rate (stereo)
 :975-976  MMA_write 0,12, SFC_0_values              ; channel 0 sample format
 :978-979  MMA_write 1,12, SFC_1_values              ; channel 1 sample format
 later     PRC_0_shadow OR 00000001b -> MMA_write 0,9 ; set GO to start
```

The `MMA_write <channel>, <reg>, <value>` form selects the channel: channel 0 through its
address/data port and channel 1 through its own. That is the channel-1 access the current
driver lacks, and it lines up with the driver's existing but unused `ALG_REG_MMA1_ADDR` /
`ALG_REG_MMA1_DATA` constants, so the driver's symmetric channel-1 port model is correct.

## The data path

The DMA controller moves one interleaved left-right 16-bit byte stream and is format-blind;
the MMA de-interleaves it internally into the two FIFOs per the ILV and per-channel format
set above. So stereo does not need a second DMA channel or a software interleave in this
design: one DMA channel feeds interleaved data and the chip splits it. The DMA is set up
for a single channel, byte count from the buffer length, single-mode read.

The rate reconciliation is worth noting: the application passes a doubled sample rate for
stereo, and the driver halves it back to the true per-channel rate before matching the
hardware rate table.

## Volume and pan

The stereo volume path keeps independent left and right attenuation rather than collapsing
to mono:

```host-lint:ignore
set_volume, ADLIBG branch  (AIL source :1049-1063)
 :1057-1059  MMA_write 0,10, right_attenuation   ; channel 0 = RIGHT
 :1061-1063  MMA_write 1,10, left_attenuation    ; channel 1 = LEFT
```

The mono fall-through averages the two and cannot pan, because a single FIFO drives both
outputs. The fixed mapping is channel 0 to the right output, channel 1 to the left.

## Validated against the manual

Every register value in the trace checks out against ch07:

- The play register's layout (RST, R, L, FREQ, PCM, playback-or-record, GO) and its
  reset-by-writing-80h sequence match the Playback and Recording Control register.
- Its R and L bits are a per-channel output enable: "Setting L or R enables output from the
  left or right channel", so a channel can drive either output.
- The format register's layout (ILV, DATA FORMAT, FIFO INT, MSK, ENB) matches the Sampling
  Format and Control register. ILV is channel-0-only and needs ENB on both channels, exactly as the
  index-5 values set it: ILV on channel 0, ENB on both, and channel 1's FIFO interrupt masked
  so channel 0 drives the shared interrupt. The FIFO INT codes decode to 96 and 112 bytes, and
  DATA FORMAT 2 is the same 12-bit-in-two-bytes encoding the driver already writes.
- Interleave alternates the two channels with channel 0 initiating, and the manual has channel
  0 govern the playback-or-record, FREQ and GO bits for both channels. The trace stays
  consistent with that: it sets GO only on channel 0, and its per-channel FREQ write to
  channel 1 is redundant rather than harmful.

One point runs against the default and has to be confirmed on the card. The manual names
channel 0 the left channel and channel 1 the right (the sampling-gain register at ch07 and the
antialiasing-filter register both say so). The trace routes the other way: it sets channel 0
to the right output and channel 1 to the left, in the play register and again in the volume
attenuation. Reg 9's R and L bits make that legal, since they choose each channel's output,
but the audible left-right result then depends on the order the interleaved DMA stream feeds
the two channels. However stereo is implemented, the channel-to-output mapping has to be
checked on hardware so left and right are not swapped.

## What this means for adlib_gold

The reference programs 16-bit stereo over DMA with ILV and both channels' play and format
registers. The DMA carries raw interleaved samples, and the DAC reproduces them at 12-bit
resolution.
The current driver instead runs 16-bit as mono PIO so it can dither each sample down to
12 bits for quality. Those two approaches do not compose: ILV interleave needs DMA
(`ENB = 1`), and the PIO fill dithers.

So implementing stereo is a design choice, and this reference documents the first option:

- **DMA raw, as in this reference.** Program both channels' reg 9 and reg 12 with the
  index-5 bit patterns, set ILV on channel 0, feed one interleaved DMA stream, and accept
  the raw 12-bit-from-16 truncation the DAC gives (no dither). Smallest and closest to a
  proven driver.
- **PIO with a software dual-FIFO interleave.** Keep the dither, and write the left sample
  to channel 0's FIFO and the right to channel 1's FIFO each frame. Larger, keeps the
  quality the mono path has, and has no proven reference here.

Either path needs the channel-1 register and FIFO programming this trace details, plus
hardware verification, which is the work `call/0016` deferred. This document is the
hardware contract for that work.
