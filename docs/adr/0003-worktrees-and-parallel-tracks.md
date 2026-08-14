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
- One graph node is one mergeable outcome a reviewer can hold in their head. The CoS writes phases before dispatch (living graph and goal prompt). Each phase is one PR. Mapping: phase N → graph node → one PR → one outcome. Size is the backstop; grouping is the method.
- Use a short phased ADR only when the shape of the system changes (new runtime, new auth boundary, new deploy path), so later tracks cannot collapse the phases. Do not ADR a rename, a lockfile bump, or a docs-only fix.
- If a track would touch two or more subsystems, it is already two tracks. Write that split in the living graph before anyone codes.
- Prefer spreading tracks across hosts. One laptop may still run serial tracks in separate worktrees.
- Resolve the worker pool live at every dispatch (and on the 10-minute heartbeat) via `scripts/fleet-preflight`. The pool file is a local name list, not a cached IP inventory. A logged-out implementer is AWAITING LOGIN, not down. An SSH timeout on one host is not "no hosts." Stale tmux panes without a live implementer process are not busy.

## Consequences

- Dispatch requires a dependency graph that names phases, outcomes, overlaps, and blast radius before a wave starts. Soft split happens in the graph, not as a post-hoc justification of a megadiff.
- A worker must not add a second outcome to the same PR.
- The CoS clobber-checks before merge and rebases onto the latest default branch.
- Tooling that claims to create a worktree in headless mode is not a substitute for `git worktree add`.
- v1 of a strip-gated public repo should not run parallel writers on the same tree.
