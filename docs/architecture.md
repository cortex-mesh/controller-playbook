# Architecture

The playbook is a control loop around ordinary git repositories. Nothing here replaces your product architecture.

## System

```mermaid
flowchart TB
  subgraph control [Control plane]
    Goal[Goal prompt]
    Graph[Dependency graph]
    CoS[Chief of Staff]
    Beat[10-minute heartbeat]
    Meta[Meta-repo map]
  end

  subgraph pool [Worker pool]
    Dev1[dev-1 worktree]
    Dev2[dev-2 worktree]
    CiBox[ci-box worktree]
  end

  subgraph review [Review]
    Rev[Different-family reviewer]
    Gh[Draft PR on GitHub]
  end

  subgraph gates [Gate classes]
    Repo[Repository]
    Stage[Staging]
    Prod[Production - human]
  end

  Goal --> CoS
  Graph --> CoS
  Meta --> CoS
  Beat --> CoS
  CoS -->|dispatch one phase| Dev1
  CoS -->|dispatch one phase| Dev2
  CoS -->|heavy jobs| CiBox
  Dev1 --> Gh
  Dev2 --> Gh
  CiBox --> Gh
  Gh --> Rev
  Rev -->|COMMENT never Approve| Gh
  Gh --> Repo
  Repo --> Stage
  Stage --> Prod
```

- The **goal prompt** is the standing instruction every fresh session re-reads.
- The **meta-repo** is a map of products, ADRs, and runbooks. It is not the product.
- Each **worker** owns one git worktree on one host. A laptop-only pool is one host with many worktrees, still one track per worktree.
- **Review** is a different process and a different model family from the writer.
- **Production** is outside the worker's autonomy.

## One-track sequence

```mermaid
sequenceDiagram
  actor Human
  participant CoS as Chief of Staff
  participant Worker as Worker CLI
  participant Rev as Reviewer CLI
  participant GH as GitHub
  participant Stage as Staging

  Human->>CoS: lock decisions and goal prompt
  CoS->>CoS: write dependency graph
  CoS->>Worker: dispatch next incomplete phase
  Worker->>Worker: implement in dedicated worktree
  Worker->>Worker: lint, test, mutation-check
  Worker->>GH: open draft pull request
  Worker->>Rev: different-family review on this head
  Rev->>GH: COMMENT review
  Worker->>CoS: AWAITING GATE plus COMMENT URL
  CoS->>GH: confirm COMMENT and CI on this SHA
  alt blocking findings
    CoS->>Worker: REVISE with file, failure, proof
    Worker->>GH: new head
    Worker->>Rev: review the new head
  else gate
    CoS->>GH: serialize merge
    CoS->>Stage: deploy merged SHA
    CoS->>CoS: live-verify staging
    CoS->>Human: production remains a human gate
  end
```

## Track artifact

Every live track should be nameable from the graph:

| Field | Example |
| --- | --- |
| Track | `harbor-berth-api` |
| Host | `dev-1` |
| Worktree | a path on that host, never the shared default-branch checkout |
| Branch | `feat/berth-api` |
| Base SHA | the default-branch SHA at dispatch |
| Input contract | ADR-0001 accepted; schema from track A |
| Overlaps | none with `harbor-web-shell` |
| Repo gate | lint, test, COMMENT, CI on this SHA |
| Staging gate | merge after track A; live GET `/health` |
| CoS may do without a human | rebase, COMMENT confirm, merge, staging verify |

## Watchdog

The CoS cannot sit in one turn forever. A 10-minute routine (or a tmux watchdog that only repeats an authorized step) keeps the loop moving:

- Heartbeat while any track is running, including "still working."
- Inspect tmux, git, PR head, and CI.
- Take the next already-authorized repo-safe step.
- Do not double-dispatch a progressing track.
- After three documented attempts at the same blocker, escalate and stop that track.

See [ADR 0007](adr/0007-watchdog-heartbeat.md).
