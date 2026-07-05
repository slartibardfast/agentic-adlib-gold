# 0008 Harden the driver for production, each fix locked behind a spec obligation

An adversarially-verified review of the driver (a thirteen-dimension multi-agent
pass, every finding refuted-or-confirmed against the source) surfaced twenty-six
real defects: one critical, eight high, four medium, thirteen low. They cluster on
exactly the two axes the design named as highest-risk, the shared-port bank switch
and the digital-audio service loop, plus a family of uninitialised-state and
error-swallowing bugs. This milestone remediates all of them, and it does so
spec-first: no fix lands without a regression obligation that makes the defect
impossible to reintroduce silently.

## Who

- **Casey** stops hearing corruption. Today a volume change during playback can
  scribble on an OPL3 voice, and the default output format plays for a moment and
  then goes silent. Both are fixed here.
- **Piotr** gets a driver whose invariants are written down and checked, so a later
  change that breaks one fails the build rather than the hardware.
- **Quill** gets specs that describe what the driver actually guarantees, and they
  now cover the MIDI UART that had no spec before.

## The rule for this milestone: every fix carries a discharged obligation

The methodology's no-hollow-green rule governs the whole milestone. For each defect,
the obligation is authored where it can actually be checked:

- A **pure decision** (a register-value computation, an initial-state rule, a
  bounds check, a validation table) is extracted into a header and unit-tested; the
  obligation is a `test:` disposition in the owning spec's `.obligations`, linked to
  the exercising symbol.
- A **concurrency property** (the shared bank-switch mutual exclusion) is modelled
  in a `.tla` spec and TLC-checked in the Specula lane.
- A **behaviour rule** goes into the owning `.allium` spec (`wave`, `topology`,
  `fmsynth`, or a new `midi`), and `allium plan` derives its obligation.
- A property that only the running driver or a checked build can prove (paging
  legality, the run-state service liveness) gets an honest `attested`/`structural`
  disposition that names how it is confirmed, never a silent pass.

`allium check` + `analyse` + `plan` + `obligations`, the Specula TLC lane, and the
userspace test lane all stay green, and the reproducible build stays byte-identical.

## The two show-stoppers, remediated first

### Serialize every shared base+2/base+3 access

The critical finding. `ControlRegWrite`, the OPL3 array-1 path inside `WriteOPL3`,
`WriteSurroundReg`, and both EEPROM paths drive the shared control/OPL3 ports with a
raw enable/write/restore sequence, while the interrupt service routine flips the same
ports at device IRQL. `CallSynchronizedRoutine` is used in exactly one place in the
whole driver. Any of those writes concurrent with active audio or MIDI lets the ISR
land mid-sequence, and the following bytes reach the wrong chip.

- **Fix.** Add one synchronized accessor in `CAdapterCommon` that owns all base+2 and
  base+3 traffic, and route `ControlRegWrite`, `EnableControlBank`,
  `EnableOPL3Bank1`, the array-1 half of `WriteOPL3`, `WriteSurroundReg`,
  `SaveToEEPROM`, `RestoreFromEEPROM`, and every topology property SET handler
  through `m_pInterruptSync->CallSynchronizedRoutine`. The ISR, already at device
  IRQL, calls the inner routine directly. The multi-millisecond EEPROM stall is kept
  outside the held region. Caller discipline and the FM private spinlock are retired.
- **Obligation.** A new Specula spec `spec/BankAccess.tla` models two writer contexts
  and the ISR sharing the bank-select port, with the enable/write/restore guarded by
  the synchronized routine. Its invariant is mutual exclusion: no context observes a
  bank value another context is mid-sequence on. TLC-checked in CI. A companion
  `structural` obligation records that no source path outside the accessor writes
  base+2 or base+3 raw, checked by a source scan.
- **Verify.** `tools/specula` TLC green on `BankAccess.tla`; the source scan finds no
  raw shared-port writes outside the accessor.

### Service the digital-audio FIFO while the stream runs

The default KMixer format is sixteen-bit, and the sixteen-bit path is never serviced
after start: the fill routine runs once as a pre-fill, the drain routine has no
caller, and the ISR notifies without re-entering either. Sixteen-bit playback emits a
few milliseconds and then underruns; capture records silence; the reported position
freezes.

- **Fix.** Give the wave miniport a pointer to the active render and capture stream,
  set in `NewStream` and cleared in the stream destructor through the interrupt-sync
  routine. From `ServiceWaveISR`, gated on the correctly-read sampling bit, call the
  stream fill routine for a playback request and the drain routine for a capture
  request. Track the true service position for the position query.
- **Obligation.** A `wave.allium` liveness rule: a stream in the running transport
  state advances its buffer position. Because liveness needs the running driver, this
  rule is dispositioned `attested`, discharged by the loopback bring-up on hardware
  (the acceptance below) and by a service-loop unit test that drives the pure fill
  bookkeeping. The pure part, the position-advance arithmetic over the cyclic buffer,
  is extracted and unit-tested (`test:`).
- **Verify.** The position-advance unit test passes; on hardware a running sixteen-bit
  stream advances position and plays continuously.

## The remaining high-severity remediations

### Program the channel count into the sampling hardware

`m_Stereo` is recorded but never programmed: the play and format registers are built
the same for mono and stereo, so the interleave bit is never set. Fix by deriving the
play-register channel bits and the format-register interleave bit from the stream
channel count. Extract that mapping into a pure, unit-tested register-config helper
(the driver's real register logic, distinct from the removed tone experiment). The
obligation is a `wave.allium` rule that a format's channel count determines the
programmed routing, discharged by the helper's `test:`.

Done in two steps. First, mono-only: consulting the manual showed stereo needs the
dual-channel interleaved DMA path and channel-1 access the driver lacked, so the pin
advertised mono and the helper's stereo mapping was defined but unreached (`call/0016`).
Then, full stereo: a register-by-register trace of the Miles AIL driver, checked against
the manual (`stereo-mma-reference.md`), resolved the channel-1 addressing. Stereo now runs
over the DMA interleave path with both channels' play and format registers programmed,
interleave set on channel 0, and channel-1 access via `WriteMMA1` (`call/0017`, which
supersedes `call/0016`). The channel-to-output mapping is attested on the GoldLib.

### Initialise all miniport and stream state

The FM miniport reads `m_fStreamExists` before it is ever written, and the FM stream
leaves its voice table, sustain, patch, and pitch-bend arrays uninitialised. The DDK
placement `new` does not zero pool, proven inside the same class by its manual clear
of the shadow register array. On a free build this silences the synth or hangs notes.
Fix by zeroing the scalar members and the arrays in both init paths. The obligation
is an `fmsynth.allium` rule that a freshly created stream starts with every voice
free, no sustain held, and a defined patch and bend, discharged by extending the pure
`fmvoice.h` with a tested reset that the stream init calls.

### Give the MIDI transmit path flow control, and give the MIDI UART a spec

The transmit routine writes up to sixteen bytes with no read of the transmit-ready
status, so it overruns the sixteen-deep hardware FIFO on any message longer than the
FIFO and drops bytes. The dead-man retry counter is unreachable as a result. Fix by
polling the transmit-ready bit before each byte within a bounded budget, writing only
while space remains, and returning the true byte count so the port driver retries the
remainder. This also closes the MIDI UART spec lacuna carried from plan/0007. Author
`spec/midi.allium` with the receive ring-buffer invariant (no byte lost until full)
and the transmit flow-control rule (write only while space remains, report the true
count), discharged by pure ring-buffer and transmit-decision helpers with `test:`
dispositions, plus a bounded-poll `structural` obligation.

### Fix the paged accessor reached at raised IRQL

`GetInterruptSync` is pageable yet is called from the deliberately non-paged,
device-IRQL MIDI write path, a checked-build bugcheck and a free-build fault if the
page is trimmed. Fix by moving the one-line accessor into the non-paged segment.
Grouped with the other paging corrections below; the obligation is `structural`,
discharged by a checked build with the page-trim stress that exercises the transmit
path, recorded as an attested build step.

### Decode the sampling status correctly (manual-corrected)

Consulting the manual (ch07) while implementing this corrected the review here. The
sampling status is the direct read of base+4 (38CH); the index protocol reads a
register at base+5 and never returns the status, so the ISR's own direct read is
already right. The real defects are that the MIDI service routine reads the status
through the register accessor with index zero, which selects the Test register rather
than the status, and that the status bit constants are mislabeled (0x01 is the
playback FIFO request, not a timer bit; 0x04 for MIDI receive is correct; the MIDI
transmit-empty bit is 0x08). Fix by adding a direct status-read method, correcting the
constants against the manual, having the ISR read the status once and hand the value
to each service routine on its own bit (the playback FIFO bit for the wave path, the
receive bit for MIDI), and having the MIDI service routine read the status through the
direct method. Extract the bit decode into a pure helper and unit-test it; the
obligation is a `test:` in the wave `.obligations`.

## The medium and low remediations, grouped by owning spec

Each carries its obligation. The obligations catalog below is the authoritative list.

- **Timing (`chiptiming.h`, Specula lane).** Add the EEPROM completion delay and the
  default-case settle so control registers outside the volume range, including the SP2
  serial port, get their tabulated delay; the save path waits the full programming
  time rather than a bounded poll. Extend the delay table with the EEPROM entry and
  cover it with the existing timing test; the `ChipTiming.tla` inter-write invariant
  gains the EEPROM case.
- **WaveCyclic (`wave.allium`).** The sampling-frequency field is written with two
  meanings across `NewStream` and `SetFormat`, and full-duplex silently shares one DMA
  channel, buffer, and rate. Fix by computing the hardware rate and resample step in
  one place, and by rejecting the second direction until independent per-direction
  channels exist (or implementing them). The obligation is a `wave.allium` rule that
  full-duplex requires distinct channels, discharged by a pure `NewStream` admission
  test; and the resampler block-size bound (the phase accumulator wrap past sixty-five
  thousand samples) becomes an enforced guard with a `test:`.
- **FM synth (`fmsynth.allium`).** The pitch-bend path writes the block byte before
  the frequency byte, reversed from note-on and audible as a blip; the destructor
  dereferences the adapter pointer without the guard its siblings use; dead DDK drum
  tables remain and the drum patch bank indexing needs confirmation. Fix the write
  order, guard the destructor, and either confirm the note-indexed drum bank and
  remove the dead tables or restore the remap. Obligations: a register-write-order
  `test:` for pitch bend, and an `fmsynth.allium` rule that a note-off never leaves a
  hung voice.
- **Adapter and common (`topology.allium` and code-level).** The mixer-restore return
  propagates the last per-key query status and lets a reset clobber restored settings;
  `StartDevice` aborts the whole device when an optional miniport's resource sublist
  fails, which defeats the non-fatal design; `ControlRegWrite` and the EEPROM paths ignore
  the ready-poll timeout and report success on a busy chip; the DMA channel is
  truncated without the validation the IRQ path has; the ISR back-pointers are cleared
  without interrupt-sync serialization. Fix each. Obligations: a pure mixer-restore
  result test, a pure DMA-channel validation test mirroring the IRQ table, a
  `structural` obligation that the ready-poll timeout is honoured on every hot-path
  write, and the back-pointer clear folded into the same synchronized accessor as the
  bank fix.

## Debug logging brought up to a diagnosable bar

The review's explicit finding: logging is good on the lifecycle paths and absent in
exactly the code that harbours these defects, so a field failure is undiagnosable.
Add, at the right levels and without over-logging the device-IRQL path: the ready-poll
timeout (terse); an ISR spurious-versus-serviced counter and a gated status dump; the
register, value, and bank on each control and OPL3 array-1 write; the node, register,
and value on each topology SET and the surround mode selected; the wave service
invocation, bytes moved, and underrun; and the MIDI bytes-written-versus-requested.
The obligation is `structural`: a source scan confirms a log point exists at each
named hot path and that no unconditional per-interrupt print sits on the ISR path.

## Obligations catalog

Every finding, its location, and the obligation that locks the fix. Discharge kinds:
`test` (pure unit test), `tla` (Specula model), `rule` (allium behaviour rule from a
spec), `structural` (source or build scan), `attested` (confirmed on hardware or a
checked build, recorded).

| Finding (location) | Sev | Owning spec | Obligation | Discharge |
|---|---|---|---|---|
| Shared bank switch not serialized (common.cpp:537) | crit | spec/BankAccess.tla (new) | bank mutual exclusion + no-raw-writers | tla + structural |
| 16-bit path never serviced (algwave.cpp:1357) | high | wave.allium | running stream advances position | test + attested |
| Full-duplex shares one DMA channel (algwave.cpp:980) | high | wave.allium | full-duplex needs distinct channels | rule + test |
| Channel count never programmed (algwave.cpp:1379) | high | wave.allium | channels determine programmed routing | rule + test |
| m_fStreamExists uninitialised (fmsynth.cpp:1008) | high | fmsynth.allium | fresh stream state is defined | rule + test |
| Voice/sustain/patch/bend uninitialised (fmsynth.cpp:1341) | high | fmsynth.allium | fresh stream starts all voices free | rule + test |
| MIDI transmit no flow control (midi.cpp:787) | high | spec/midi.allium (new) | transmit only while FIFO has space | test + structural |
| Paged GetInterruptSync at DISPATCH (common.cpp:419) | high | (code) | non-paged accessor legal at raised IRQL | structural + attested |
| Sampling status decoded wrong: MIDI reads the Test register + mislabeled bits (midi.cpp:659, common.h:86) | high | wave.allium | status via the direct read and correct bits | test |
| FM Init paged body under spinlock (fmsynth.cpp:959) | med | (code) | init runs non-paged | structural + attested |
| EEPROM unsynchronized, holds bank (common.cpp:1049) | med | spec/BankAccess.tla | folded into the bank obligation | tla + structural |
| Sampling-frequency dual meaning (algwave.cpp:975) | med | wave.allium | rate resolved once, before run | test |
| Mixer restore returns wrong status (common.cpp:913) | med | (code) | restore succeeds when the key opened | test |
| EEPROM save omits completion wait (common.cpp:1022) | low | chiptiming.h | EEPROM write waits full cycle | test |
| Settle skipped outside 0x09-0x16 (common.cpp:549) | low | chiptiming.h + ChipTiming.tla | every control write honours a delay | test + tla |
| FM destructor unguarded pointer (fmsynth.cpp:891) | low | fmsynth.allium | teardown guards the adapter pointer | rule |
| ISR back-pointers unsynchronized (common.cpp:92) | low | spec/BankAccess.tla | folded into the synchronized accessor | tla |
| GetCardModel missing PAGED_CODE (common.cpp:462) | low | (code) | pageable code asserts its segment | structural |
| FM Write over-reads a DWORD (fmsynth.cpp:1454) | low | (code) | read only the provided length | test |
| DMA channel not validated (algwave.cpp:547) | low | wave.allium | DMA channel validated like the IRQ | test |
| Resampler wraps past 65536 (wavesrc.h:98) | low | wave.allium | resample bounded to a safe block | test |
| Pitch bend writes B0 before A0 (fmsynth.cpp:1977) | low | fmsynth.allium | note registers written low then high | test |
| Dead DDK drum tables (fmsynth.cpp:37) | low | fmsynth.allium | drum bank indexing confirmed | attested |
| EEPROM ignores ready timeout (common.cpp:1027) | low | (code) | ready timeout reported not swallowed | structural |
| ControlRegWrite ignores ready timeout (common.cpp:540) | low | (code) | ready timeout reported not swallowed | structural |
| StartDevice sublist defeats non-fatal (adapter.cpp:449) | low | (code) | optional miniport failure stays non-fatal | test |
| Debug logging thin on hot paths (multiple) | n/a | (structural) | log point present at each hot path | structural |

## Acceptance

- Every finding is fixed, and every row of the catalog has a discharged obligation:
  the pure tests pass in the userspace lane; `spec/BankAccess.tla` and the extended
  `ChipTiming.tla` are TLC-green; `spec/midi.allium` and the new rules in
  `wave.allium`, `topology.allium`, and `fmsynth.allium` pass `allium check` +
  `analyse` + `plan`; and `host-lifecycle obligations` reports every obligation
  dispositioned with no undischarged behavioural gap.
- The reproducible dual-hosted build still produces `adlibgold.sys` byte-for-byte on
  both lanes, and `host-lifecycle software --check` is green.
- On the GoldLib hardware under Windows 98SE: a sixteen-bit stream plays and captures
  continuously with an advancing position; a mixer change during playback no longer
  corrupts FM output; a long MIDI message transmits without dropped bytes. These are
  the attested obligations, recorded when the hardware bring-up runs.

## Depends on

- The review that produced the findings ledger above.
- plan/0004 (the sixteen-bit path this repairs), plan/0005 (the mixer this protects),
  plan/0006 (the FM synth this initialises), and plan/0007 (whose MIDI UART lacuna
  this closes with `spec/midi.allium`).
