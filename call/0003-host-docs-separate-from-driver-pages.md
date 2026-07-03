# Host docs stay separate from the driver's GitHub Pages

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The `adlib_gold` driver already publishes its manual as a searchable mdBook to
`https://slartibardfast.github.io/adlib_gold/`, built by its own repository's
workflow. The methodology's `host-lifecycle book` also generates an mdBook, of the
host rooms. Publishing both from one place risked clobbering the existing driver
site.

## Decision

Keep the two sites separate. The driver's Pages stays exactly as it is, published
from the driver repository and untouched by this host. This host publishes its own
mdBook of its rooms to its own Pages and links out to the live driver site from the
Where section, rather than moving the driver's book.

## Consequences

- Good: the existing driver documentation URL keeps working, and the driver
  repository needs no change.
- Neutral: the two books publish independently, so a reader opens the driver manual
  through a link on the host site.
