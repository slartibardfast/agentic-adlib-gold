# 0001 Import the skeleton and plan the WDM design

This milestone imports the existing Ad Lib Gold driver skeleton (the Where-room
software) into the host's thought and plans the design that carries it to working
sound for the primary persona. It produces a plan, not code: the driver source
already exists and is pinned in `.host-software`.

## Who

- **Primary: Casey** (`cast/casey.md`), the retro gamer who wants authentic Ad Lib
  Gold sound on a Windows 9x machine, installed from the INF and judged by ear. The
  operator has confirmed period hardware is available to test on.
- **Supporting:** Piotr (`cast/piotr.md`) builds and debugs the driver; Quill
  (`cast/quill.md`) keeps the manual and the code consistent.

## What is here: the imported skeleton

The software (`adlib_gold`, pinned at `d898b8f` in `.host-software`) is a WDM
kernel-mode audio driver for the Ad Lib Gold 1000/2000, around nine thousand lines
of C++ adapted from the Windows 2000 DDK audio samples (the SB16 sample is the
primary structural template). Its architecture is an adapter with four PortCls
miniports:

- **Adapter** (`adapter.cpp`) sets up the driver, allocates resources, and starts
  the miniports.
- **Common object** (`common.cpp`) reaches the Control Chip registers through bank
  switching, dispatches the ISR, and shadows the mixer.
- **Topology miniport** (`algtopo.cpp`) presents the Control Chip mixer as a KS
  topology filter.
- **WaveCyclic miniport** (`algwave.cpp`) drives the YMZ263 (MMA) PCM and ADPCM
  digital audio.
- **FM synth miniport** (`fmsynth.cpp`) drives the OPL3 (YMF262) synthesizer.
- **MIDI UART miniport** (`midi.cpp`) drives the YMZ263 (MMA) MIDI port.

The hardware it targets:

- the OPL3 (YMF262) FM synthesizer, the card's signature voice;
- the YMZ263 (MMA) digital audio controller, for PCM, ADPCM, and MIDI;
- an optional YM7128 (SP2) surround processor;
- the MIDI and game port.

The build recipe is the DDK `sources` file, and the install surface is
`adlibgold.inf`. The driver's own architectural notes live in its repository
(`CLAUDE.md`, `doc/wdm.txt`, `doc/sdk.txt`), and the SDK is the published manual.
This milestone references them rather than copying them, so the design and the code
version together.

## The design plan

Carrying the skeleton to authentic sound for Casey breaks into design questions to
settle and increments to build. Each becomes its own milestone or `call/` decision
as it is taken up.

- **Establish what the skeleton already does.** Read each miniport against the SDK
  register interface and the DDK sample it derives from, and record which paths are
  real today and which are still stubbed. That fixes the starting point of Casey's
  path.
- **Define Casey's sound path.** Find the shortest route to audible output: which
  miniport produces the first sound a game or a test tone drives, together with the
  INF and resource wiring an install needs. The FM synth is the natural first
  target, since it is the card's signature voice.
- **Decide the reproducible build lane.** The driver is `repro-exempt` today
  (`call/0002`). Design the pinned WDK toolchain and the `attest-host = windows`
  build that retires the exemption, so Piotr's build reproduces.
- **Plan verification on hardware.** With period hardware available, define the
  acceptance a build must pass. The card is detected, a miniport enumerates, and a
  known reference produces the expected voice. These become behavioural (`.allium`)
  specs in the software repository, where the lane runs.

## Open questions

All resolved during planning:

- Casey's first sound path is the 8-bit PCM path (`call/0005`); the FM voice already
  works.
- Authentic-sound acceptance is an 8-bit PCM test tone (`call/0005`).
- The first test target is Windows 98SE through the WDM path, on a GoldLib recreation
  of the Ad Lib Gold (`call/0006`).

## Specs

Behavioural and timing specs live with the software, in the `adlib_gold`
repository, verified by that repository's CI (see `CLAUDE.md`, "Specs, with the
software they constrain"). This milestone references them by path and pin; it does
not hold them.
