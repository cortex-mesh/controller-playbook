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

Default tmux `remain-on-exit` is off. A finished `grok -p` then destroys the only pane, so `AWAITING GATE` and failures vanish between heartbeats. Set `remain-on-exit on` before the process can exit, tee stdout/stderr to `$HOME/.grok/logs/<track>.log`, and capture from the pane **or** that log.

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
     'Read <absolute-goal-prompt> and execute the next incomplete phase end-to-end. Open a draft PR, then run the reviewer CLI and post gh pr review --comment. Report AWAITING GATE with branch, PR, SHA, COMMENT URL.' \
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
     --continue -p 'Execute the next incomplete phase. Same gate protocol.' \
     2>&1 | tee -a \"$LOG\"" Enter
```

Prefer `--continue` or `--resume <uuid>` over a new conversation.

## Steer

Interactive TUI (`grok --no-alt-screen` in tmux) only when someone must watch. Send a short "read this file" command rather than pasting a large prompt. Verify with `tmux capture-pane -t <track> -p` or `tail` of `$HOME/.grok/logs/<track>.log` that work started.

## After launch

- Confirm the pane is alive or the log has started. After `grok -p` exits, read the held pane or the log — do not treat a dead command as missing output.
- Do not double-dispatch this host until the track reports GATE, block, or done.
- Heartbeat every 10 minutes from the CoS routine.
