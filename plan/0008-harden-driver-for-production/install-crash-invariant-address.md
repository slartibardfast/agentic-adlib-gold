# The 0028:C0031B0A crash is in Windows, invariant across builds -- re-analysis

This corrects `page-fault-diagnosis.md`, whose conclusion ("This fault is in the driver's own
code, not in setup") is now disproved. Written to survive a context compact: a later session
must not re-chase driver-code fixes for this crash.

## The decisive evidence: the address does not move

`A fatal exception 0E has occurred at 0028:C0031B0A` appears at the **identical** address on
`v1.0.0-alpha.1`, `.2`, `.3`, `.4`, and `.5`. Those five builds have completely different
`adlibgold.sys` binaries: alpha.3 added five paging fixes, alpha.5 added stereo, the record
gain, 4-operator FM, the MMA timing change, and the INF model matrix. The binary's size,
code layout, and load base all changed.

A fault **in** the driver's own code would move with the code -- a different offset in a
different binary, a different relocated base. An address that never moves is in a **fixed
Windows system routine** (the VMM/setup/config-manager arena at `C0000000+`), reached the
same way every time. The crash is not in `adlibgold.sys`.

Two more facts confirm it is an **install-phase** fault, before the driver ever runs:

- The operator reported the driver `.sys` was **not present in the drivers folder** after the
  attempt -- so `CopyFiles` never ran, so the fault is **before** the file is copied, long
  before `DriverEntry`.
- Neither the alpha.3 paging fixes nor the alpha.4 `AlsoInstall`->`Include`/`Needs` change moved
  the address. Both edited things that only matter after the driver loads; the fault is earlier.

## What the address means

`0E` is a page fault; selector `0028` is ring-0 code; `C0031B0A` is in the shared VMM/system
arena. A Windows setup or config-manager routine (setupx / CONFIGMG / NTKERN) page-faults while
processing the INF, before the driver file is copied.

## Ruled out

- **The driver's code** (paging, IRQL, features): the .sys never loads. `page-fault-diagnosis.md`'s
  pageable-code theory does not apply to a crash that happens before the file is present.
- **The LogConfig resource-descriptor format.** The DDK SB16 INF (this driver's template,
  `doc/wdm.txt` ~27110) is a known-working Win98SE WDM audio INF and uses the **same** forms this
  INF uses: a bare `IOConfig=388-38B` with no count, and **spaces** in the lists
  (`IRQConfig=5 , 7 , 9 , 10`, `DMAConfig=0 , 1 , 3`). So the bare range and the spaces are fine.

## Still open -- the likely triggers, to test on the card

The fault is a Windows setup routine choking on some INF directive processed before `CopyFiles`.
Candidates, to bisect against the working SB16 INF:

1. **The `Include`/`Needs` of `ks.inf` + `wdmaudio.inf`** (`KS.Registration`, `WDMAUDIO.Registration`).
   If Win98SE's copies of those INFs lack the named sections, or the pulled-in directives fault,
   setup dies here. Compare exactly what the SB16 INF includes and needs.
2. **The class / model / provider setup**, or the model-matrix models line, differing from the SB16.
3. **An `AddReg` or interface directive** that references a value setup rejects on 98SE.

## Diagnostic plan (for after the compact)

1. **Bisect the INF against the working SB16 INF.** Diff our install section, `Include`/`Needs`,
   `AddReg`, `[Version]`, and interface sections against `doc/wdm.txt`'s SB16 INF line by line;
   the difference that setup faults on is the bug. This is the highest-value step and needs no
   hardware.
2. **Ship a stripped INF to isolate the directive.** A minimal INF (no `Include`/`Needs`, no
   LogConfig, bare CopyFiles + Services) that installs proves the fault is in what was removed;
   add directives back one group at a time on the card.
3. Only after the install completes does any driver-code question (the old paging analysis)
   become testable.

## Refined finding: the divergence is the non-PnP root devnode, not the KS registration

Reading the INF's own git history against the SB16 sharpens the picture:

- alpha.4 (`fcef658`) replaced `AlsoInstall=ks.registration(ks.inf),wdmaudio.registration(wdmaudio.inf)`
  with `Include`/`Needs`, on the stated premise that the `section(inf)` parenthesis form "is not a
  valid setupx directive." That premise is **false**: the DDK SB16 INF (`WDMPNPB003_Device`,
  `doc/wdm.txt:27036`) uses that exact `AlsoInstall=...(...)` form and installs. The change neither
  caused nor fixed the crash -- alpha.1-3 used `AlsoInstall`, alpha.4-5 use `Include`/`Needs`, and
  the address never moved. So the KS/WDMAUDIO registration form is **not** the crash site.
- alpha.5 (`f742e6e`) added the three-model matrix; the address did not move. So the model matrix
  is **not** the crash site either.

What is invariant across every build, and divergent from the working SB16, is the **enumeration
path**. The SB16's "non-PnP" device (`WDMPNPB003_Device`) is still matched by a real ISA-PnP
hardware id (`*PNPB003`) reported by the ISA-PnP enumerator. The Ad Lib Gold (1992) predates
ISA-PnP and reports nothing, so `d499dce` made it a manually-created **root devnode** with a
synthetic hardware id (`ALG1000`) plus a forced `LogConfig`. A WDM-audio driver bound to a
non-enumerated root devnode with a forced resource config is the combination the DDK sample never
exercises, and the strongest remaining suspect for the config-manager fault.

## Ground truth needed before the next change

Five blind INF ships have each guessed a directive and failed. Pure analysis cannot localise
further; two cheap observations from the card fork the search space decisively:

1. **When does the crash appear?** Selecting the INF in the wizard (before any copy) vs finishing
   the wizard vs the reboot vs first audio use -- this separates the config-manager/setupx phase
   from the driver-load phase.
2. **Is `adlibgold.sys` on disk after the crash?** Present => install completed and the fault is at
   load/start (`DriverEntry`/`StartDevice`); absent => the fault is before `CopyFiles`, in setup.

The next INF or driver change is gated on these two facts, not on a sixth guess.

## Ground truth from the card (operator, alpha.5)

Two observations answered, and they are decisive:

- **The crash fires the instant the driver/INF is selected in the Have-Disk wizard** -- before any
  file copy, and before the wizard's "finishing" (resource-arbitration) step.
- **`adlibgold.sys` is absent** from the drivers folder afterwards -- `CopyFiles` never ran.

So the fault is in the earliest install phase: setupx parses the INF, the operator selects the
device, and CONFIGMG (ring-0) creates the root devnode and **registers its forced `LogConfig`**.
That registration -- not arbitration, not copy, not driver load -- is where it page-faults. This
rules out every later-phase suspect: the KS/WDMAUDIO registration execution, the AddInterface
proxies, the CopyFiles source-disk mapping, and all driver code.

A crucial corollary: **the SB16 sample never validated this path.** The SB16 devnode is
auto-created by the ISA-PnP enumerator matching `*PNPB003`; it is never manually selected as a
non-PnP device, so its (working) INF is not evidence that a manual non-PnP MEDIA install with a
forced `LogConfig` works on 98SE. The suspect is now narrow: **the forced `LogConfig` (or the
whole manual-root-devnode approach) as CONFIGMG registers it at select time.**

## The principle to carry forward

The invariant address is proof: **this is an INF/setup fault, not driver code.** Fix the INF.
Do not ship another driver-code change for this crash. Confirm each INF change installs on the
GoldLib before trusting it.
