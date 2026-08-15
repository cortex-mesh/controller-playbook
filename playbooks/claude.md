# Claude controller playbook

Use this when **Claude** is the Chief of Staff. Review is **Opus**. Implementation is a worker CLI in a **different family** from Opus — commonly Codex. Do not let the implementing session review itself, and do not use Opus as both writer and reviewer.

Pin current model selectors on the controller host before the first dispatch. If `opus` does not resolve to the intended Opus tier, stop rather than silently reviewing with an older tier.

## Model contract

| Role | Default |
| --- | --- |
| Implement | Different family from Opus (default Codex `gpt-5.6-terra` / high) |
| Authoritative review | **Opus**, fresh session, high effort |
| High-risk re-review | Fresh Opus with a risk-specific prompt |

This playbook's reviewer choice overrides any generic default in a launch skill.

## Loop

```text
dispatch precise task/goal
  → worker implements and verifies
  → CI-equivalent check in the worktree; red = do not push
  → worker runs scripts/pr-size-check; over cap → AWAITING SPLIT (do not open a megadiff)
  → draft PR; this-SHA CI green; then COMMENT, AWAITING GATE
  → Claude CoS runs a fresh Opus review
  → REVISE with concrete findings, or GATE
  → serialize merge → staging → live-verify
```

## Dispatch

Before every dispatch: confirm tmux, implementer login, default-branch freshness, and that the host is not already busy. Create a dedicated worktree. Name the next incomplete **worker** phase. Do not dispatch staging.

If the worker is Codex:

```sh
codex -m gpt-5.6-terra \
  -c model_reasoning_effort=high \
  --no-alt-screen \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. In the assigned worktree, run the product repo CI-equivalent check (make check, or the exact commands from that repo CI). Push only if green. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff. COMMENT only after this-SHA CI is green. See playbooks/general.md#cicd."
```

Unattended:

```sh
codex exec -m gpt-5.6-terra \
  -c model_reasoning_effort=high \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete worker phase end-to-end. Skip staging. In the assigned worktree, run the product repo CI-equivalent check (make check, or the exact commands from that repo CI). Push only if green. Run scripts/pr-size-check before opening a draft. Over cap: AWAITING SPLIT, do not open a megadiff. COMMENT only after this-SHA CI is green. See playbooks/general.md#cicd."
```

A Claude CLI worker is only valid if it is **not** the Opus reviewer family. If you implement with Opus, pick another playbook's reviewer. Prefer Codex on this path.

Run the worker in a named tmux session on the worker host. Autonomous scope never includes production, destructive data, paid services, secret creation, or decisions outside the locked goal.

## Opus review

Fresh Opus context against a clean checkout at the current PR head:

Current Claude Code takes `/code-review <level> [target]`. Confirm `claude --version` on the controller before first use. If the binary still documents `--effort`, stop and align the CLI rather than mixing forms.

```sh
claude --model opus --dangerously-skip-permissions -p \
  "/code-review high <PR#>"
```

Capture the verdict to a file and publish it on the existing draft PR. `/code-review` does not, by itself, create the GitHub COMMENT the gate requires:

Tee the verdict outside the repo. Pin `commit_id` to the exact head. Abort if the live head moved. Then:

```sh
gh pr review <PR#> --comment --body-file <verdict.md>
```

The review prompt must name base and head and ask for concrete bugs, regressions, missing tests, and scope-specific risks. Read the verdict. A finding is a claim to verify, not a command to obey.

Every PR needs this review. Schema, migration, auth, tenant-boundary, security, or data-write changes get a second focused Opus pass.

Return `REVISE` with file, line, failure scenario, expected correction, and required proof. After revisions, review the **new** head.

## Gate and merge

Rebase onto the latest default branch. Scan files other tracks touch. If rebase moved the SHA, wait for this-SHA CI (or record that no current-head run exists), then re-run Opus and post a new COMMENT. Confirm the COMMENT covers **this** head. Confirm CI is green on this SHA, or that no current-head run exists. Serialize overlapping merges. Live-verify staging. Production remains human-run.

## Heartbeat

Poll at least every 10 minutes until the track is complete, blocked, or handed off. Check tmux output, git, PR head, CI, and the progress log. Same three-attempt escalation as the [general playbook](general.md). Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED; see [general.md](general.md#heartbeat). Do not go quiet at GATE.
