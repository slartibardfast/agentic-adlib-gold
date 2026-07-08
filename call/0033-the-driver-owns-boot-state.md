# The driver owns every audio-affecting register at boot

- Status: accepted
- Scope: software
- Date: 2026-07-08

## Context and Problem Statement

The GoldLib is a legacy ISA card with no PnP layer, confirmed by the operator. Nothing
external ever configures it: Windows "assigns" resources only because the INF's hardwired
LogConfig declares them (`call/0021`/`call/0022`), and the card's own power-on state comes
from its EEPROM, which any DOS tool can rewrite. So every register the driver leaves
unwritten is undefined on every boot, on every machine, forever, and the driver is the sole
agent that can align the card with the claimed resources. Three releases in a row were
un-written-state defects (the MMA output volume, `call/0025`; the FIFO masks, `call/0029`;
the timer masks, `call/0032`). A full sweep of the SDK register set against the boot path
found the remainder.

## Decision

At boot the driver writes every audio-affecting register it owns, under one invariant: AEN
is off from the driver's first touch until every interrupt source on the shared line is
masked, and the wave miniport's reg-13h write is the only AEN enable.

Control chip, in `ControlRegReset` after the mixer restore (fixed policy, deliberately NOT
in the registry-backed defaults table, so a future default change cannot be shadowed by a
stale saved value, the `call/0029` lesson):

| Reg | Value | Meaning |
|---|---|---|
| 13h | 0x00 | AEN and DMA off first; the wave miniport writes the real 0x9B last |
| 14h | 0x00 | Second-channel DMA enable off |
| 01h | 0x00 | Telephone line disengaged |
| 10h | 0x80 | Telephone volume silent (linear floor) |
| 11h | 0x08 | MFB set (mic removed from the output mix), PC speaker disconnected, aux stereo, both antialiasing filters set for playback |

MMA, in the wave miniport's pre-AEN insurance block, now in this order: playback-engine
reset on both channels (the manual's RST procedure, clearing a warm-reboot leftover GO),
format masks, MIDI control masks, timer masks, then `ConfigureDmaAndIrq`. The MIDI masks
close a second storm window this plan's review found: the MIDI subdevice installs after the
wave subdevice, so between AEN going live and MIDI's own Init, an unmasked power-on receive
interrupt with stale FIFO bytes had no registered handler on a level-sensitive line. MIDI
Init still owns the proper reset, default masks, and stale-byte drain afterwards.

OPL3: `Opl3_BoardReset` zeroes the test registers (01h and 101h), scratch state that is
meaningless under NEW=1 but otherwise survives a warm reboot.

YM7128: when the surround module is present, topology Init downloads the bypass preset once
instead of trusting the chip's initial-clear, which holds at cold power-on but not across a
warm reboot. This is gated on the sp2modes payload audit (`plan/0015`), because an unaudited
write sequence must not join the boot path.

## Consequences

- Deliberately unwritten, and why: reg 15h (EEPROM-pinned base address; if it does not match
  the claimed base the card is unreachable and no write can arrive), regs 16h/17h (the SCSI
  subsystem's), reg 00h ST/RT (EEPROM persistence decided out, `call/0019`).
- Capture runs with the filters in playback direction, deterministic but unfiltered, until
  per-direction filter switching lands as a `plan/0014` menu column.
- On a telephone-equipped card the reg-01h write hangs up the line at boot, which is the
  correct posture for a fresh OS session.
- Comment corrections ride along: `ConfigureDmaAndIrq` stops saying "PnP-assigned", the
  `NewStream` rejection cites `call/0031`/`call/0032`, and reg 15h's header documents the
  non-PnP rationale.
- The boot sequence's correctness depends on the subdevice install order (Topology, FMSynth,
  Wave, MIDI); the reg-13h zeroing site records that contract in a comment.
