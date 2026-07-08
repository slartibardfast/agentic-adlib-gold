# 0014 Full duplex from the configuration menu

`call/0032` puts full duplex back in scope without the second DMA line the GoldLib lacks:
wave streams draw (channels, transport, service clock) from a menu resolved at stream open by
a pure allocator. The second direction runs the free MMA channel over its own port pair in
PIO, or takes the idle DMA line when the open render is the 16-bit PIO path. DMA streams move
their notification onto MMA Timer 0 and mask their FIFO interrupt, cutting the interrupt rate
about tenfold. This milestone implements the menu; it is gated behind the `plan/0013` retest
because every PIO path is hardware-unproven until PCM is audible at all.

## Who

- **Casey** gets voice chat and record-while-monitoring: playback and capture at once, on the
  card as wired.
- **Piotr** gets the allocator as a pure function whose table is the contract: one test per
  menu row, the behaviour rule in `wave.allium`, and a regression fails the suite rather than
  silently double-booking a channel.

## Design

The menu and its warts are recorded in `call/0032`: duplex is mono plus mono; which input
side capture hears depends on open order; KMixer's mono fallback when the second direction
restricts formats needs hardware validation. The allocator consumes
(open-state, direction, format) and yields (channel set, transport, service clock) or
refuses; `NewStream`, `SetState`, and the ISR dispatch consume its output and encode no
pairings of their own.

## Build sequence

### Derive the allocator from the menu {#allocator}

- verify: gcc -o /tmp/wavealloc_test software/adlib_gold/main/tests/wavealloc_test.c -I software/adlib_gold/main && /tmp/wavealloc_test
- inputs: software/adlib_gold/main/wavereg.h, software/adlib_gold/main/tests/wavealloc_test.c
- depends: plan/0013#ship-retest-build

A pure helper in the `wavereg.h` style maps every (open-state, request) pair to its menu row;
the test enumerates the whole table, refusals included.

### Plumb channel 1 for PIO {#channel1-pio}

- depends: #allocator
- verify: attested call/0032
- inputs: software/adlib_gold/main/common.cpp, software/adlib_gold/main/common.h

Add the channel-1 read path (`ReadMMA1`, mirroring `WriteMMA1` through the 38Eh/38Fh port
pair) and the channel-1 PIO stream programming: play and format registers per the menu row,
FIFO threshold unmasked, fill and drain against channel 1's data port.

### Let two streams coexist under the allocator {#dual-streams}

- depends: #channel1-pio
- verify: attested call/0032
- inputs: software/adlib_gold/main/algwave.cpp, software/adlib_gold/main/algwave.h

Replace the single active-stream slot with a render slot and a capture slot. `NewStream`
admits what the allocator admits and refuses what it refuses; the ISR dispatches each
channel's FIFO flag to its own stream and notifies each service group independently.

### Move DMA notification onto Timer 0 {#timer-notify}

- depends: #dual-streams
- verify: attested operator
- inputs: software/adlib_gold/main/algwave.cpp

A DMA stream programs Timer 0 to its notification interval, masks its FIFO interrupt, and
notifies on the timer flag; the checked build's stop line shows the interrupt count falling
to the notification cadence while audio stays continuous.

### Attest duplex on the GoldLib {#duplex-attest}

- depends: #timer-notify
- verify: attested operator

Mono render plus mono capture run together (both orders, both transports per the menu),
KMixer falls back to mono when the second direction restricts formats, and the recorded
capture is intelligible. The `wave.allium` rule and its obligations are dispositioned.

## Depends on

`call/0032` (the menu this implements), `plan/0013` (the retest that first proves PCM and the
hardware ground the PIO paths stand on), and `call/0031`'s wiring finding (the one-DMA-line
constraint the menu encodes). `plan/0010` stays withdrawn; this design replaces it.
