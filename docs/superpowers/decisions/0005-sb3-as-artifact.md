# ADR-0005 — `.sb3` as Artifact, Transcripts as Source of Truth

**Status:** Accepted
**Date:** 2026-04-22

## Context

Scratch projects export as `.sb3` — a zip containing `project.json` plus costume and sound assets. From git's perspective this is a binary blob:

- GitHub cannot render diffs between revisions.
- Merge conflicts on `.sb3` files are effectively unresolvable.
- A reviewer cannot read `.sb3` in the web UI to leave inline comments.

But the assignment requires the project on GitHub, and the author requires "solid code with comments" that is reviewable.

## Decision

- Commit the `.sb3` file for **reproducibility** — anyone can download it, open it in Scratch, and run the exact project. Lives in `project/ant-adventure.sb3`.
- Treat human-readable **script transcripts** as the source of truth for review. Every sprite's scripts are documented block-by-block in `transcripts/*.md` with purpose paragraphs, contracts, and comments that mirror the comments inside Scratch.

## Rationale

- **Reviewability.** Reviewers can read and comment on transcripts line by line in GitHub. They cannot do this with `.sb3`.
- **Reproducibility.** Without the `.sb3`, the project is not runnable from a checkout. Both forms are needed.
- **Commit discipline.** Transcripts text-diff cleanly, so changes to scripts produce readable diffs in git history. `.sb3` re-commits show up as binary updates and are accepted as noise.

## Alternatives rejected

- **`.sb3` only.** Fails the "reviewable" bar. GitHub cannot meaningfully render it.
- **Transcripts only.** Fails reproducibility. A reviewer would have to hand-rebuild the project.
- **Auto-extract `project.json` from the `.sb3` and commit it separately.** `project.json` is machine-readable but not human-friendly; it mixes positions, IDs, and opcodes in a way that makes manual review almost as hard as reading the binary. Tried and rejected as a middle ground that is worse than either endpoint.

## Consequences

- The author must keep transcripts in sync with the `.sb3` on every change. This is a discipline cost, acknowledged and accepted.
- Commits that change scripts touch two places: `.sb3` (binary) and the relevant `transcripts/<sprite>.md`. Commit messages call out both locations in the `Where:` section.
- A CI check (future) could extract `project.json` and verify that transcript claims (number of scripts, list of custom blocks, variable names) match the actual project. Out of scope for now.
