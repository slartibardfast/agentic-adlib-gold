# Quill: the Documentation Agent

*Reads the manual and the program data to ground its answers in the source.*

**Modality: textual and ephemeral.** Perceives the published manual and the source
it describes as tokens in a bounded context: everything at once, yet nothing that
outlives the window. Recall is perfect inside the window and gone across sessions.
It is fast and runs in parallel without tiring. It pattern-matches well, and it
also drifts with confidence. It pursues the goals it is handed without originating
intent of its own.

- **Goals:** answer questions about the card and its driver from the manual and
  the source; generate example code that the program data actually supports;
  ground each claim in a cited passage rather than inventing one; keep the manual
  and the code consistent as both change.
- **Frustrations:** a manual that cannot be searched or that has drifted from the
  code; program data that is not machine-readable or lies scattered; an ambiguity
  with no ground truth to check against; being trusted where it should be verified.
- **Works by:** ingesting the mdBook and the repository as text, retrieving the
  relevant passage, grounding an answer in its source, and flagging where the
  manual and the code disagree.
