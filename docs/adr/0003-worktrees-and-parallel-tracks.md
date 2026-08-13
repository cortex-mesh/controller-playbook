# ADR 0003: Worktrees isolate tracks; parallel only when files do not overlap

- Status: Accepted
- Date: 2026-08-13

## Context

Two agents in one checkout overwrite each other. Two agents on overlapping files produce unreviewable merge conflicts and silent clobbers. A shared default-branch working tree also mixes unrelated dirty state into a track.

## Decision

- One **git worktree** per track, created from the current default-branch tip on the host that owns the track.
- Never implement in the shared default-branch checkout.
- Parallelize tracks only when files, generated interfaces, and merge slots do not overlap.
- If two tracks must touch the same files, run them in series or split the shared contract into an accepted ADR first.
- Prefer spreading tracks across hosts. One laptop may still run serial tracks in separate worktrees.

## Consequences

- Dispatch requires a dependency graph that names overlaps before a wave starts.
- The CoS clobber-checks before merge and rebases onto the latest default branch.
- Tooling that claims to create a worktree in headless mode is not a substitute for `git worktree add`.
- v1 of a strip-gated public repo should not run parallel writers on the same tree.
