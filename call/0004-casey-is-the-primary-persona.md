# Casey (the retro gamer) is the primary persona

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The project serves three personas (see cast/). The persona workshop
(cast/applying-personas.md, step six) asks for a single primary persona: the one
who must be satisfied and cannot be satisfied by an interface designed for another.
The primary persona takes highest priority and drives how the milestones in plan/
are prioritised.

## Decision

Casey, the retro gamer on Windows 9x, is the primary persona.

The Ad Lib Gold driver exists to make the card produce authentic sound for a player
on period hardware. Casey cannot be served by an interface built for Piotr (the
developer's build-and-debug path) or Quill (the agent's searchable manual): Casey
installs from the INF and judges the result by ear, with no toolchain and no kernel
access. Serving Casey well satisfies the driver's reason to exist, and the
developer and agent personas are served in support of that end.

## Consequences

- Good: planning has a clear priority. A milestone earns its place by moving Casey
  closer to authentic sound on a 9x machine.
- Neutral: Piotr and Quill stay first-class personas; their needs are planned as
  they enable Casey's, not dropped.
- The primary persona can be revisited if the project's intent shifts. This record
  is then superseded rather than edited.
