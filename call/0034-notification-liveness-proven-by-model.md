# The notification path is proven live by model, not discriminated on hardware

- Status: accepted
- Scope: software
- Date: 2026-07-09

## Context and Problem Statement

The `plan/0013` retest heard PCM for the first time, looping its prefill: Windows
fills the cyclic buffer once, the auto-initialize DMA replays it, and the refill
never comes. That sound proves every layer below the interrupt leg (the reg-13h
write, the DMA line, the FIFO, the DAC, the analog path) and leaves only the
notification chain suspect: the card's interrupt, its delivery, and the portcls
copy. The checked build
carries a stop line that would localize the break, but the DebugView log from the
session was not captured, hardware sessions are scarce, and the standing
constraint is to minimize them.

The code audit cleared every static suspect: the IRQ select encoding matches the
SDK table, the claim gate honours the active-low status flags, and the boot masks
leave the sampling FIFO interrupt enabled. What stays uncertain is hardware
semantics the SDK does not pin down: whether the MMA's interrupt flags are
latched events cleared by a status read or level conditions that stand while the
FIFO sits below its threshold, and when the shared interrupt line drops. What is
certain is the bus: the ISA interrupt controller in edge mode delivers only on a
rising edge. Each uncertain semantics admits a starvation trace consistent with
the retest. A level condition holds the line high, so at most one interrupt is
ever delivered. A latched flag that re-sets inside the service routine leaves the
line asserted across the return, so no fresh edge ever arrives. Either way the
notification dies while the DMA plays on, which is exactly what was heard.

## Decision

Prove the notification loop live by model instead of discriminating the failure
on hardware. A TLA+ spec (`spec/NotifyLiveness.tla`, in the driver's existing
specula lane per `call/0013`) models the FIFO service loop, the flag register,
the shared line, the edge-triggered controller, and the service routine's exact
read sequence. The unknown hardware semantics are chosen nondeterministically at
initialization, so a single TLC run covers every world, and the checked property
is liveness under weak fairness: the cyclic buffer is refilled infinitely often
while the stream runs.

The service routine is then hardened until that property holds in every world: a
bounded drain that re-reads the MMA status until no flag is pending, and a
mask-then-unmask of the FIFO interrupt so the line demonstrably drops and a fresh
edge is regenerated. The naive routine's machine-checked counterexample and the
hardened routine's passing run are recorded with the `plan/0016` milestone that
carries the work.

## Consequences

- Good: the fix does not depend on discriminating the failure on hardware, so a
  single retest validates it, and the interrupt protocol becomes a checked spec
  the `plan/0014` timer-notification design will re-use.
- Cost: the service routine does more work per interrupt (a bounded re-read of
  the status register and two format-register writes).
- Residue: if the hardened build still loops its prefill, the surviving
  hypothesis is the physical delivery of the selected interrupt line on that
  machine, addressed by a deliberate INF change to another line. The operator
  rejected carrying an alternate configuration in the INF; the install keeps its
  single hardwired claim, and the driver already programs whatever line the
  resource list carries.
