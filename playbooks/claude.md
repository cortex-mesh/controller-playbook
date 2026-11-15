# Claude Code as controller

Use this setup when **Claude Code** is the Chief of Staff. Shared rules are in [general.md](general.md). This page is the Claude Code setup: how it dispatches workers, when it may code in its own session, which fleet reviewers pair with it, and its limits.

Review always runs on a fleet node, on a different and at-least-as-capable model, preferably a different family, posted as a GitHub `COMMENT`. Never Approve. Do not let the implementing session review itself, and do not use the same model as both writer and reviewer.

## Dispatching workers

Before every dispatch: confirm tmux, implementer login, default-branch freshness, and that the host is not already busy. Create a dedicated worktree. Name the next incomplete **worker** phase. Do not dispatch staging. Run `scripts/fleet-preflight` against the local pool file; see [general.md](general.md#worker-pool-fill-in). If the chosen path is fleet dispatch and no logged-in host fits, escalate dispatch. Do not silently switch the locus to the controller session.

This setup commonly dispatches a **Codex** worker so the controller can review with Claude Code (different family). It may also dispatch a Grok CLI worker, or implement in a recorded Claude Code session. Launch snippets are in [Worker CLIs](#worker-clis).

```text
dispatch precise task/goal
  → worker implements and verifies (or a recorded internal session)
  → CI-equivalent check in the worktree; red = do not push
  → worker runs scripts/pr-size-check; over cap → AWAITING SPLIT (do not open a megadiff)
  → draft PR; this-SHA CI green; then fleet COMMENT, AWAITING GATE
  → fleet review with a qualifying model (different family preferred)
  → REVISE with concrete findings, or WAITING ON YOU: merge
  → human merges → staging → live-verify
```

Run the worker in a named tmux session on the worker host. Autonomous scope never includes production, destructive data, paid services, secret creation, or decisions outside the locked goal.

## Internal-session coding

Claude Code may implement in its own session only as an explicit, recorded choice ([ADR 0009](../docs/adr/0009-coding-locus-and-review-gate.md)). Isolation still comes from a dedicated worktree, a draft PR, and the repository gate. The session that writes the change does not review it, does not run `gh pr ready`, and does not merge.

When the session exposes a model selector, pin `<coder-model>` to the current intended model. Do not invent a name. If the in-session model cannot be pinned or named, the PR records:

```
Coded in: internal session
Coded by: controller internal session / Claude Code / <model not pinnable>
```

In that case the fleet reviewer is the strongest available qualifying tier (see [Fleet reviewer pairings](#fleet-reviewer-pairings)).

## Fleet reviewer pairings

Prefer a **different model family** from the coder, on a fleet node, at least as capable and preferably more capable:

| Coder | Preferred fleet reviewer |
| --- | --- |
| A Codex or Grok CLI worker dispatched by this controller | Claude Code, strongest available qualifying tier |
| A Claude Code internal session, or a Claude Code worker | Codex or Grok CLI, strongest available qualifying tier |

Pin current model selectors on the review host before the first review. If a tier alias does not resolve to the intended current model, stop rather than silently reviewing with an older tier. See the example table as of 2026-11 in [general.md](general.md#model-pins). Same-family review is last-resort only, must be more capable, and the PR states why no other family was available.

This setup's reviewer pairing overrides any generic default in a launch skill. Never review in the coder's session. Fresh reviewer context against a clean checkout at the current PR head.

Current Claude Code takes `/code-review <level> [target]`. Confirm `claude --version` on the review host before first use. If the binary still documents `--effort`, stop and align the CLI rather than mixing forms. Pin `<reviewer-model>` to the current most capable signed-in Claude model when Claude Code is the reviewer; do not invent a name.

```sh
claude --model <reviewer-model> --dangerously-skip-permissions -p \
  "/code-review high <PR#>"
```

Capture the verdict to a file and publish it on the existing draft PR. `/code-review` does not, by itself, create the GitHub COMMENT the gate requires.

Tee the verdict outside the repo. Pin `commit_id` to the exact head. Abort if the live head moved. Then:

```sh
gh pr review <PR#> --comment --body-file <verdict.md>
```

The review prompt must name base and head and ask for concrete bugs, regressions, missing tests, and scope-specific risks. Read the verdict. A finding is a claim to verify, not a command to obey. For a Codex reviewer, use the review command in [codex.md](codex.md#fleet-reviewer-pairings).

Every PR needs this review. Schema, migration, auth, tenant-boundary, security, or data-write changes get a second focused fleet pass. Return `REVISE` with file, line, failure scenario, expected correction, and required proof. After revisions, review the **new** head.

## Limits

- If a tier alias does not resolve to the intended current model, stop. Do not silently fall back to an older tier.
- If the in-session model cannot be pinned or named, record `<model not pinnable>` and pick the strongest available fleet reviewer tier.
- Prefer a different-family implementer when Claude Code will be the reviewer, so the review stays a different family.
- Shared method rules, coverage, the quality-gate checklist, and the human-only waiver are in [general.md](general.md). The Cursor IDE is not a controller setup; see [general.md](general.md#cursor-ide-is-operator-homework).

## Worker CLIs

### Worker: Codex

Steerable:

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

### Worker: Grok CLI

A Claude Code controller may dispatch a Grok CLI worker on a fleet node. Use the Grok CLI launch commands in [grok.md](grok.md#worker-grok-cli), with a pinned `<coder-model>` and a dedicated worktree. Review then prefers Claude Code (different family).

### Worker: Claude Code

A Claude Code worker is valid when the reviewer is a different and at-least-as-capable model, preferably a different family (Codex or Grok). Same-family review is last-resort only, with the reason on the PR. Pin `<coder-model>` on the worker CLI when the selector is available.

## Gate and merge

Rebase onto the latest default branch. Scan files other tracks touch. If rebase moved the SHA, wait for this-SHA CI (or record that no current-head run exists), then re-run the fleet review and post a new COMMENT. Confirm the COMMENT covers **this** head. Confirm CI is green on this SHA, or that no current-head run exists. Mark **only the next PR in merge order** ready, then emit `WAITING ON YOU: merge PR #N`. Leave later PRs draft. Do not merge while waiting. After the human merges, rebase and re-gate the next PR, then live-verify staging. Production remains human-run.

## Heartbeat

Poll at least every 10 minutes until the track is complete, blocked, or handed off. Check tmux output, git, PR head, CI, and the progress log. Same three-attempt escalation as the [general playbook](general.md). Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED; see [general.md](general.md#heartbeat). Do not go quiet at GATE.
