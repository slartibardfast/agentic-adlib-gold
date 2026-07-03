# 0001 example milestone

(Example.) A milestone names the persona it serves, holds its specs under
`spec/`, and is shaped by decisions recorded in the top-level `call/` log.

- **Who**: the primary persona it serves (see `cast/`).
- **What**: the specs in `spec/`: behavioural (`.allium`, via allium) and, where
  the goal turns on timing or concurrency, temporal (`.tla/`, via Specula).
- **Why**: the `call/` decisions that shaped it.

Build it by walking the XP-Persona process (`cast/applying-personas.md`): the
persona's goals and scenarios become the stories here, and their acceptance
criteria become the specs under `spec/`.
