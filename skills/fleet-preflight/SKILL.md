---
name: fleet-preflight
description: Probe the worker pool live before every dispatch and on the 10-minute heartbeat. Use when picking a host, when a heartbeat runs while tracks are implementing or in CI, or when deciding if a box is down vs AWAITING LOGIN. Never cache host IPs.
---

# Fleet preflight

Run [`scripts/fleet-preflight`](../../scripts/fleet-preflight) against a **local** pool file before every dispatch, and again on every 10-minute heartbeat while tracks are implementing or in CI.

Do not remember an address from the last probe. VPN IPs move. Resolve the pool live every time.

```sh
scripts/fleet-preflight /path/to/local-pool.tsv
# or
FLEET_POOL=/path/to/local-pool.tsv scripts/fleet-preflight
```

The committed [example](../../examples/worker-pool.example.tsv) is placeholders (`dev-1`, `dev-2`, `ci-box`) only. Copy it locally. Never commit real hostnames, IPs, emails, or VPN names to a public tree.

## When

- **Dispatch.** Before picking a host. The recommended line is the least-loaded free logged-in host.
- **Heartbeat.** While any track is implementing or in CI, re-run the script. Do not reuse last heartbeat's IPs or last heartbeat's "down" list.
- **Not a wave blocker.** A logged-out box does not mean the pool is empty.

## How to read the table

| status | Meaning | What the CoS does |
| --- | --- | --- |
| `free` | Reachable, implementer logged in, `gh` logged in, live implementer PIDs below `max_parallel` | Dispatch here (prefer the `recommended:` host) |
| `busy` | Logged in, and a **live** implementer process is using the slot | Do not double-dispatch |
| `login-needed` | Reachable, but implementer and/or `gh` is logged out (or the CLI is missing) | **AWAITING LOGIN.** Skip. Re-pick a logged-in host and write the new owner into the graph. This is not down |
| `down` | SSH failed this run (timeout, refused) or the VPN CLI says the peer is offline | Check the **rest** of the pool. One timeout is not "no hosts" |

`live_sessions` counts live implementer PIDs (`grok -p` / `--continue` / `--prompt-file`, `codex exec`, `claude -p`). A leftover tmux pane with `remain-on-exit` and no live process is **not** busy.

## Hard rules

- Never cache host IPs in memory, in the graph, or in a note "for next time." The pool file holds names. The script resolves this run only.
- Never skip the rest of the pool after a timeout. Finish the table.
- Never classify `login-needed` as down. AWAITING LOGIN is a skip-and-re-pick, not an empty pool.
- If the script exits 2, read `login-needed (AWAITING LOGIN):` vs `down:` vs `busy:`. Escalate dispatch only when no logged-in free host fits. The CoS does not become the implementer.
- Leftover panes are not occupancy. Live PIDs are.
- Do not paste real IPs, emails, or private host tables into PRs, heartbeats, or the public tree.
