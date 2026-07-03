# 0002 Bring up the 8-bit PCM test tone

Play an 8-bit PCM test tone through the WaveCyclic path on the card, the first
increment Casey can judge by ear (`call/0005`). This is a bring-up and verification
milestone: the code path already exists in the skeleton, so the work is to build it,
install it, and prove the tone on hardware.

## Who

- **Primary: Casey** (`cast/casey.md`) hears the tone and judges it.
- **Supporting:** Piotr (`cast/piotr.md`) builds and installs the driver and reads
  the debugger when the tone does not sound.

## What the skeleton already provides

The WaveCyclic miniport for the YMZ263 (MMA) digital audio is implemented in the
software, not stubbed (`software/adlib_gold` at the `.host-software` pin):

- `algwave.cpp` carries the WaveCyclic miniport and stream classes with their
  standard methods written (initialisation, stream creation, format and state
  transitions, the wave ISR, and stream position).
- The 8-bit path is ISA slave DMA: it allocates a DMA channel, sets the MMA format
  register to 8-bit with DMA enabled, and starts the transfer on the run transition.
- The MMA register model and the playback control bits are defined in `algwave.h`.

The path is already written; the target is to prove it. The first audible tone turns
on a correct build and a working install over the hardware wiring the code assumes.

## Acceptance

- The driver builds from the pin and installs from `adlibgold.inf` on the test
  machine.
- The card enumerates its wave render endpoint.
- An 8-bit PCM test tone played to that endpoint is audible from the card.

These become behavioural (`.allium`) specs in the software repository, where the lane
runs; this milestone references them by pin.

## Depends on

- A reproducible WDK build lane that retires the `repro-exempt` (`call/0002`), so the
  build Piotr installs is the pinned one.

## Open question

- Which Windows 9x release and which card revision (Gold 1000 or 2000) is the first
  test target? (Carried from plan/0001.)
