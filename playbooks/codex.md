# Codex controller playbook

Use this when **Codex** is the Chief of Staff. Implementation is Codex CLI. Review is a **fresh** `gpt-5.6-sol` / high session — not the implementing session.

## Model contract

| Role | Required model |
| --- | --- |
| Implement | `gpt-5.6-terra`, `model_reasoning_effort=high` |
| Authoritative review | `gpt-5.6-sol`, `model_reasoning_effort=high` |
| High-risk re-review | Fresh sol/high with a risk-specific prompt |

Pin the explicit model names. Do not float a generic alias, and do not use Opus as the authoritative reviewer on this path.

## Loop

```text
dispatch precise task/goal
  → Terra/high implements and verifies
  → draft PR, AWAITING GATE
  → CoS runs fresh Sol/high review
  → REVISE or GATE
  → serialize merge → staging → live-verify
```

## Dispatch

Steerable:

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

Run in a named tmux session. Verify `codex --version` and `codex login status` before first use on a host. Never implement in the shared default-branch checkout.

For a small task on an existing persistent session: send one precise outcome, press Enter, and verify the pane started. Confirm the session is still Terra/high.

## Review

From a fresh session and a clean worktree against the current PR head:

```sh
codex review --base origin/main \
  -c 'model="gpt-5.6-sol"' \
  -c 'model_reasoning_effort="high"'
```

Post as a GitHub COMMENT if the worker did not already, or if you re-reviewed a new head:

```sh
gh pr review <PR#> --comment --body "<sol/high verdict>"
```

Never Approve. Read the verdict and check material findings against the actual head.

## Worker must

Same handoff as [general.md](general.md): real caller path, repo gates, mutation tests, draft PR, `AWAITING GATE`, no self-merge, no production.

## Gate and merge

Rebase, clobber-check, COMMENT covers this head, CI green on this SHA, serialize merges. After merge, watch default-branch CI and staging. A green PR is not proof that the default branch deployed. Live-verify the endpoint, data effect, or UI.

## Heartbeat

Ten-minute cadence while tracks run. Watchdog may inspect and repeat an authorized steer. It may not redefine scope, invent gates, merge production, or retry an unresolved escalation indefinitely. Three attempts, then escalate.
