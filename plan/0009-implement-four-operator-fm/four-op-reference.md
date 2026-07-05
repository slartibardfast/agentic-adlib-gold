# Reference: 4-operator FM on the YMF262 (OPL3), and where the manual and toolkit help

The driver plays General MIDI through 18 two-operator OPL3 voices. The chip can instead
pair two 2-operator channels into one 4-operator voice for richer timbres. This reference
records what the manual and the toolkit provide, so the implementation rests on documented
hardware behaviour and real instrument data rather than a guess. It is the 4-operator
counterpart to the stereo reference.

The headline: 4-operator FM is feasible and well-supported. The register programming is
fully documented in the manual, the driver's note structure already holds four operators,
the four 4-operator algorithms are already coded in `Opl3_CalcVolume`, and a bank of real
4-operator instruments ships on disk in this repository. The DDK sample the driver descends
from simply left 4-operator mode stubbed.

## The register contract (manual ch07)

- **The NEW bit** (Array 1 register 05h, D0; `AD_NEW = 0x105`) enables the OPL3 extended
  features and gates every other Array-1 register. The driver already sets it at init and
  restores it on resume, so this prerequisite is met.
- **Connection-select** (Array 1 register 04h; `AD_CONNECTION = 0x104`) promotes channel
  pairs into 4-operator voices, one bit per pair. The manual's prose heading misprints the
  address as 05h; the register map and the SDK text both place it at 04h, and 05h is the NEW
  register, so the driver constant `0x104` is correct.
- **The six pairs.** Each connection-select bit pairs two channels into one 4-operator voice,
  three pairs per register array:

```host-lint:ignore
  CONNECTION SELECT bit  ->  channels combined  ->  C-register pair (manual ch07)
   D0  ch0+ch3 (Array 0)   0xC0 , 0xC3
   D1  ch1+ch4 (Array 0)   0xC1 , 0xC4
   D2  ch2+ch5 (Array 0)   0xC2 , 0xC5
   D3  ch0+ch3 (Array 1)   0x1C0, 0x1C3
   D4  ch1+ch4 (Array 1)   0x1C1, 0x1C4
   D5  ch2+ch5 (Array 1)   0x1C2, 0x1C5
```

- **The four operators** of a paired voice are the two operators of the first channel plus
  the two of the second, at the driver's existing per-channel operator offsets
  (`gw2OpOffset`). The F-number, key-on and block come from the first channel's A0/B0 only;
  feedback comes from the first C-register only.
- **The four algorithms** are selected by the two CNT bits together (D0 of each paired
  C-register). The four carrier topologies are exactly the cases the driver already computes
  in `Opl3_CalcVolume` (serial; 1 plus a 3-op chain; two 2-op chains; 1 plus a 2-op chain
  plus a carrier), so the volume math is already present and just needs to be reached.
- **The stereo constraint.** The left and right output bits must match across both
  C-registers of a pair, or all four operators mute. The driver already sets both output bits
  on every voice; the 4-operator path applies the same mask to both C-registers.

## The instrument data (toolkit `OPL3.BNK`)

Real 4-operator instrument data ships in this repository, so no bank has to be authored from
nothing:

- `software/adlib_gold/main/manual/src/disks/program-disks-v1.00/installed/OPL3.BNK` is an
  Ad Lib timbre bank of 471 instruments. Of these, 157 are genuine 4-operator instruments,
  marked by the type byte and carrying populated operators three and four. The naming
  convention the manual documents holds: 4-operator sound names begin lower-case (for example
  `drysynt1`, `Tuba1`), 2-operator names upper-case.
- **The record format** (from the toolkit's `FM.ASM`) is 28 bytes: four operators of six
  bytes each in register order 20h, 40h, 60h, 80h, E0h plus a spare; a connect byte carrying
  the first channel's CNT in D0, feedback in D3-D1, and the second channel's CNT in D7; a type
  byte; a transpose byte; a spare. The driver's own note structure carries the same five
  register bytes per operator in the same order, so the conversion drops the spare byte and
  splits the connect byte across the two C-registers.

## What is stubbed in the driver today

The note structure already reserves four operators, so no structural change is needed to hold
4-operator data. The gaps are behavioural, inherited from the DDK sample:

- Connection-select is always written `0x00`, so no voice is ever paired.
- `Opl3_NoteOn` computes a 4-operator flag that is left unused, and its operator loops cover
  only two operators.
- The mode value that indexes `Opl3_CalcVolume` never takes the 4-operator cases.
- `Opl3_FMNote` writes only two operators and only the first channel's play registers.
- There is no 4-operator voice allocator; allocation is over a flat set of 2-operator slots.
- The compiled-in bank is 2-operator: its 4-operator-flagged entries are empty drum slots, so
  the bank must gain converted `OPL3.BNK` data for the paired path to have anything to play.

## What stays attested on hardware

The register sequence and the data format are documented. What the card actually sounds like
is not provable here, which has no card. Three things are confirmed by ear on the GoldLib.
The correctness of each of the four algorithms rests on the manual's figure, which is an
image. Which `OPL3.BNK` timbre each General MIDI program selects is a curation
choice, because the bank is indexed by timbre name and carries no program number. And the
transpose byte's effect on pitch is unverified, since the driver computes pitch itself.
