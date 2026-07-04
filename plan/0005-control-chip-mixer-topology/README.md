# 0005 Control Chip mixer as a topology filter

Bring up the Control Chip mixer (algtopo) as a KS topology filter, so Casey sets levels
in sndvol32 and the wave and FM sources route to the speaker. This is the Mixer
subsystem of the completion goal (`call/0010`).

## Who

- **Primary: Casey** (`cast/casey.md`) sets the master volume, the tone, and the mute,
  and balances the wave against the FM source.
- **Supporting:** Piotr (`cast/piotr.md`) maps the KS nodes onto the Control Chip
  registers.

## What the skeleton already provides

`algtopo.h` defines the source pins (wave, FM, aux, mic) into the line-out, and one node
per Control Chip control: the sampling, FM, aux, mic, and master volumes, plus bass,
treble, and mute, each mapped to a register. `common.cpp` shadows the mixer.

The card also carries the YM7128 (SP2) surround processor: `common.cpp` detects it and
reaches it over register `0x18`, but `algtopo.h` exposes no SP2 node. Surfacing it
through the mixer for every documented mode is new work (`call/0012`).

## Spec

`topology.allium` in the adlib_gold repo, distilled from `algtopo.cpp` and extended with
the new SP2 enable and mode nodes, specifies the node set, the register mapping, and the
source routing. The SP2 register-`0x18` bit-serial timing is held by the `.tla` timing
spec (`call/0013`). Both run in that repo's CI with the obligations discharged by tests,
and are referenced here by pin.

## Acceptance

- sndvol32 shows the Ad Lib Gold volume, tone, and mute controls.
- Moving the master volume changes the output level on the GoldLib recreation, and the
  mute silences it.
- The SP2 special-effects modes appear in sndvol32 and switch the effect on the GoldLib
  recreation, for every mode the SDK documents (`call/0012`).

## Depends on

- plan/0003 (the build lane). Independent of the PCM milestones.
