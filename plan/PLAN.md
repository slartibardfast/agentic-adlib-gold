# PLAN: milestone index

The ordering authority. Milestones are listed here in the order they are worked;
their folders are named `NNNN-slug` (zero-padded, and the number is identity, not a
sort key; see `call/0000`). A milestone is a thin, persona-serving increment.

The goal (`call/0010`) is a complete WDM driver end-to-end: the PCM, mixer, and FM
synth subsystems working on Windows 98SE, proven by repeatable dual-hosted builds.

- [0001 import the skeleton and plan the WDM design](0001-import-skeleton-and-plan-design/README.md)
- [0003 establish the reproducible WDK build lane](0003-reproducible-wdk-build-lane/README.md)
- [0002 bring up the 8-bit PCM test tone](0002-bring-up-8bit-pcm-test-tone/README.md)
- [0004 16-bit PCM with high-fidelity downsampling](0004-16bit-pcm-high-fidelity-downsampling/README.md)
- [0005 Control Chip mixer as a topology filter](0005-control-chip-mixer-topology/README.md)
- [0006 FM synth end-to-end under spec](0006-fm-synth-end-to-end/README.md)
- [0007 complete driver, end-to-end](0007-complete-driver-end-to-end/README.md)
- [0008 harden the driver for production and lock each fix behind a spec obligation](0008-harden-driver-for-production/README.md)
- [0009 implement four-operator FM voices from the on-disk timbre bank](0009-implement-four-operator-fm/README.md)
- [0010 full-duplex over the second DMA channel](0010-full-duplex-over-the-second-dma/README.md)
- [0011 one source of truth for the build-artifact hash](0011-single-source-build-artifact-hash/README.md)
- [0012 move the Wine build prefix out of the worktree](0012-wine-prefix-outside-the-worktree/README.md)
