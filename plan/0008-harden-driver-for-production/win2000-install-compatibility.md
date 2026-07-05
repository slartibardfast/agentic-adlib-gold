# Grounding: the driver installs and runs on Windows 2000 as-is

This is the web and DDK grounding for whether the same `adlibgold.sys` plus its INF installs
and runs on Windows 2000, not only Windows 98 SE. It was built to answer that question with
evidence rather than reasoning from the INF alone: five research searches, three adversarial
reviews, and a local inspection of the driver binary. The short answer is yes, with no INF
edit required, and Windows 2000 is the safest of the four target platforms.

The research ran as a host Workflow (five research dimensions plus three adversarial passes);
all three adversarial passes tried to find a Windows 2000 blocker and none did. Inline markers
like `[1]` point into the citation list at the end.

## The question

The 1992 Ad Lib Gold is a non-PnP ISA card. The driver is a WDM/PortCls audio driver built
with the Windows 2000 DDK, intended to target "Windows 98 SE and above." On Windows 98 SE the
manual install faults with a ring-0 blue screen (`0028:C0031B0A`) in the 9x configuration
manager. Can the same package be installed and run on Windows 2000?

## Verdict: yes, as-is

The install path on Windows 2000 is a different subsystem from the one that crashes on 98 SE,
the INF is valid there without a signature or directive change, and the binary is native to
Windows 2000. Four pillars, each documented.

### The INF is valid on Windows 2000 without a change

`Signature="$CHICAGO$"` is not a Windows 2000 blocker. Microsoft's INF Version reference
states that the signature must be `$Windows NT$` or `$Chicago$`, maps both to "All Windows
operating systems," and says Windows does not differentiate between them and "it does not
matter which one" [1]. An INF supplies per-platform install data by appending the `.NT`
extension to its install sections whichever signature it uses [1]. So the suspected
`$Windows NT$` edit is not required, and `$CHICAGO$` with `.NT` sections is the sanctioned
cross-platform pattern.

Windows 2000 selects the `.NT`-decorated sections. The multi-platform reference gives the
precedence: Windows processes a `.nt<arch>` variant if present, else a `.nt` variant, else
the undecorated one [3]. So Windows 2000 reads `[AdLibGold_Device.NT]` while Windows 9x, which
ignores `.NT`, reads the undecorated `[AdLibGold_Device]`. The required Windows 2000 pieces are
present: an install package must carry a `DDInstall.Services` section starting with Windows
2000 [2], supplied here as `[AdLibGold_Device.NT.Services]` with `AddService=adlibgold`;
`DriverVer` is required starting with Windows 2000 [1] and is present; the MEDIA `ClassGUID`
and `Provider` are supplied.

The undecorated `Needs=KS.Registration, WDMAUDIO.Registration` inside the `.NT` section
resolves. Microsoft's own audio install guidance uses that exact undecorated form inside an
`.NTX86` example [5], and a shipping Windows 2000 sound driver (C-Media CMI8338, `WIN2K` folder
`CMEDIA.INF`) uses the identical `Signature="$CHICAGO$"`, `Class=MEDIA`, the same MEDIA
`ClassGUID`, `.NT` sections, and the same undecorated `Needs`. That is this INF's exact pattern
in a package that shipped on and installed on Windows 2000 [4][5][6].

### The binary is native to Windows 2000

WDM drivers are forward-compatible: a driver runs on a Windows version newer than the one it
was built for, and the fragile direction is older [8]. This driver is built with the Windows
2000 DDK, so Windows 2000 is its home platform; the risky direction is downward to 98 SE,
which is exactly where the crash is. Every operating-system component the driver needs ships
on Windows 2000: PortCls, KS, SysAudio, KMixer, WDMAud, the WaveCyclic and Topology and MIDI
miniport support, and slave ISA DMA [7].

A local inspection of `adlibgold.sys` confirms it links nothing newer than Windows 2000. Its
only imports are from `portcls.sys`, `ntoskrnl.exe`, and `HAL.dll`. Every `portcls.sys` import
is a Windows 2000 era function (`PcInitializeAdapterDriver`, `PcAddAdapterDevice`, `PcNewPort`,
`PcNewMiniport`, `PcRegisterSubdevice`, `PcRegisterPhysicalConnection`, `PcNewInterruptSync`,
`PcNewServiceGroup`, `PcNewResourceSublist`, `PcRegisterAdapterPowerManagement`,
`PcNewRegistryKey`), none of the later additions. The `ntoskrnl` and `HAL` imports are all
long-standing kernel and hardware-abstraction calls. The PE optional header declares subsystem
version 5.0 with a native driver subsystem, and version 5.0 is the Windows 2000 kernel line, so
the binary is stamped for it.

### The 98 SE crash cannot occur on Windows 2000

The crash format `0028:C0031B0A` is a Windows 9x construct. Selector `0028` is the 9x ring-0
protected-mode code selector, and the fault address is in the `C0000000` VxD and VMM arena; the
9x kernel emits that fatal-exception string from `VWIN32.VXD` [11][20]. The NT family, including
Windows 2000, runs no VxDs and has no VMM ring-0 arena [11]. It installs through `setupapi` and
the kernel and user-mode PnP managers (`umpnpmgr`) rather than the 9x `setupx` and
`CONFIGMG.VXD`, which do not exist on Windows 2000 [13][15]. A Windows 2000 fatal error comes
from `KeBugCheckEx` as a STOP code naming a module, never as `0028:… in VXD` [12]. So the exact
select-time fault is architecturally impossible on Windows 2000; a Windows 2000 failure instead
surfaces as a Device Manager problem code or an NT STOP bugcheck.

### The resource declaration is honored

For a non-PnP device, the INF `LogConfig` directive creates the basic configuration, and one
resource from each entry is selected and assigned at install time [9]. The `LogConfig`
reference itself ships a Windows 2000 system pattern byte-for-byte like ours: an ESDI hard-disk
controller declares `ConfigPriority=HARDWIRED` with `IOConfig=1f0-1f7(3ff::)` and `IRQConfig=14`
[9]. The `[ALG.LC0]` section (`HARDWIRED`, `IOConfig=388-397(ffff::)`, `IRQConfig=7`,
`DMAConfig=1`) is that documented, shipping form. The `FactDef` section is the factory-default
resource source for a manually installed non-PnP device [10].

## The one real risk on Windows 2000: a resource conflict, not the install

The INF pins a single fixed configuration (I/O `388-397`, IRQ 7, DMA 1) with no alternative
ranges. On a real machine IRQ 7 is the LPT1 and ECP parallel-port line and DMA 1 is a common
ECP and legacy-ISA channel, so if either is already claimed the device installs but starts as
Device Manager Code 12 ("cannot find enough free resources"), with no lower-ranked config to
fall back to, and Windows 2000 does not rebalance resources the way Windows 98 does [14]. The
practical prerequisite is to free IRQ 7, DMA 1, and port `388h` first, for example by disabling
the LPT1 or ECP parallel port in the BIOS and reserving IRQ 7 for a legacy ISA device. This is
honest hardware truth: the card decodes those resources, so a single fixed configuration is
correct, and the remedy for a conflict is to free them, not to lie to the operating system with
alternates the card cannot use.

A second, smaller caveat: the INF carries no `CatalogFile`, so the driver is unsigned and
Windows 2000's default Warn policy raises a dismissable "Digital Signature Not Found" dialog
during install. It is a click-through, not a refusal; only a Block policy would stop it [15].

## Correction to an earlier claim

An earlier answer said to prefer the grounded variant over the `forceconfig` variant on Windows
2000. That was over-stated. On Windows 2000 the two variants barely differ: both declare a
single non-rebalanceable configuration, so neither avoids a Code 12 if a resource is occupied.
A forced configuration only pins the resources more deterministically; it is not more reliable
against a conflict. Either form installs. What decides success is whether IRQ 7, DMA 1, and port
`388h` are free.

## Why this matters: Windows 2000 as the primary validation path

Windows 2000 is the safest of the four targets and sidesteps the exact bug the 98 SE install is
stuck on. Installing there validates the whole driver, that the binary loads, that the wave,
topology, FM, and MIDI subdevices register, and that audio plays, independently of the 9x
manual-install crash. A Windows 2000 failure is a readable Device Manager code, not a blue
screen. For that reason the release ships one INF that serves both platforms (`call/0023`), and
Windows 2000 is the recommended first validation.

## Citations

1. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-version-section
2. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-ddinstall-services-section
3. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/creating-inf-files-for-multiple-platforms-and-operating-systems
4. https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/installing-in-windows
5. https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/installing-core-system-components-for-an-audio-adapter
6. https://contents.driverguide.com/content.php?id=1448162&path=SOUND/C-MEDIA/CMI8338/WIN2K/CMEDIA.INF
7. https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/audio-port-class-driver
8. https://en.wikipedia.org/wiki/Windows_Driver_Model
9. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-logconfig-directive
10. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-ddinstall-factdef-section
11. https://en.wikipedia.org/wiki/VxD
12. https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-kebugcheckex
13. https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/pnp-components
14. https://support.microsoft.com/en-us/topic/error-codes-in-device-manager-in-windows-524e9e89-4dee-8883-0afa-6bca0456324e
15. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/troubleshooting-driver-signing-installation
20. https://www.helpwithwindows.com/techfiles/fatal-0e-errors.html
