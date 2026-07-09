# 0016 Notification liveness and stable four-op pairing

The `plan/0013` retest heard PCM loop its prefill and FM MIDI thump at every
four-operator note boundary. The first defect is the interrupt notification leg,
the one part of the wave path the session could not localize because the
DebugView log was not captured; `call/0034` decides to prove it live by model
instead of discriminating it on hardware. The second defect is root-caused in
code: the connection-select register bit for a four-operator pair is flipped
while its release tail still sounds, and the flip splits the voice mid-decay. This
milestone models the notification protocol, hardens the interrupt service
routine to the model's verdict, stabilizes the pairing, and ships both fixes as
one build so a single operator session validates them together.

## Who

- **Casey** hears the dxdiag test sound play through instead of looping its
  first moment, and MIDI music without the thump at every note change.
- **Piotr** gets the interrupt protocol as a checked TLA+ model whose starvation
  counterexamples TLC reproduces, and a pairing invariant pinned by a pure test,
  so neither fix rests on a hunch.

## Design

`spec/NotifyLiveness.tla` models the FIFO service loop, the flag register, the
shared line, the edge-triggered controller, and the service routine's exact read
sequence, with the SDK-unpinned hardware semantics chosen nondeterministically
at initialization (`call/0034`). The committed configuration checks the hardened
routine and must show the refill liveness in every world; the naive routine's
counterexample is reproduced locally and recorded in `MEMORY.md`. The pairing
fix defers the unpair to reallocation: a pair keeps its connection bit through
release, and any split first silences both member channels. The operator
rejected an alternate interrupt-line configuration in the INF; the install keeps
its single hardwired claim.

## Build sequence

### Make TLC runnable at the verify gate {#tlc-gate}

- verify: sh software/adlib_gold/main/spec/tlc.sh ChipTiming
- inputs: software/adlib_gold/main/spec/tlc.sh

A small `spec/tlc.sh` in the driver repository fetches the pinned tla2tools
release (checksum-verified, cached under the user cache directory) and runs TLC
exactly as the CI lane does, so every spec verify line is re-runnable and
re-derivable on this machine. Baseline: the existing timing and bank-access
specs pass locally.

### Model the notification protocol {#notify-spec}

- verify: sh software/adlib_gold/main/spec/tlc.sh NotifyLiveness
- inputs: software/adlib_gold/main/spec/NotifyLiveness.tla, software/adlib_gold/main/spec/NotifyLiveness.cfg

Author the spec through the specula skills. A design constant selects the naive
or hardened service routine; the committed configuration proves the hardened
design refills the buffer infinitely often under weak fairness in every
semantics world, and the naive design's machine-checked starvation trace is
captured before it.

### Harden the service routine to the model's verdict {#isr-hardening}

- verify: gcc -o /tmp/mmastatus_test software/adlib_gold/main/tests/mmastatus_test.c -I software/adlib_gold/main && /tmp/mmastatus_test
- inputs: software/adlib_gold/main/common.cpp, software/adlib_gold/main/algwave.cpp, software/adlib_gold/main/mmastatus.h, software/adlib_gold/main/tests/mmastatus_test.c

What the model required turned out to be a different service clock, not a
hardened read sequence: the DMA transport runs Timer 0 at a 10 ms cadence
(started with GO, stopped with the stream), the ISR re-arms it on each elapse
(the status flags are level-sensitive, so the re-arm is what drops the line for
the next edge) and notifies, and the DMA paths' FIFO threshold interrupts are
masked, since they never fire while auto-initialize DMA keeps the FIFOs fed.
PIO streams keep FIFO-driven service because their FIFOs genuinely drain. Pure
decode extends `mmastatus.h` and its test, `wave.allium` gains the timer service
rule, and the checked-build stop line stays for the session.

### Keep four-op pairs stable through release {#four-op-pairing}

- verify: gcc -o /tmp/fmvoice4op_test software/adlib_gold/main/tests/fmvoice4op_test.c -I software/adlib_gold/main && /tmp/fmvoice4op_test
- inputs: software/adlib_gold/main/fmsynth.cpp, software/adlib_gold/main/tests/fmvoice4op_test.c

Note-off keeps the pair's connection bit and releases only the voice bookkeeping;
the allocator protects only live pairs, so released pairs stay stealable; a new
split helper silences both member channels before clearing the bit; and the
note-on path pre-silences any decaying tails before it connects a pair. The
invariant pinned by the test: a pair's connection bit changes only while both
member channels are silent.

### Ship the hardened build and run one session {#ship-hardened-build}

- depends: #isr-hardening, #four-op-pairing
- verify: attested operator

Version bump, both reproducible builds byte-identical with a valid checksum, the
`.host-software` build stanzas and pin updated, an annotated tag pushed, and the
USB packaged with a plain-ASCII readme. One session answers both fixes; if PCM
still loops its prefill, the surviving hypothesis is physical delivery of the
selected interrupt line (`call/0034`).

## Depends on

`plan/0013` (the retest session this answers, via `plan/0013#ship-retest-build`),
`call/0034` (the prove-by-model decision), and `call/0013` (the timing lane the
new spec joins). The pairing defect traces to `plan/0009`'s dynamic connection
management.
