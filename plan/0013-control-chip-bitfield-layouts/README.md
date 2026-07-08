# 0013 Correct the control-chip bitfield layouts

The alpha.11 hardware retest heard FM MIDI but no PCM. The trace plus an audit of every
control-chip bitfield against the SDK PDF's register-map figures found five layouts shifted by
one bit, and the reg-13h shift is the PCM killer: the boot write 0xB3 selected DMA line 3 and
left audio interrupts disabled, so PCM starved while FM (which needs neither DMA nor
interrupts) played. `call/0029` records the audit and the fixes this milestone carries out;
alpha.12 ships them with retest instrumentation sized to answer everything in one operator
session.

## Who

- **Casey** hears PCM from the card for the first time; every dxdiag sound test so far has
  been silent while the music test played.
- **Piotr** gets a checked-build log that localizes any remaining silence (DRQ vs IRQ vs
  analog) in the same run, instead of scheduling another blind round.

## Build sequence

### Fix the five bitfield layouts {#fix-bitfields}

- verify: gcc -o /tmp/wavereg_test tests/wavereg_test.c -I. && /tmp/wavereg_test
- inputs: software/adlib_gold/main/common.h, software/adlib_gold/main/wavereg.h, software/adlib_gold/main/tests/wavereg_test.c

Registers 13h/14h (DMA SEL at D6-D4, AEN at D3, boot write becomes 0x9B), 08h (ST-MONO at
D4-D3, SOURCE at D2-D0, default 0xCE), 00h options (TEL=D4, SUR=D5, SCSI=D6), 11h (FLT0=D1,
FLT1=D0), and the model-ID values (0=2000, 1=1000, 2=2000MC). The `wavereg` test pins the
absolute bytes so a layout regression fails instead of passing shift-relatively.

### Self-heal the persisted OutputMode {#outputmode-selfheal}

- depends: #fix-bitfields
- verify: attested call/0029
- inputs: software/adlib_gold/main/common.cpp

The registry restore replays saved bytes verbatim, and the test machine has 0xC4 persisted
from earlier alphas, which would leave the reg-08h fix inert. Normalize a restored OutputMode
to the fixed layout and preserve only the user's mute bit (the `call/0025` self-heal shape).

### Prime, mask, and instrument the MMA paths {#mma-hardening}

- depends: #fix-bitfields
- verify: attested call/0029
- inputs: software/adlib_gold/main/algwave.cpp

Restore the AIL-attested four-byte FIFO prime before GO on the render DMA paths; mask both
MMA format registers before the reg-13h write so the first-ever AEN enable cannot storm; add
the checked-only ISR counter and RUN-to-PAUSE stop line (ISR count, DMA position, MMA status).

### Ship alpha.12 and retest in one session {#ship-alpha12}

- depends: #outputmode-selfheal, #mma-hardening
- verify: attested operator

Both reproducible builds byte-identical with a valid checksum, `adlibgold.sys.sha256` and both
`.host-software` build stanzas updated, tag `v1.0.0-alpha.12`, host re-pinned, USB packaged
with the one-session protocol: checked build first (dxdiag sound + music tests, DebugView log),
then the free build by ear at about 75 percent master volume (garble A/B for logging overhead
and clipping). The checked log's stop lines answer PCM audibility with built-in localization.

## Depends on

Milestone `0008` (the stereo reference this corrects back toward) and `call/0029`. Related:
`call/0025`/`call/0026` (the earlier volume-decoding lessons this extends to bit layouts) and
the follow-up audits it queues (SP2 preset payloads; reg-13h restore on power-up).
