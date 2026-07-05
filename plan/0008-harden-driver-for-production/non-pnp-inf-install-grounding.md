# Grounding: how a non-PnP ISA sound card installs on Windows 98 SE

This document is the web and DDK grounding for the Ad Lib Gold install crash, built to
replace guesswork with cited evidence. It draws on the Microsoft INF reference, the Windows
95/98 DDK configuration-manager documentation, and a corpus of real legacy ISA sound-card
INF files. It confirms part of the shipped fix (`call/0021`), refutes part of it, and records
what the evidence can and cannot establish. It corrects
`install-crash-invariant-address.md` and the over-stated claims in `call/0021`; a later
session must read this before touching the install path again.

The research behind it ran seven independent web and DDK searches plus three adversarial
review passes. The full citation list is at the end; inline markers like `[13]` point into it.

## The question

Installing the driver on the GoldLib (Windows 98 SE) faults at `0028:C0031B0A` the instant
the device is selected in the Have-Disk wizard, before any file is copied. The shipped fix
(`call/0021`, alpha.6) put `ConfigPriority=FORCECONFIG` in the INF's `DDInstall.FactDef`
section, pinned I/O `388-38F` with IRQ 7 and DMA 1, and deleted the arbitratable
`NORMAL`-priority `LogConfig` sections. Is that the correct fix?

## The short answer

The fix is on the correct axis but uses the wrong idiom, rests on a refuted premise, and
was recorded with an over-stated cause.

- **Correct axis (proven).** A fault at a byte-identical address across five different
  `adlibgold.sys` binaries is in fixed, statically-loaded ring-0 operating-system code, not
  in the driver. Driver code relocates between builds; a fixed address does not. So the
  remedy belongs in the INF, not in the `.sys` [20][21][22].
- **Wrong idiom (empirical).** The one real INF that installs an Adlib/OPL FM chip at base
  `388` is the Windows 95 `MIDI.INF`. It declares the fixed FM resources with
  `ConfigPriority=HARDWIRED` inside a `LogConfig` section, keeps a `NORMAL` `FactDef`
  alongside, and binds the standard compatible ids. No real shipping Windows 9x sound INF
  uses `FORCECONFIG` anywhere [13][14][15].
- **Refuted premise (documented).** No Microsoft source states that a manually-installed
  non-PnP device requires a forced boot configuration to install. Real Windows 98 SE non-PnP
  INF files install with arbitratable `HARDRECONFIG` or `NORMAL` `LogConfig` sections and no
  `FactDef` at all [16][17].
- **Over-stated cause (inference).** Nothing in the primary sources ties an arbitratable
  `LogConfig`, or the absence of a forced config, to a page fault at select time. The
  attribution to the configuration manager (CONFIGMG) is a leading hypothesis, not an
  established fact, because the Windows 9x crash screen names the faulting driver and that
  name was never captured.

## What the invariant address proves, and what it does not

Exception `0E` is a page fault. Selector `0028` is the Windows 9x flat ring-0 code segment
[20]. The linear address `C0031B0A` sits in the `C0000000` and upward system arena where
`VMM32.VXD` and every statically-loaded system driver live [21]. The configuration manager
is packed into `VMM32.VXD`, so a fault there is physically consistent with an
operating-system configuration routine failing during install [21].

The single firm result is that the fault is in fixed operating-system code reached during
install, not in `adlibgold.sys`. That is what makes an INF-only remedy correct in principle.

The address alone does not name the module. Real Windows 9x crash logs show the `C003xxxx`
band belonging to different drivers depending on the machine's `VMM32.VXD` composition: one
system placed the file system driver at `C003BD1E` [22]. The Windows 98 crash screen normally
prints the faulting module by name, in the form `in VXD CONFIGMG(nn) + offset` or
`in VXD VMM(nn) + offset`. That name was not captured for this crash, so the evidence in hand
cannot distinguish the configuration manager from the virtual machine manager acting as a
heap helper called from the configuration path. Capturing that line on the next attempt is
the highest-value diagnostic available.

## The ConfigPriority vocabulary

The complete, sourced token set for a Windows 9x INF resource-configuration section:

| Token | Documented in | Meaning |
|---|---|---|
| `DESIRED` | LogConfig [2] | The optimal configuration |
| `NORMAL` | LogConfig [2], FactDef [1] | The typical soft-configured setting; the manager may reassign it |
| `SUBOPTIMAL` | LogConfig [2] | Below the normal setting |
| `HARDRECONFIG` | LogConfig [2] | Changing it requires a jumper change |
| `HARDWIRED` | LogConfig [2] | The value cannot be changed, though the manager still ranks it |
| `RESTART`, `REBOOT`, `POWEROFF`, `DISABLED` | LogConfig [2] | State-transition priorities |
| `FORCECONFIG` | FactDef [1] | A forced configuration the manager must assign |

The Windows 95 DDK lists the matching configuration-manager constants
(`LCPRI_FORCECONFIG`, `LCPRI_HARDWIRED`, `LCPRI_NORMAL`, and so on) that the INF tokens map
onto, so `FORCECONFIG` is native to the Windows 9x configuration manager, not an NT-only
invention [3].

Two distinctions matter.

`HARDWIRED` versus `FORCECONFIG`. `HARDWIRED` means the resource value cannot be changed, but
it remains one ranked candidate the installer arbitrates among and may pass over on a
conflict [2]. `FORCECONFIG` is imposed: the manager must assign exactly it [1]. To stop
arbitration entirely, `FORCECONFIG` semantics are what an author wants. The operating
system's own worked example of pinning a fixed legacy ISA resource set (an ESDI hard-disk
controller) nonetheless uses `HARDWIRED` in a `LogConfig`, not `FORCECONFIG` [2].

A documentation asymmetry. The current Microsoft reference lists `FORCECONFIG` as a valid
`ConfigPriority` only in `FactDef` [1]; the `LogConfig` directive reference omits
it [2]. The long-standing driver-author lore is the inverse: that `FORCECONFIG` takes effect
only in a `LogConfig` section and that the ChkInf validator reports an error when it appears
in `FactDef` [10][12]. That lore is scoped to Windows 2000 and the NT-era ChkInf validator,
so it does not bind the Windows 98 SE setupx runtime. The two sources disagree, and no
fetched primary source resolves the disagreement for Windows 98 SE. This is the exact point
the A/B test below is designed to settle. (The token `PALEO`, guessed in an earlier session,
does not exist in any authoritative source.)

## The real-INF corpus

Every excerpt below is from a fetched real INF or the Microsoft reference.

Windows 95 `MIDI.INF`, the closest analogue, installs an Adlib/OPL FM chip at base `388`
[13]:

```
[Generic]           %*PNPB006.DeviceDesc% = MPU401, *PNPB006
[MPU401.FactDef]    ConfigPriority=NORMAL     IOConfig=330-331  IRQConfig=9
[OPL3_Dev.LogConfig] ConfigPriority=HARDWIRED  IOConfig=388-38b
[OPL2_Dev.LogConfig] ConfigPriority=HARDWIRED  IOConfig=388-389
```

The immovable FM range is a `HARDWIRED` `LogConfig`; the MPU-401 keeps a `NORMAL` `FactDef`;
the ids are the standard compatible ids. There is no `FORCECONFIG`.

Creative `SB16AWE.INF`, a real ISA sound card [14]:

```
[PNPB003_Device.FactDef]
ConfigPriority=NORMAL  IOConfig=220-22F ... IOConfig=388-38B  IRQConfig=5  DMAConfig=1  DMAConfig=5
```

`NORMAL` throughout, standard ISA-PnP ids, no `FORCECONFIG`.

ESS AudioDrive `OEMSETUP.INF` [15] independently confirms the same shape: a `NORMAL`
`FactDef` at `220`/`388` plus a set of `NORMAL` (some `HARDRECONFIG`) `LogConfig` sections.

Two real Windows 98 SE non-PnP INF files install with no `FactDef` and no `FORCECONFIG` at
all, which is what refutes the "requires a forced config" premise:

- `SSMCIRDA.INF`, a Windows 98 SE non-PnP infrared port, uses four `LogConfig` sections all
  at `ConfigPriority=HARDRECONFIG`, for example `IOConfig=3e8-3ef(ffff::)`, `IRQConfig=5`,
  `DMAConfig=1`, with no `DDInstall.FactDef` at all [16].
- `MSPORTS.INF`, legacy serial and parallel ports, uses `HARDRECONFIG` `LogConfig` sections
  and no `FORCECONFIG` [17].

The only forced-style configuration in any Microsoft-authored INF in the corpus is the ESDI
example, and it uses `HARDWIRED` in a `LogConfig` [2]:

```
[esdilc1] ConfigPriority=HARDWIRED  IOConfig=1f0-1f7(3ff::)  IoConfig=3f6-3f6(3ff::)  IRQConfig=14
```

The consensus of the corpus is that the idiomatic Windows 9x way to declare a fixed legacy
ISA resource set is a `HARDWIRED` (or `HARDRECONFIG`) `LogConfig`, optionally beside a
`NORMAL` `FactDef`. `FORCECONFIG`-in-`FactDef` appears in no real shipping INF, only in the
`FactDef` reference's own minimal example [1].

## The manual install path and the refuted premise

On the Have-Disk path, Windows generates the device instance from the INF `[Models]`
hardware id when the installation is manual [5], under the root enumerator that acts as the
bus driver for non-PnP drivers. Root-enumerated devices can be created with a
driver-supplied hardware id [6], so a synthetic id such as `ALG1000` is a legal, installable
binding. It does not need a real compatible id, and it is therefore not the crash cause.

For a non-PnP or root-enumerated device the `LogConfig` and `FactDef` records are the live
resource path (they are ignored for true PnP devices) [2], and the system installer builds
binary logical-configuration records from them and stores them in the registry [2]. So for
this card the resource-descriptor path is the one part of INF processing that reaches the
configuration manager at select time, which is consistent with a fault there being resolved
by rewriting those descriptors, whatever the descriptors say.

The claim that a manually-installed non-PnP root device requires a forced boot configuration
to install is not documented. The `FactDef` reference presents `FORCECONFIG` as one of ten
`ConfigPriority` choices, not a requirement, and it recommends that a card whose factory
settings can be changed should also use the `LogConfig` directive [1]. Real non-PnP INF files
install without any forced config [16][17]. As literally stated in `call/0021`, the premise
is refuted.

The `IoReportDetectedDevice` root-enumeration alternative, which an earlier session held in
reserve as a fallback, is available only starting with Windows 2000 [7]. It does not exist on
Windows 98. On Windows 98 SE the INF-driven configuration-manager path is the only mechanism,
so the fix is on the correct axis and the fallback does not apply to the target platform.

## The WDM audio INF specifics on Windows 98 SE

A dual-platform WDM audio INF carries an undecorated Windows 9x install section (matched by
`Signature="$CHICAGO$"`) and a decorated `.NT` section [8][18][19]. Both wire the fixed KS and
WDM-audio infrastructure, by different directives:

- The Windows 9x, OEM-disk form is `AlsoInstall=KS.Registration(ks.inf),
  WDMAUDIO.Registration(wdmaudio.inf)`. The DDK AC97 sample comments that this form is used
  for an OEM-distributed disk, which is why `AlsoInstall` is used instead of `Needs` and
  `Include` [19].
- The NT form is `Include=ks.inf,wdmaudio.inf` with `Needs=KS.Registration,
  WDMAUDIO.Registration` plus a `.Services` section with `AddService` [8][19].

The `KS.Registration` and `WDMAUDIO.Registration` sections contain only `AddReg` and
`CopyFiles`; they declare no `IOConfig`, `IRQConfig`, or `DMAConfig` [9]. Two consequences
follow. The crash-relevant resource descriptors must come from the install section's own
`LogConfig` or `FactDef`, and adding a `LogConfig` cannot break KS registration. Every real
Windows 98 SE WDM audio INF examined is for a PCI card and carries no `LogConfig` or `FactDef`
at all [18][19], so a WDM audio INF that combines KS registration with an ISA resource
declaration has no matching audio exemplar to copy. Its correctness rests on the general
`FactDef` and `LogConfig` reference, not on a WDM audio precedent.

## The resource range was too small

Separate from the priority-token question, the shipped INF under-claims the card's I/O. The
driver's own architecture document records that the MMA chip (YMZ263) occupies `base+00h`
through `base+0Eh`, and that the card decodes sixteen ports starting at `388h` by default.
The shipped INF claimed `IOConfig=388-38F`, which is eight ports and stops at `base+07h`. It
does not cover the MMA data, format, and MIDI ports at `base+08h` through `base+0Eh` that the
wave miniport and the MIDI miniport read and write. The correct range is `388-397` (sixteen
ports). This
under-claim is a latent defect that would bite after install even if it does not contribute
to the select-time fault, so the grounded INF widens the range.

## The grounded INF structure

Grounded in the corpus (chiefly `MIDI.INF` [13]) and the Microsoft references [1][2][8]. It
declares the fixed resources as a `HARDWIRED` `LogConfig`, keeps a `NORMAL` `FactDef`
alongside rather than deleting arbitration, widens the I/O range to the card's real footprint,
and retains the `AlsoInstall` registration form.

```
[AdLibGold_Device]
AlsoInstall=KS.Registration(ks.inf),WDMAUDIO.Registration(wdmaudio.inf)
CopyFiles=AdLibGold.CopyList
AddReg=AdLibGold.AddReg
LogConfig=ALG.LC0

[ALG.LC0]
ConfigPriority=HARDWIRED
IOConfig=388-397(ffff::)
IRQConfig=7
DMAConfig=1

[AdLibGold_Device.FactDef]
ConfigPriority=NORMAL
IOConfig=388-397
IRQConfig=7
DMAConfig=1
```

Rationale:

- `HARDWIRED` `LogConfig` rather than `FORCECONFIG` `FactDef` is the documented and
  empirically-used shape for an immovable legacy ISA resource set [2][13], and it avoids the
  ChkInf hazard reported for `FORCECONFIG`-in-`FactDef` [10][12].
- Keeping a `NORMAL` `FactDef` mirrors `MIDI.INF` [13] and follows the reference's advice that
  a software-selectable card should also use `LogConfig` [1]. The Ad Lib Gold selects its I/O,
  IRQ, and DMA in software, so deleting all arbitratable configs was more restrictive than
  documented practice.
- The 16-bit decode mask `(ffff::)` is the conventional real-ISA form [16].
- The `AlsoInstall` registration path declares no resources and is orthogonal to the fix [9].

A documented alternative, not taken in the primary build, is to bind the FM function to the
standard compatible ids (`*PNPB005` for the OPL2 FM, `*PNPB020` for OPL3, `*PNPB006` for the
MPU-401 interface [23][24]) as `MIDI.INF` does. It is not taken because Windows ships inbox
drivers for those ids, and binding them to this aggregate WDM adapter risks the inbox OPL3
MIDI driver being ranked over this one. The earlier premise that Adlib is `*PNPB003` was
wrong: `*PNPB003` is a Sound Blaster id.

## The A/B experiment

The documented and empirical sources disagree on whether the forced resource declaration
belongs in `FactDef` as `FORCECONFIG` or in a `LogConfig` as `HARDWIRED`. Only the hardware
can settle it. Because each hardware test is an expensive round-trip, the release ships both
variants so one session distinguishes them:

- The grounded variant (`HARDWIRED` `LogConfig` plus `NORMAL` `FactDef`, widened range) is the
  primary, tried first.
- The `FORCECONFIG` `FactDef` variant (the alpha.6 INF, with the range corrected) is the
  alternate, tried only if the grounded variant still faults.

Both variants use the identical `adlibgold.sys`, so the test isolates the INF. If either
variant still faults at `0028:C0031B0A`, the operator captures the `in VXD NAME(nn) + offset`
line from the crash screen, which resolves the configuration-manager-versus-manager question
directly.

## Open questions

- Which driver does the crash screen name at `0028:C0031B0A`? The named module and offset
  resolves the attribution. Only the raw address is known so far.
- Does the Windows 98 SE setupx parser honor, reject, or ignore `FORCECONFIG` inside a
  `FactDef`? The ChkInf caveat is an NT-era validator claim [10][12]; a Windows 98 SE runtime
  confirmation is missing. The A/B test answers this.
- Did the crashing INF also carry a malformed resource token or a bad hardware-id binding, so
  that the shipped fix worked incidentally by rewriting that section rather than because
  `FORCECONFIG` was required? A diff of the crashing and fixed INF is the offline check.
- Would binding the standard compatible ids with a `HARDWIRED` `LogConfig` install without the
  crash, and without an inbox-driver ranking conflict? This is the strongest untested
  alternative.

## Provenance

The Microsoft `learn.microsoft.com` INF, `FactDef`, and `LogConfig` pages [1][2] are the
NT-lineage documentation, but their token set matches the Windows 95 DDK configuration-manager
enumeration [3], and OSR confirms that `FactDef` and `LogConfig` are deprecated while the code
that handles them is still present [12]. The Windows 98 SE runtime behaviour is inferred from
this shared vocabulary, not from a fetched Windows-98-only reference. Two of the three
adversarial review passes did not refute the grounded recommendation; the crash-attribution
pass refuted the causal story specifically, which is why the cause is recorded here as an
inference and the module attribution as a hypothesis.

## Citations

1. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-ddinstall-factdef-section
2. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-logconfig-directive
3. https://library.thedatadungeon.com/msdn-1998-06/win95ddk/HTML/S2506.content.HTM
4. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-ddinstall-section
5. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-models-section
6. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/hardware-ids
7. https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-ioreportdetecteddevice
8. https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/installing-in-windows
9. https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/installing-core-system-components-for-an-audio-adapter
10. http://jdebp.uk/FGA/non-pap-with-wdm.html
11. http://www.44342.com/device-driver-f31-t5089-p1.htm
12. https://community.osr.com/t/driver-development-found-legacy-operation-logconfig-and-factdef/58261
13. https://contents.driverguide.com/content.php?id=133681&path=WIN95/MIDI.INF
14. https://contents.driverguide.com/content.php?id=18452&path=SB16AWE.INF
15. https://contents.driverguide.com/content.php?id=110193&path=OEMSETUP.INF
16. https://contents.driverguide.com/content.php?id=961909&path=FIR%2Fdata1.cab%2FWIN98SE_INF_File%2FSSMCIRDA.INF
17. https://contents.driverguide.com/content.php?id=82519&path=USB/Eusbsupp/MSPORTS.INF
18. https://contents.driverguide.com/content.php?id=97215&path=AUDIO/CS429X/Win98/CWAWDM.INF
19. https://raw.githubusercontent.com/uri247/wdk81/master/AC97%20Driver%20Sample/C%2B%2B/driver/ac97smpl.inf
20. https://www.helpwithwindows.com/techfiles/fatal-0e-errors.html
21. https://www.helpwithwindows.com/techfiles/vmm32.html
22. https://www.experts-exchange.com/questions/10414257/CLEAN-install-has-Fatal-Exception-0E-at-0028-C003BD1E-in-VXD-VFAT-01-00003DF2.html
23. https://raw.githubusercontent.com/linuxhw/IDs/master/isa.ids
24. https://cdn.netbsd.org/pub/NetBSD/NetBSD-current/src/sys/dev/isapnp/isapnpdevs.c
25. https://archive.org/stream/adlib-gold-users-guide/Adlib%20Gold%20Users%20Guide_djvu.txt
