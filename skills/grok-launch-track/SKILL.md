---
name: grok-launch-track
description: Launch, resume, or inspect one Grok CLI implementation track on a worker host for a Grok Bot CoS. Use when dispatching the next phase of a Grok goal prompt to a free machine in the worker pool.
---

# Launch a Grok CLI track

Use this after the dependency graph says a track is unblocked. Full rules: [playbooks/grok.md](../../playbooks/grok.md).

## Pick a host

1. Read the goal's worker-pool table.
2. On each candidate: tmux list, `$HOME/.grok/bin/grok --version`, login check, reviewer CLI present, repo on the default branch and not behind origin, working tree clean.
3. Skip busy or unhealthy hosts. Assign the least-loaded host that fits.
4. Record host, worktree path, tmux session, and grok session id in the graph.

## Worktree

```sh
ssh user@host 'git -C <repo> fetch origin -q && git -C <repo> worktree add <worktree> -b <branch> origin/<default>'
```

`user@host` is a placeholder. Never implement in the shared default-branch checkout. Do not rely on `grok --worktree` in `-p` mode.

## First phase

```sh
GROK="$HOME/.grok/bin/grok"
tmux new-session -d -s <track> \
  "$GROK --no-auto-update --always-approve --permission-mode bypassPermissions \
     -m grok-4.6 --cwd <worktree> --verbatim -p \
     'Read <absolute-goal-prompt> and execute the next incomplete phase end-to-end. Open a draft PR, then run the reviewer CLI and post gh pr review --comment. Report AWAITING GATE with branch, PR, SHA, COMMENT URL.'"
```

## Later phases (same track)

```sh
tmux new-session -d -s <track> \
  "$GROK --no-auto-update --always-approve -m grok-4.6 --cwd <worktree> \
     --continue -p 'Execute the next incomplete phase. Same gate protocol.'"
```

Prefer `--continue` or `--resume <uuid>` over a new conversation.

## Steer

Interactive TUI (`grok --no-alt-screen` in tmux) only when someone must watch. Send a short "read this file" command rather than pasting a large prompt. Verify with `tmux capture-pane` that work started.

## After launch

- Confirm the pane is alive.
- Do not double-dispatch this host until the track reports GATE, block, or done.
- Heartbeat every 10 minutes from the CoS routine.
