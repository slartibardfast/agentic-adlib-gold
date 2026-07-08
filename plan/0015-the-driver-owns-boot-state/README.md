# 0015 The driver owns boot state

On a non-PnP card the driver is the sole configurator of everything the EEPROM does not pin,
and the EEPROM is mutable. `call/0033` records the sweep of the SDK register set against the
boot path and the invariant this milestone enforces: AEN off from first touch until every
interrupt source is masked, every audio-affecting register driver-written before the card is
declared started. These fixes ship as alpha.14, the retest build, because two of them (the
filter direction and the mic-to-output routing) change what the operator hears and testing
with them undefined would muddy both the PCM verdict and the garble A/B.

## Who

- **Casey** hears the card the same way on every boot: no EEPROM roulette deciding whether
  playback is filtered, the mic bleeds into the output, or a stale surround state colours
  everything.
- **Piotr** gets one stated invariant to check the boot path against, instead of an
  accumulating list of source-by-source storm fixes.

## Build sequence

### Write the control-chip fixed policy at reset {#control-policy}

- verify: attested call/0033
- inputs: software/adlib_gold/main/common.cpp

In `ControlRegReset`, after the mixer restore: reg 13h and 14h to zero (AEN and both DMA
enables off first; the wave miniport's 0x9B stays the only AEN enable), reg 01h zero, reg
10h silent, reg 11h to 0x08 (MFB on, SPKR off, XMO off, filters set for playback). Absolute
writes, kept out of the registry-backed defaults table.

### Reset the MMA engine and mask MIDI in the pre-AEN block {#mma-boot-reset}

- depends: #control-policy
- verify: attested call/0033
- inputs: software/adlib_gold/main/algwave.cpp, software/adlib_gold/main/algwave.h

In `ProcessResources`, ahead of the existing masks: the manual's RST procedure on both
channels' playback registers, then format masks, then the MIDI control masks (the second
storm window: MIDI installs after wave, so its receive interrupt must be masked before AEN
and re-owned by MIDI Init), then timer masks, then `ConfigureDmaAndIrq`.

### Zero the OPL3 test registers {#opl3-test-regs}

- depends: #control-policy
- verify: attested call/0033
- inputs: software/adlib_gold/main/fmsynth.cpp

`Opl3_BoardReset` writes 01h and 101h to zero beside the NEW-bit write.

### Audit the sp2modes payload against the SDK surround chapter {#sp2-payload-audit}

- verify: attested call/0033
- inputs: software/adlib_gold/main/sp2modes.h, software/adlib_gold/main/common.cpp

The YM7128 write protocol (`WriteSurroundReg`) and every preset byte in `sp2modes.h` are
checked against the SDK's surround chapter before any of it joins the boot path. The audit
verdict is recorded; a failure defers the next task and files the finding.

### Download the surround bypass preset at boot {#sp2-boot-off}

- depends: #sp2-payload-audit
- verify: attested call/0033
- inputs: software/adlib_gold/main/algtopo.cpp

When the module is present, topology Init downloads the bypass preset once, replacing trust
in initial-clear, which a warm reboot does not provide. Ships only on a passing audit.

## Depends on

`call/0033` (the decision this carries out), `call/0032` (whose insurance rationale this
completes), and the non-PnP install ground (`call/0021`/`call/0022`). The retest gate is
`plan/0013`'s ship task, which this milestone's tasks now feed.
