# ADR 0007: A 10-minute heartbeat watches running tracks

- Status: Accepted
- Date: 2026-08-13

## Context

Controller chats end. Headless workers continue. Without an external wake-up, nobody notices a stuck tmux pane, a red default-branch CI, or a PR that has been `AWAITING GATE` for an hour. Quiet-at-GATE (one notice, then silent) looks like the CoS died while it is waiting on the human to merge.

## Decision

Every CoS-visible message starts with one label:

- **WORKING** — CoS/worker still going. No action from the human. While any track is **implementing or in CI**, emit a **dual-timestamped heartbeat at least every 10 minutes**, including "still working" / "will check in again in 10 minutes."
- **WAITING ON YOU** — first line, then the exact action (merge PR #N, authorize device login, approve a graph split). `AWAITING GATE` in the graph is still the repo gate; the human-visible line is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE). Repository merge at GATE is a human action. Stop the 10-minute "still working" heartbeat, but keep a wake-up: one reminder after about 30-60 minutes or the next morning (Grok Bot: install a one-shot routine; the Bot cannot sit in a turn). The reminder re-probes the PR first: if already merged, continue post-merge work (`WORKING`); if still open, the message starts WAITING ON YOU, not a 10-minute drip.
- **BLOCKED** — not the human (implementer logged out, CI, SSH). Name what is blocked and whether the CoS is unblocking it.

Do not phrase a human decision as a status ("merge as-is, or I send the worker back"). That is WAITING ON YOU. Quiet-at-GATE is not allowed.

A watchdog (a scheduled CoS routine, or a tmux loop that only repeats an authorized step) may:

- inspect tmux, git, PR head, and CI
- read the worker status file, `scripts/track-status`, and the append-only GATE log. Do not invent state, PR, SHA, or next action from tmux output or chat.
- infer `AWAITING GATE` from git/PR when the pane is gone (draft PR + COMMENT on this head + SHA). A missing pane is not a dead track. Do not re-dispatch.
- take the next already-authorized repo-safe step
- refuse to double-dispatch a progressing track

It may not redefine scope, invent gates, merge production, become the implementer, or retry an unresolved escalation forever.

Escalation for the same blocker: (1) capture evidence, (2) one REVISE, (3) stop the track and escalate. There is no attempt 4. Independent tracks continue. If a same-class P1 was already fixed once in this SHA lineage, accept the residual or escalate.

Every WORKING implementing/CI heartbeat opens with the label, then local time and `YYYY-MM-DD HH:MM:SS TZ`.

## Consequences

- Grok Bot CoS uses a 10-minute routine every calendar day, including weekends, while any track is implementing or in CI. Claude-style controllers poll on the same cadence.
- `WORKING` plus "still working on `dev-1`" is a valid heartbeat. Quiet-at-GATE is not. `AWAITING GATE` in the graph is `WAITING ON YOU: merge PR #N` to the human. A Bot with no implementing/CI tracks still needs the one-shot reminder while WAITING ON YOU.
- Operators see whether a ping is a status, a request for them, or a stall the CoS is unblocking.
