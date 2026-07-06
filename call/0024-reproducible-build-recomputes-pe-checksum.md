# The reproducible build recomputes the PE checksum, it does not zero it

- Status: accepted
- Scope: software
- Date: 2026-07-05
- Amends: call/0009

## Context and Problem Statement

`call/0009` requires the Wine build to produce a byte-identical `adlibgold.sys`. To reach that,
`pe_normalize.py` zeroed the non-deterministic PE header fields: the COFF `TimeDateStamp`, the
debug-directory `TimeDateStamp`, and the PE `CheckSum`. The checksum was zeroed for the same
reason as the timestamps: `link /release` computes it over a file that still contains those
clock-derived timestamps, so the checksum varied build to build.

Zeroing the checksum is a build defect. The linker's `/release` sets a checksum precisely because
the operating system expects a device driver to carry one (the `/release` reference states the
operating system requires the checksum for device drivers), and zeroing it discards that
guarantee. So every build shipped a driver whose header checksum was zero.

What is not established is whether a zero checksum by itself blocks the load on Windows 2000. The
documented NT loader sources (the ReactOS reimplementation of `LdrVerifyImageMatchesChecksum` and
the Windows Research Kit `MmCheckSystemImage`) treat a zero checksum as a not-computed sentinel
and accept it; they fail only a non-zero-but-wrong value, and that failure is a hard bugcheck
(`STATUS_IMAGE_CHECKSUM_MISMATCH`), not a silent Code 31. So on the documented lineage a zero
checksum loads. A Windows-2000-RTM-specific driver policy cannot be excluded without
disassembling the RTM loader, so the zeroed checksum is a leading but unconfirmed suspect for the
Windows 2000 Code 31 (the driver never reaching `DriverEntry`), not a proven cause.

The decision below stands on correctness and reproducibility regardless of that open causal
question: a driver should carry a valid checksum, and zeroing it was a normalization step
over-reaching.

## Decision

`pe_normalize.py` recomputes the PE checksum last, over the already-normalized bytes, rather than
zeroing it. The order matters:

- Zero the COFF `TimeDateStamp` and the debug-directory `TimeDateStamp` first, which removes every
  clock-derived field from the image.
- Then compute the checksum over the resulting bytes with the imagehlp `CheckSumMappedFile`
  algorithm (the 16-bit-folded sum of the file treating the checksum field as zero, plus the file
  length) and write it.

Because the input to the checksum is now fully deterministic, the checksum is identical on every
build, so the byte-identical guarantee of `call/0009` is preserved. The prior build being
byte-identical across runs with a zero checksum is itself the proof that every other byte is
deterministic, so the recomputed checksum is the only field that changes and it changes the same
way every build. A double build confirmed identical bytes with a valid, non-zero checksum.

The general rule this establishes: a reproducibility normalization step may zero a field only if
it is both non-deterministic and not required by the system that consumes the artifact. A field
that is non-deterministic but expected, like the checksum, is made deterministic by normalizing
its inputs and recomputing it, never by zeroing it.

## Consequences

- Good: the artifact is now both byte-reproducible and carries a valid checksum, so it matches
  what `link /release` was meant to produce and removes the one header field the build
  deliberately corrupted. This is correct independent of whether it resolves the Code 31.
- The recorded artifact hash changes: the build now writes the four computed checksum bytes where
  it previously wrote zero, so the `adlibgold.sys` hash differs from the earlier `92c480bc` and is
  re-recorded in `.host-software` and re-pinned.
- Verified two ways: a double build produces identical bytes, and `software --verify-build`
  re-derives the recorded hash. The checksum algorithm is pure integer arithmetic in the hermetic
  post-link step, validated byte-for-byte against a reference, so it is environment-independent.
- Honest limit: this is the cheapest single variable to eliminate, not a confirmed cure. If the
  Windows 2000 Code 31 survives a valid checksum, the checksum is exonerated and the next suspect
  is import resolution against the specific machine's `ntoskrnl.exe`, `hal.dll`, and `portcls.sys`
  export tables, verified with a dependency dump on the target. The `0x80` section alignment was
  ruled out by direct inspection (every section satisfies the sub-page invariant).
- This amends `call/0009`. The byte-identical requirement itself stands; only the normalization
  method changes: the build now recomputes the checksum instead of zeroing it. The fix ships as
  `v1.0.0-alpha.9`.
