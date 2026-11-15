# Claude controller playbook

Use this when **Claude** is the Chief of Staff. Review runs on a fleet node with a different and at-least-as-capable model, preferably a different family from the coder. Implementation is a worker CLI (commonly Codex) or a recorded Claude CoS session ([ADR 0009](../docs/adr/0009-coding-locus-and-review-gate.md)). Do not let the implementing session review itself, and do not use the same model as both writer and reviewer.

Pin current model selectors on the controller host before the first dispatch. If a tier alias does not resolve to the intended current model, stop rather than silently reviewing with an older tier. See the example table as of 2026-11 in [general.md](general.md#model-pins).

## Model contract

| Role | Default |
| --- | --- |
| Implement | `<coder-model>` (often a Codex model in a different family from the Claude reviewer) |
| Authoritative review | `<reviewer-model>` on a fleet node — pin the current most capable Claude model when Claude reviews |
| High-risk re-review | Fresh fleet review with a qualifying model and a risk-specific prompt |

This playbook's reviewer choice overrides any generic default in a launch skill.

## Loop

```text
dispatch precise task/goal
  → worker implements and verifies
  → CI-equivalent check in the worktree; red = do not push
  → worker runs scripts/pr-size-check; over cap → AWAITING SPLIT (do not open a megadiff)
  → draft PR; this-SHA CI green; then COMMENT, AWAITING GATE
  → fleet review with a qualifying model (different, more capable; different family preferred)
  → REVISE with concrete findings, or WAITING ON YOU: merge
  → human merges → staging → live-verify
```

## Dispatch

Before every dispatch: confirm tmux, implementer login, default-branch freshness, and that the host is not already busy. Create a dedicated worktree. Name the next incomplete **worker** phase. Do not dispatch staging.

If the worker is Codex:

```sh
codex -m <coder-model> \
  -c model_reasoning_effort=high \
  --no-alt-screen \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. In the assigned worktree, run the product repo CI-equivalent check (make check, or the exact commands from that repo CI). Push only if green. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff. COMMENT only after this-SHA CI is green. See playbooks/general.md#cicd."
```

Unattended:

```sh
codex exec -m <coder-model> \
  -c model_reasoning_effort=high \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. In the assigned worktree, run the product repo CI-equivalent check (make check, or the exact commands from that repo CI). Push only if green. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff. COMMENT only after this-SHA CI is green. See playbooks/general.md#cicd."
```

If Claude implements, the reviewer must be a different and at-least-as-capable model, preferably a different family (Codex or Grok). Same-family review is last-resort only, with the reason on the PR. Prefer a different-family implementer when Claude is the reviewer.

Run the worker in a named tmux session on the worker host. Autonomous scope never includes production, destructive data, paid services, secret creation, or decisions outside the locked goal.

## Review

Fresh reviewer context on a **fleet node** against a clean checkout at the current PR head. Never review in the coder's session.

Current Claude Code takes `/code-review <level> [target]`. Confirm `claude --version` on the review host before first use. If the binary still documents `--effort`, stop and align the CLI rather than mixing forms. Pin `<reviewer-model>` to the current most capable signed-in Claude model; do not invent a name.

```sh
claude --model <reviewer-model> --dangerously-skip-permissions -p \
  "/code-review high <PR#>"
```

Capture the verdict to a file and publish it on the existing draft PR. `/code-review` does not, by itself, create the GitHub COMMENT the gate requires:

Tee the verdict outside the repo. Pin `commit_id` to the exact head. Abort if the live head moved. Then:

```sh
gh pr review <PR#> --comment --body-file <verdict.md>
```

The review prompt must name base and head and ask for concrete bugs, regressions, missing tests, and scope-specific risks. Read the verdict. A finding is a claim to verify, not a command to obey.

Every PR needs this review. Schema, migration, auth, tenant-boundary, security, or data-write changes get a second focused fleet pass.

Return `REVISE` with file, line, failure scenario, expected correction, and required proof. After revisions, review the **new** head.

## Gate and merge

Rebase onto the latest default branch. Scan files other tracks touch. If rebase moved the SHA, wait for this-SHA CI (or record that no current-head run exists), then re-run the fleet review and post a new COMMENT. Confirm the COMMENT covers **this** head. Confirm CI is green on this SHA, or that no current-head run exists. Mark **only the next PR in merge order** ready, then emit `WAITING ON YOU: merge PR #N`. Leave later PRs draft. Do not merge while waiting. After the human merges, rebase and re-gate the next PR, then live-verify staging. Production remains human-run.

## Heartbeat

Poll at least every 10 minutes until the track is complete, blocked, or handed off. Check tmux output, git, PR head, CI, and the progress log. Same three-attempt escalation as the [general playbook](general.md). Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED; see [general.md](general.md#heartbeat). Do not go quiet at GATE.
