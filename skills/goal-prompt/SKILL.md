---
name: goal-prompt
description: Author or revise a standing goal prompt for a Chief of Staff plus worker CLIs. Use when creating autonomous multi-phase work that any controller (Claude, Codex, Grok) will re-read each session.
---

# Author a goal prompt

A goal prompt is a self-contained document that a CoS and its workers re-read every session: find the next incomplete phase, execute it, update the progress log. It must survive as the only context a fresh session has.

Write them **long**. Short goals stall. Lock human decisions as `D1`, `D2`, … so workers do not re-litigate them.

For a Grok Bot CoS, also use [`grok-goal-prompt`](../grok-goal-prompt/SKILL.md). Do not point a goal written for one controller at another without rewriting the provider block.

## Authoring order

1. Scope the backlog against the current default branch (ADRs, roadmap, open issues).
2. Lock human decisions (scope cut, spend, production, destructive data).
3. Verify context today: default-branch facts, and each candidate worker (binary, login, repo, tools).
4. Draft with the skeleton below.
5. Ship via feature branch + PR. Do not commit the goal to the default branch from a dirty worker checkout.
6. Record implementer pool, CoS identity, locked descopes, and host prep gaps.

## Skeleton (all sections required)

```markdown
# Goal Prompt — <Project Name>

_<date> · Owner: <human> · CoS: **<name>** ·
Workers: <implementer CLI and model> on the **worker pool** (CoS assigns host at dispatch)_

Standing instruction: read this file, find the next incomplete phase, execute
it end-to-end, do not skip verification. Report AWAITING GATE with branch, PR,
head SHA, and COMMENT review URL.

## Goal
## Decisions already made (<human>, <date>)   — D1..Dn
## Context (verified <date> — re-validate against current default branch)
## Phases          — Phase 0 is docs/ADRs; then code phases
## Worker pool     — fill-in host table: SSH, fit, max parallel tracks
## Autonomy
## Model policy    — writer vs different-family reviewer
## Workflow (per PR)
## Verification (every phase)
## Definition of done (per phase)
## Progress log    — Phase | Status | PRs / ADRs | Notes
```

## Section rules

- **Worker pool:** hosts, fit, max sessions. Placeholders: `dev-1`, `ci-box`, `user@host`. CoS picks a free host per track. Do not hard-code one machine as the only implementer unless the pool truly has one row.
- **Phase 0** is always docs/ADRs. Code after.
- **Model policy:** writer CLI writes. Review is a different family in a fresh session. `gh pr review --comment`, never Approve. Never tell the worker to review itself in the writing session.
- **Workflow:** git worktree per track; conventional commits; lint/test/build before push; mutation-test new tests; draft PR; reviewer CLI; `AWAITING GATE`; CoS serializes merges. No production dispatch.
- **Autonomy:** in-scope work does not stop to ask. Human-only: production, secret creation, destructive data, spend above the locked cap, interactive auth, DNS.
- **Heartbeat:** CoS every 10 minutes while tracks run, timestamped, including "still working."

## Hard rules

- Review and checkpoint notes are mandatory, not "if warranted."
- Descope new paid accounts and human-held credentials; record the descope.
- Production, real money, and data-destructive work stay out of scope unless a decision unlocks them.
- Environments keep distinct names (local / staging / prod). Do not use a pet hostname as an environment name.
- Amend the goal via PR. Progress log is append/update, not rewritten history.
- Public trees contain no personal names, IPs, or real host inventory.
