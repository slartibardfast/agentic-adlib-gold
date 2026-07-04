# The Windows 98 SE page fault is pageable code reached at raised IRQL

A hardware-test write-up for continuation on another host. It records what the
`0028:C0031B0A` blue screen means, the ranked causes a source audit found, and the
remediation, all of which are already open findings in this milestone.

## Symptom

On the first hardware run of the driver on a Gold 1000 under Windows 98 SE, the machine
blue-screened with:

```
A fatal exception 0E has occurred at 0028:C0031B0A
```

The INF recognition problem is resolved in `v1.0.0-alpha.2`, so the device is listed,
selected, and the driver loads and executes. This fault is in the driver's own code, not
in setup.

Reading the fault: exception `0E` is a page fault. Selector `0028` is the ring-0 kernel
code segment, so the fault is in kernel mode. The instruction pointer `C0031B0A` lies in
the Windows 98 VMM arena, which means the kernel pager was on the stack when the fault was
taken. That is the signature of a page fault raised at an IRQL where the pager is not
allowed to run.

## Mechanism

Pageable code compiles into a discardable section that the memory manager can trim from
the working set. Fetching an instruction from a trimmed pageable page is safe only at
`PASSIVE_LEVEL`, where the pager can fault it back in. At `DISPATCH_LEVEL` or above the
pager cannot run, so the fault is unrecoverable and the kernel raises a ring-0 exception
`0E` from inside the VMM. Every candidate below is one instance of this single mechanism:
a routine compiled into the pageable segment is reached from a path that runs at raised
IRQL.

## Ranked causes

From a three-agent source audit of the load path, the IRQL and paging discipline, and the
interrupt path. Confidence and timing differ; the mechanism is identical.

| Cause | Location | When it fires | Confidence | Catalogued |
|---|---|---|---|---|
| Pageable `GetInterruptSync` reached at DISPATCH | `common.cpp:442` (PAGE segment) called from `midi.cpp:1064` (non-paged `Write`) | External MIDI output | High, deterministic | Yes (paged `GetInterruptSync` at DISPATCH) |
| FM `Init` holds a spinlock inside a pageable body | `fmsynth.cpp:958-967` | Driver load / `StartDevice` | Medium | Yes (FM Init paged body under spinlock) |
| Wave `Init` leaves interface pointers uninitialised | `algwave.cpp` Init and destructor | Driver load, only on an early `Init` failure | Low | New (uninitialised-state family) |
| `GetCardModel` pageable without `PAGED_CODE` | `common.cpp:491` | Only if a raised-IRQL caller is added; none today | Low, inert | Yes (`GetCardModel` missing `PAGED_CODE`) |
| Back-pointer teardown clear not synchronised | `algwave.cpp:322`, `midi.cpp:349` | Teardown, narrow race | Low | Yes (folded into the synchronized accessor) |

Two of these match the two plausible crash moments:

- **If the fault is at load** (right after selecting the device), the temporal match is FM
  `Init`. `Init` sits in `#pragma code_seg("PAGE")` yet acquires `m_SpinLock` at
  `fmsynth.cpp:959`, which raises IRQL to `DISPATCH_LEVEL`, then runs the register-zeroing
  loop and `Opl3_BoardReset()` before releasing at `:967`. The board-reset callee is
  already non-paged, so the residual fault is `Init`'s own trailing instructions crossing
  into a trimmed page of the PAGE section while IRQL is raised. That makes it probabilistic
  rather than every-time, which fits a fault seen once on first load.
- **If the fault is on MIDI output**, the deterministic match is `GetInterruptSync`. It is
  a one-line accessor compiled into the PAGE segment and still carrying `PAGED_CODE()`, yet
  `CMiniportMidiStreamUartAdLibGold::Write` (deliberately non-paged, reached at
  `DISPATCH_LEVEL`) calls it at `midi.cpp:1064` before `CallSynchronizedRoutine`. A
  non-paged caller jumping into a trimmed pageable page at raised IRQL faults every time
  that page is out. plan/0008 already prescribes moving this accessor to the non-paged
  segment; the source change was not applied.

The audit explicitly cleared the tempting first-interrupt theory: the ISR is non-paged and
NULL-guards every miniport back-pointer, and `m_pPortBase` is set before `Connect()` and is
only used for port I/O, so a spurious shared IRQ 7 the instant the ISR is connected is
handled safely and does not fault.

## Remediation

Each cause is the same class of defect, so fixing the paging cluster removes the whole
exception-`0E` class whichever one fired. The concrete changes, all local to the driver:

1. `common.cpp` `GetInterruptSync`: move the definition below the non-paged `#pragma
   code_seg()` at `:504` and delete `PAGED_CODE()`. It returns `m_pInterruptSync` and is
   safe at any IRQL.
2. `fmsynth.cpp` `Init`: retire `m_SpinLock` around the register clear and board reset. No
   stream exists and the ISR does not dispatch to the FM miniport during `StartDevice`, so
   nothing races; bank access is already serialised through the adapter's interrupt-sync
   accessor. Run the resident-only work at `PASSIVE_LEVEL` with no lock. If a lock has to
   stay, move `Init` to the non-paged segment instead.
3. `algwave.cpp` `CMiniportWaveCyclicAdLibGold::Init`: set `m_AdapterCommon`,
   `m_ServiceGroup`, and `m_DmaChannel` to `NULL` at the top, before the `QueryInterface`,
   so the destructor's `if (ptr) ptr->Release()` guards hold on any early-exit path.
4. `common.cpp` `GetCardModel`: add `PAGED_CODE()`, or move it to the non-paged segment if a
   raised-IRQL reader is ever intended. It is unreachable today, so this is hygiene.
5. `algwave.cpp:322` and `midi.cpp:349`: route the back-pointer clear through
   `CallSynchronizedRoutine` (mirroring `SyncClearActiveStream`), so the ISR can never
   observe a half-cleared back-pointer.

Each change carries its plan/0008 obligation. The paging changes are dispositioned
`structural` and confirmed by a checked build under page-trim stress; the wave `Init`
change gets a pure guard test.

## Confirming which one fired

Two signals pin the exact cause and are worth gathering on the next run:

- **The crash moment.** Load or install time points at FM `Init`. A crash only when an
  external MIDI application sends output points at `GetInterruptSync`.
- **The checked build.** `adlibgold.chk.sys` (in the release) narrates the driver flow over
  DebugView, and with `PAGED_CODE()` live it breaks with a named assertion at the exact
  routine that is pageable-at-DISPATCH. The assertion identifies the routine behind
  `C0031B0A` and settles which cause fired.

## State at write-up

- Recognition solved: `v1.0.0-alpha.2` INF lists and installs the non-PnP card.
- Driver `d499dce`, free artifact `4744eff6`, checked `2a5ca48c`; host tracks the pin.
- These five changes are the next code work, ahead of the remaining wave findings, because
  they unblock every hardware test.
