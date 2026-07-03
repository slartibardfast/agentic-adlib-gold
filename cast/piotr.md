# Piotr: the Driver Developer

*Builds, debugs, and extends the WDM driver on Windows 2000 and XP.*

**Modality: embodied and kernel-deep.** Works on Windows 2000 and XP with the
period DDK and the v1.01 Developer Toolkit SDK. Reads and writes kernel-mode C++,
loads the driver, and steps through it in a kernel debugger. Holds the WDM audio
topology in mind and cares where every register write lands.

- **Goals:** build the driver from the pinned source with the recorded toolkit;
  understand the audio topology and how synthesis and MIDI move through it; fix or
  extend behaviour without regressing the stack; lean on the SDK and the WDM notes
  rather than guesswork.
- **Frustrations:** an undocumented register or topology assumption; a spec that
  has drifted from the code; a build recipe that will not reproduce on a clean
  machine; losing the reasoning behind an earlier design choice.
- **Works by:** reading the source and the SDK, building and loading the driver,
  stepping through it under a debugger, and recording a decision wherever the next
  developer will ask why.
