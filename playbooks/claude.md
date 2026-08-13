# Claude controller playbook

Use this when **Claude** is the Chief of Staff. Review is **Opus**. Implementation is a worker CLI — commonly Codex, or Claude CLI if you choose that. Do not let the implementing session review itself.

Pin current model selectors on the controller host before the first dispatch. If `opus` does not resolve to the intended Opus tier, stop rather than silently reviewing with an older tier.

## Model contract

| Role | Default |
| --- | --- |
| Implement | Worker CLI of record for the goal (often Codex `gpt-5.6-terra` / high) |
| Authoritative review | **Opus**, fresh session, `--effort high` |
| High-risk re-review | Fresh Opus with a risk-specific prompt |

This playbook's reviewer choice overrides any generic default in a launch skill.

## Loop

```text
dispatch precise task/goal
  → worker implements and verifies
  → worker opens a draft PR and reports AWAITING GATE
  → Claude CoS runs a fresh Opus review
  → REVISE with concrete findings, or GATE
  → serialize merge → staging → live-verify
```

## Dispatch

Before every dispatch: confirm tmux, implementer login, default-branch freshness, and that the host is not already busy. Create a dedicated worktree. Name the next incomplete phase.

If the worker is Codex:

```sh
codex -m gpt-5.6-terra \
  -c model_reasoning_effort=high \
  --no-alt-screen \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete phase end-to-end."
```

Unattended:

```sh
codex exec -m gpt-5.6-terra \
  -c model_reasoning_effort=high \
  --dangerously-bypass-approvals-and-sandbox \
  -C <absolute-worktree-path> \
  "Read <absolute-goal-prompt-path> and execute the next incomplete phase end-to-end."
```

If the worker is Claude CLI:

```sh
claude --dangerously-skip-permissions -p \
  "Read <absolute-goal-prompt-path> and execute the next incomplete phase end-to-end."
```

Run either shape in a named tmux session on the worker host. Autonomous scope never includes production, destructive data, paid services, secret creation, or decisions outside the locked goal.

## Opus review

Fresh Opus context against a clean checkout at the current PR head:

```sh
claude --model opus --dangerously-skip-permissions -p \
  "/code-review <PR#> --effort high"
```

Capture the verdict and publish it on the existing draft PR. `/code-review` does not, by itself, create the GitHub COMMENT the gate requires:

```sh
gh pr review <PR#> --comment --body "<opus verdict>"
```

The review prompt must name base and head and ask for concrete bugs, regressions, missing tests, and scope-specific risks. Read the verdict. A finding is a claim to verify, not a command to obey.

Every PR needs this review. Schema, migration, auth, tenant-boundary, security, or data-write changes get a second focused Opus pass.

Return `REVISE` with file, line, failure scenario, expected correction, and required proof. After revisions, review the **new** head.

## Gate and merge

Rebase onto the latest default branch. Scan files other tracks touch. If rebase moved the SHA, re-run Opus and post a new COMMENT. Confirm the COMMENT covers **this** head. Confirm CI is green on this SHA. Serialize overlapping merges. Live-verify staging. Production remains human-run.

## Heartbeat

Poll at least every 10 minutes until the track is complete, blocked, or handed off. Check tmux output, git, PR head, CI, and the progress log. Same three-attempt escalation as the [general playbook](general.md).
