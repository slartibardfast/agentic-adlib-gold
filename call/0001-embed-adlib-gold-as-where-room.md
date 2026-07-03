# Embed adlib_gold as the Where room

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

This host, `agentic-adlib-gold`, governs the `adlib_gold` Windows WDM audio
driver. The methodology requires the software under development to live beneath the
host as a bare store with worktrees (the Where room), never adopted in place and
never a submodule, so that thought and action stay separable and independently
versioned.

## Decision

Embed `adlib_gold` (`https://github.com/slartibardfast/adlib_gold`) as the sole
Where-room component, recorded in `.host-software`:

- Source pinned at `d898b8fe896e3334c482d29947ea2881e8de96f9`, canonical branch
  `main`.
- Materialized locally under `software/adlib_gold/` (the bare store plus the `main`
  worktree), gitignored. The recorded pin is the audit anchor.

## Consequences

- Good: the driver source and the thought that governs it version separately, and
  the pin gives a reproducible checkout without a submodule gitlink.
- Neutral: the worktree is local and untracked, so a fresh clone materializes it
  from the recipe with `host-lifecycle software --materialize`.
