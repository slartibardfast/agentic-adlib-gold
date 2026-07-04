# 0007 Complete driver, end-to-end

The capstone: the whole driver enumerates and works together on the target, and the
completion goal (`call/0010`) is met. This milestone integrates the subsystems and
dispositions the one lacuna the goal left open.

## Who

- **All three personas.** Casey hears PCM and FM and sets levels; Piotr ships a
  reproducible build; Quill confirms the specs and the manual describe the shipped
  driver.

## Acceptance

- The driver's KS filters enumerate on Windows 98SE with the GoldLib recreation
  (`call/0006`).
- Each subsystem meets its spec: the PCM path (plan/0002, plan/0004), the mixer
  (plan/0005), and the FM synth (plan/0006).
- Both build lanes reproduce `adlibgold.sys` byte-for-byte (`call/0009`), so
  `software --verify-build` is green and the `repro-exempt` (`call/0002`) retires.

## The MIDI UART lacuna

The driver carries a fourth miniport, the MIDI UART (the external MIDI port,
`midi.cpp`), which the completion goal does not name. Disposition it here before the
driver is called complete: either fold it into the end-to-end scope with its own spec,
or record it as deferred with a reason. It is not left silent.

## Depends on

- plan/0002, plan/0004, plan/0005, plan/0006.
