---
name: grok-launch-track
description: Launch, resume, or inspect one Grok CLI implementation track on a worker host for a Grok Bot CoS. Use when dispatching the next worker phase of a Grok goal prompt to a free machine in the worker pool.
---

# Launch a Grok CLI track

Use this after the dependency graph says a track is unblocked. Full rules: [playbooks/grok.md](../../playbooks/grok.md).

Do not dispatch staging or last integration. Those phases are CoS-only. Workers stop at `AWAITING GATE` or `AWAITING SPLIT`.

## Pick a host

1. Read the goal's worker-pool table.
2. On each candidate: tmux list, `$HOME/.grok/bin/grok --version`, login check (`grok` models + `gh`), reviewer CLI present, repo on the default branch and not behind origin, working tree clean.
3. Login is a dispatch input, not a wave blocker. Skip a logged-out or busy host. Re-pick the least-loaded logged-in host that fits and write the new owner into the graph.
4. If no logged-in host fits, escalate dispatch. The CoS does not become the implementer.
5. Record host, worktree path, tmux session, and grok session id in the graph.

## Worktree

```sh
ssh user@host 'git -C <repo> fetch origin -q && git -C <repo> worktree add <worktree> -b <branch> origin/<default>'
```

`user@host` is a placeholder. Never implement in the shared default-branch checkout. Do not rely on `grok --worktree` in `-p` mode. Resolve `<default>` from this repo; do not assume `main`.

Default tmux `remain-on-exit` is off. A finished `grok -p` then destroys the only pane, so `AWAITING GATE` and failures vanish between heartbeats. Set `remain-on-exit on` before the process can exit, tee stdout/stderr to `$HOME/.grok/logs/<track>.log`, and capture from the pane **or** that log.

If tmux is gone later, infer `AWAITING GATE` from the worker status file plus git/PR: draft PR + COMMENT on this head + SHA. A missing pane is not a dead track. Do not invent state from tmux. Do not re-dispatch.

## First phase

```sh
GROK="$HOME/.grok/bin/grok"
LOG="$HOME/.grok/logs/<track>.log"
mkdir -p "$HOME/.grok/logs"
tmux new-session -d -s <track>
tmux set-option -t <track> remain-on-exit on
tmux send-keys -t <track> \
  "$GROK --no-auto-update --always-approve --permission-mode bypassPermissions \
     -m grok-4.6 --cwd <worktree> --verbatim -p \
     'Read <absolute-goal-prompt> and execute the next incomplete worker phase end-to-end. Skip staging and last integration; those are CoS-only. One outcome only. Phase DoD is the product check commands (make check, or the exact commands from that repo CI), not vibes. AWAITING GATE is illegal until those commands exit 0. In the assigned worktree, run that check. Push only if it is green; fail = do not push, do not open the draft. Do not use a looser local subset. Before opening a draft PR, run scripts/pr-size-check. Over cap or a second outcome: report AWAITING SPLIT with the file list grouped by subsystem, the product-line count, and a proposed split; do not open a megadiff. File count alone does not fail GATE. Otherwise open a draft PR, wait until CI is green on this SHA, then run the reviewer CLI and post gh pr review --comment. Write the worker status file (state, PR, SHA, product lines, review round, next action). Run scripts/gate-preflight. Never gh pr ready. Report AWAITING GATE with branch, PR, SHA, COMMENT URL. If the product repo has no make check, or CI does not run on this SHA, say so in that report.' \
     2>&1 | tee \"$LOG\"" Enter
```

## Later phases (same track)

`remain-on-exit` leaves `<track>` alive with a dead pane. `tmux new-session -d -s <track>` then fails, and `send-keys` cannot run in the dead pane. Respawn the held pane (or create the session only if it is gone).

```sh
GROK="$HOME/.grok/bin/grok"
LOG="$HOME/.grok/logs/<track>.log"
mkdir -p "$HOME/.grok/logs"
if tmux has-session -t <track> 2>/dev/null; then
  tmux respawn-pane -k -t <track>
else
  tmux new-session -d -s <track>
  tmux set-option -t <track> remain-on-exit on
fi
tmux send-keys -t <track> \
  "$GROK --no-auto-update --always-approve -m grok-4.6 --cwd <worktree> \
     --continue -p 'Execute the next incomplete worker phase. Skip staging. Same gate protocol. One outcome only. Phase DoD is the product check commands; AWAITING GATE is illegal until they exit 0. Run that check in the worktree; push only if green. Run scripts/pr-size-check before opening a draft. Over cap or a second outcome: AWAITING SPLIT, do not open a megadiff. File count alone does not fail GATE. COMMENT only after this-SHA CI is green. Write the status file. Run scripts/gate-preflight.' \
     2>&1 | tee -a \"$LOG\"" Enter
```

Prefer `--continue` or `--resume <uuid>` over a new conversation.

## Steer

Interactive TUI (`grok --no-alt-screen` in tmux) only when someone must watch. If the controller cannot paste into tmux, write the instruction to a file and send a short Read of that file. Do not paste a large prompt. Verify with `tmux capture-pane -t <track> -p` or `tail` of `$HOME/.grok/logs/<track>.log` that work started.

## After launch

- Confirm the pane is alive or the log has started. After `grok -p` exits, read the held pane or the log — do not treat a dead command as missing output.
- A missing pane later is not a dead track. Infer GATE from git/PR. Do not re-dispatch.
- Do not double-dispatch this host until the track reports GATE, block, or done.
- Heartbeat every 10 minutes from the CoS routine while implementing or in CI. Every CoS-visible message starts with WORKING, WAITING ON YOU, or BLOCKED. `AWAITING GATE` in the graph is `WAITING ON YOU: merge PR #N` (or accept residual / unlock REVISE), not one notice then quiet.
