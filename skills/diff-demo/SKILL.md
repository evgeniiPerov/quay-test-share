---
name: diff-demo
description: Test fixture for exercising `quay diff`. Not a real skill — it exists so folder-level hub-vs-local comparison can be tested against multiple files without editing a skill anyone depends on. Use when verifying quay diff behaviour end to end.
version: 1.1.1
license: MIT
allowed-tools: [Read, Grep]
---

# diff-demo

A deliberately boring, multi-file skill. Its only job is to be **changed** —
on the hub, locally, or both — so `quay diff` has something to report.

## Why it has several files

`quay diff` compares the whole skill directory. A skill with only `SKILL.md`
cannot exercise the case that matters: `SKILL.md` byte-identical while a
sibling file moved. This one ships three files so that case is reachable.

| File | Role in the test |
|---|---|
| `SKILL.md` | The file a single-file comparison would look at. Leave it alone to prove siblings are still seen. |
| `references/notes.md` | Edit this to test a sibling-only change. |
| `scripts/hello.sh` | Edit or delete to test add/remove marks. |

## Scenarios worth running

1. **Hub moves, you do not** — edit `references/notes.md` on the hub and push.
   Expect `hub is ahead by N commit(s)` and a single `M references/notes.md`.
2. **You move, hub does not** — edit any file locally. Expect
   `direction is unknown`, and no bare `--force` recommendation.
3. **Both move** — expect `direction is unknown` with every changed file listed.
4. **File only on one side** — add a file locally, or delete one on the hub.
   Expect `-` and `+` marks rather than a diff body.

## Cleanup

Nothing here is load-bearing. Reset with `quay add diff-demo --force`.
