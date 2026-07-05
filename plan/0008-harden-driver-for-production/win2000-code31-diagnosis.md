# Diagnosis: Windows 2000 Device Manager Code 31 on the driver

The alpha.8 driver installed on Windows 2000 (the install path the earlier grounding predicted
would work), assigned resources without a conflict, and then showed Device Manager **Code 31**.
This records what Code 31 means on Windows 2000, the ranked causes for this driver, and the
prioritized diagnostic. It is grounded in a host Workflow (five research dimensions plus three
adversarial reviews, two of which refuted and corrected the first-pass reading) and a local
inspection of the driver binary and source. Inline markers like `[1]` point into the citations.

## What Code 31 rules in, and the Windows 2000 catch-all caveat

Code 31 is `CM_PROB_FAILED_ADD` (`0x1F`, the devnode flag `DNF_ADD_FAILED`). Microsoft defines
it as the function driver returning an error from its `AddDevice` routine, or no service listed
for the device in the registry, or the operating system failing to load a dependent device or
filter driver [1][2].

Getting Code 31 rather than a crash is real forward progress. It rules out the earlier walls:
the 9x install fault is gone (Windows 2000 uses a different install path), the install completed
(the devnode exists), resources were assigned (it is not Code 12), and the driver did not fail
at start (a start failure is Code 10, which does exist on Windows 2000 [7]).

The adversarial review corrected the earlier reading. On Windows 2000, **Codes 37 and 39 do not
exist**. `CM_PROB_FAILED_DRIVER_ENTRY` (37) and `CM_PROB_DRIVER_FAILED_LOAD` (39) were added in
Windows XP; a Windows 2000 reference lists Device Manager codes 1 through 33 only [16][17]. On
Windows XP and later, an image that fails to load (an unresolved import) is Code 39 and a
`DriverEntry` that returns failure is Code 37, but on Windows 2000 there is no separate code for
either. So on Windows 2000, **Code 31 is a catch-all** that also covers image-load failure,
unresolved-import failure, and a `DriverEntry` failure, not only an `AddDevice` error. A
documented Windows 2000 case shows a driver importing a kernel symbol absent on that build
(`_aulldvrm`) surfacing as Code 31, found with Dependency Walker [18]. The first-pass reading
that Code 31 rules in a successful load and `DriverEntry` is therefore wrong for Windows 2000.

A local inspection narrows, but does not close, the load class. The alpha.8 binary imports only
Windows 2000 era functions (eleven `portcls.sys` `Pc*` exports plus long-standing `ntoskrnl`
and `HAL` calls, PE subsystem version 5.0). Those symbols are standard on Windows 2000, so a
missing export is unlikely, but the check cannot prove they are all present in the specific,
possibly hand-installed `portcls.sys` and `ntoskrnl.exe` on the target machine. Dependency
Walker against the target's own binaries closes that gap cheaply.

## Ranked causes for this driver

Code 31 is generic on Windows 2000, so no single cause is certain; the diagnostic below
separates them. Ordered by how cheap they are to check.

1. **A registry or service-binding defect on the manually-created root devnode.** The second
   documented branch of Code 31 is no service listed, or more than one service listed [2]. The
   Add/Remove Hardware then Have Disk path, especially if retried, is exactly what leaves a
   stale second `ROOT\MEDIA` devnode or a half-written `Service` value. The INF's `AddService`
   itself is correct and matches the DDK sample [10][11], so the mechanism is sound; whether it
   took on this devnode, and whether a duplicate exists, is unverified.
2. **Stale MEDIA-class upper or lower filters left by a prior audio driver.** Any driver in the
   stack whose `AddDevice` fails marks the whole devnode `FAILED_ADD` [2]. This INF installs
   into the MEDIA class `{4d36e96c-e325-11ce-bfc1-08002be10318}`, and stale class filters are a
   canonical Code 31 sound-card cause [12]. Likely on a machine that has hosted other audio
   drivers.
3. **An image-load or unresolved-import failure against the target's own `portcls.sys`.**
   Reopened by the Windows 2000 catch-all. Screened cheaply with Dependency Walker; if the image
   never loads, the checked build prints nothing at all.
4. **`PcAddAdapterDevice` returned a failure internally** (an `IoCreateDevice`,
   `IoAttachDeviceToDeviceStack`, or device-header allocation failure inside PortCls) [9]. The
   driver's `AddDevice` passes only valid arguments (non-null objects, `MAX_MINIPORTS` of 5, a
   zero device-extension size, all legal), so a failure here is internal to PortCls and needs a
   trigger such as a poisoned stack from cause 2.
5. **The driver's own `AddDevice` returning an error.** Lowest: the routine has no allocation,
   dereference, or conditional before it returns the `PcAddAdapterDevice` result, so there is no
   driver-authored failure path.

Separately, a latent wall waits one phase later. `CAdapterCommon::Init` returns
`STATUS_DEVICE_DOES_NOT_EXIST` when the ports do not answer as a real Ad Lib Gold, and
`StartDevice` propagates it. That runs on start, so it surfaces as Code 10, not Code 31. It is
not the current cause, but it will bite next once the add phase succeeds, and it matters most if
the Windows 2000 machine does not have a real card responding at 388h (for example a virtual
machine). This is tracked as a separate item to make non-fatal or bypassable when start is
reached.

## The prioritized diagnostic

Do the cheap non-code checks first, because a silent trace is ambiguous until they are excluded.

1. **Screen the imports.** Open the installed `adlibgold.sys` in Dependency Walker (or
   `dumpbin /imports`) against the target's actual `portcls.sys`, `ntoskrnl.exe`, and `hal.dll`,
   and confirm every imported symbol is exported by those exact binaries. A missing symbol is the
   load-failure cause (cause 3) and explains a completely silent trace.
2. **Check the registry and files.** Confirm exactly one `HKLM\SYSTEM\CurrentControlSet\Enum\Root\MEDIA\...`
   Ad Lib Gold devnode with `Service=adlibgold`, exactly one `HKLM\SYSTEM\CurrentControlSet\Services\adlibgold`
   with an `ImagePath` pointing at the driver, and that `adlibgold.sys` is present at
   `%SystemRoot%\system32\drivers`. Delete duplicate `ROOT\MEDIA` devnodes from repeated Have-Disk
   installs. Inspect `HKLM\SYSTEM\CurrentControlSet\Control\Class\{4d36e96c-e325-11ce-bfc1-08002be10318}`
   for stale `UpperFilters` or `LowerFilters` and remove them (causes 1 and 2).
3. **Capture the checked-build trace.** Install the checked build (`adlibgold.chk.sys`, built with
   `DBG=1`), which traces `DriverEntry`, `AddDevice`, and `StartDevice` with verbose output [13].
   Run one capture sink only, either Sysinternals DebugView with Capture Kernel and Verbose
   enabled (a Windows 2000 compatible DebugView build) or a serial kernel debugger, never both at
   once, because an attached debugger consumes the debug output that DebugView would show. Retrigger
   the devnode and read the trace against this legend:

   - **Nothing prints at all** points at an image-load or unresolved-import failure (the import
     screen), or a no-service or duplicate-service registry state (the registry check).
   - **`DriverEntry` and `AddDevice` print, but no `StartDevice` line** points at
     `PcAddAdapterDevice` returning a failure (cause 4). To read the exact status, breakpoint the
     `AddDevice` return in a kernel debugger, or add a one-line terse trace of the returned status
     to the driver.
   - **`DriverEntry` prints but `AddDevice` does not** points at a filter above the driver failing
     first (cause 2).
   - **`StartDevice: ports=...` prints** means this is actually Code 10, not Code 31, and the
     `Init` card-detection failure is the real wall.

4. **Corroborate with `%windir%\setupapi.log`**, the single Windows 2000 install log, where a clean
   entry with no marker beside `adlibgold.sys` confirms the copy and service creation succeeded [14].
   Do not rely on System event log entries 7000 or 7026: those fire for boot, system, or auto-start
   services with no device, and this is a demand-start function driver loaded by the PnP manager,
   so they will not carry a useful reason here [15].

## Safe fixes to try before the trace

Two candidate fixes are non-code, reversible, and independent of the trace:

- Remove stale `UpperFilters` or `LowerFilters` under the MEDIA class key and the devnode's own
  hardware key, the canonical Code 31 sound-card remedy (cause 2).
- Delete duplicate `ROOT\MEDIA` devnodes, confirm a single devnode, a single service, and the
  driver file present, then remove and re-add the device once cleanly (cause 1).

Do not change the `Init` card-detection path yet. It is a latent Code 10 wall, not the Code 31
cause, and changing it now fixes nothing visible.

## Update: a reboot was requested and no trace appeared

The checked build was installed on the real GoldLib. Windows 2000 requested a reboot after the
install, a live DebugView capture showed nothing, and the device stayed at Code 31. A second
grounded pass (four research dimensions, two adversarial reviews, both confirming) refined the
reading and corrected an interim guess.

- The reboot request is not evidence that the driver loads only at boot, and a reboot alone
  will not clear Code 31. A pending-restart state has its own code, Code 14
  (`CM_PROB_NEED_RESTART`); the device shows Code 31 (`CM_PROB_FAILED_ADD`), which means a live
  start was already attempted and failed. A demand-start PnP function driver is loaded live by
  the PnP manager regardless of its start type, so it does not defer to boot.
- Every PortCls, ntoskrnl, and HAL function the driver imports is present in Windows 2000 RTM.
  The PortCls support reference marks Windows-XP-only exports with an asterisk, and all eleven
  `Pc` functions the driver binds are un-asterisked, so an unresolved import is ruled out apart
  from the unlikely case of a downlevel or corrupt `portcls.sys` on the specific machine.
- An empty live capture means either the driver image never loaded, so `DriverEntry` never ran,
  or a benign capture miss: the account was not a local Administrator, a kernel debugger on the
  `/DEBUG` boot line stole the output, the capture was armed too late, or the verbose level was
  masked.

The leading concrete suspect is a filename mismatch. The service `ImagePath` is
`%SystemRoot%\system32\drivers\adlibgold.sys`, but the checked build ships as
`adlibgold.chk.sys`. If the checked binary was copied to the drivers folder under its own name,
or the free release was installed instead of the checked build, the traced image is not the one
the service loads, which reproduces the empty-trace-plus-Code-31 signature exactly.

The cheapest decisive checks need no tools and no reboot, from an Administrator command prompt:
`sc query adlibgold` (`STATE 4 RUNNING` means the image loaded and `DriverEntry` returned
success; `STATE 1 STOPPED` matches code that never ran), `sc qc adlibgold` (the
`BINARY_PATH_NAME`), and `dir %SystemRoot%\system32\drivers\adlibgold*.sys` to see which file is
present and its size (the checked build is 78,336 bytes, the free build 57,344 bytes). If the
on-disk file does not match the `ImagePath`, copy the checked build to `adlibgold.sys` and
re-test.

If the file is correct, the decisive trace is DebugView Capture then Log Boot, which loads the
capture helper before the driver at the next boot and shows the buffered output when DebugView
is relaunched after the reboot. Confirm a local Administrator account and no `/DEBUG` on the
boot line first, since either makes the capture silent for a benign reason.

## Update: the image never loads, and a zeroed PE checksum was found

The operator confirmed the checked build (78,336 bytes) at the service `ImagePath`, that kernel
capture works (another driver's boot output appeared while `adlibgold` produced nothing), and
that a Device Manager disable and enable still produced no `adlibgold` trace. The `setupapi.log`
shows a clean install (the file copied to `D:\WINNT\system32\drivers\adlibgold.sys`, the model
matched, the `.NT` section installed), ending in `Device has problem: 31`. So the install is
sound and the image simply never reaches `DriverEntry`. (The Windows install is on `D:`, not
`C:`; earlier `C:\WINNT` paths were wrong.)

Inspecting the binary found the shipped `adlibgold.sys` carried a PE checksum of zero. The
reproducible-build step (`pe_normalize.py`) zeroed the checksum for byte-determinism, discarding
the checksum `link /release` sets so a driver carries one. This is a real build defect, fixed in
`v1.0.0-alpha.9` by recomputing the checksum last over the normalized bytes (`call/0024`,
reproducible and valid, artifact hash `92c480bc` becomes `f2573b6c`).

Whether a zero checksum by itself blocks the Windows 2000 load is not confirmed: the documented
NT loader sources accept a zero as a not-computed sentinel and reject only a non-zero-but-wrong
value (as a hard bugcheck, not a silent Code 31). So the checksum is the cheapest variable to
eliminate, not a proven cure. The `0x80` section alignment was ruled out by direct inspection
(every section satisfies the sub-page invariant `PointerToRawData == VirtualAddress`).

If Code 31 survives the valid-checksum `alpha.9` build, the checksum is exonerated and the next
suspect is import resolution against the target machine's own `ntoskrnl.exe`, `hal.dll`, and
`portcls.sys` export tables: a single missing or renamed export makes the loader fail before
`DriverEntry` with no trace, matching the symptom. Dependency Walker on the target (opening
`adlibgold.sys` against the machine's own binaries) is the decisive test and is provided on the
diagnostic media.

## Citations

1. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/cm-prob-failed-add
2. https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/device-manager-problem-codes
7. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/cm-prob-failed-start
9. https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/portcls/nf-portcls-pcaddadapterdevice
10. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-addservice-directive
11. https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/installing-windows-multimedia-system-support-for-an-audio-adapter
12. https://support.microsoft.com/en-us/topic/error-codes-in-device-manager-in-windows-524e9e89-4dee-8883-0afa-6bca0456324e
13. https://learn.microsoft.com/en-us/sysinternals/downloads/debugview
14. https://learn.microsoft.com/en-us/windows-hardware/drivers/install/setupapi-text-logs
15. https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/system-log-event-id-7000-7026
16. https://mskb.pkisolutions.com/kb/125174
17. https://ftp.zx.net.nz/pub/Patches/ftp.microsoft.com/MISC/KB/en-us/310/123.HTM
18. https://sysprogs.com/w/forums/topic/error-code-31-in-windows-2000-environment/
