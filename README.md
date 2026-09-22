# sequential-execute

A pi skill that turns a GitHub tracking issue (the "map") into a sequential execution pipeline — one sub-agent per ticket, one commit per ticket, stopping at the first failure.

## Why

A well-structured tracking issue with native sub-issues and blocking links already encodes the dependency graph of a feature. This skill takes that graph and drives it to completion:

- **No manual sequencing** — it reads the DAG from GitHub (sub-issue links + `blockedBy` edges), topologically sorts it, and dispatches in order.
- **One commit per ticket** — each sub-agent implements exactly one issue, commits it atomically, and closes it. History stays bisectable.
- **Fail-fast with clean rollback** — if a ticket fails, the tree is restored to its pre-dispatch snapshot, the failure is reported, and the run stops. No partial state bleeds into the next ticket.
- **Two human checkpoints, not twenty** — you approve the plan once (the DAG gate) and review the result once (the final report). Everything between runs autonomously.
- **Resumable** — a progress file records which tickets are done, so a second invocation picks up where the last one left off and discards any uncommitted partial work.

## How it works

```
Map issue #42
├── #43  Design schema       (closed, no deps)
├── #44  Implement schema    (blocked by #43)
├── #45  API endpoints       (blocked by #44)
├── #46  CLI commands        (blocked by #44)
└── #47  Integration tests   (blocked by #45, #46)
```

1. **Resolve** — reads `gh repo view` to get owner/repo, fixes the progress file path.
2. **Build the DAG** — `gh api` fetches sub-issues and their `blockedBy` edges; Kahn's algorithm produces the execution order.
3. **DAG gate** — shows you the resolved order (with closed/resumed tickets marked). You approve or adjust.
4. **Sequential dispatch** — for each pending ticket: snapshot the tree → dispatch a fresh-context sub-agent → verify the commit → record progress. The sub-agent reads the issue, implements via TDD, commits, and closes the issue.
5. **Wrap-up** — full typecheck + test suite, then a summary of every ticket with its commit.

## Usage

```
/sequential-execute 42
```

or with a full URL:

```
/sequential-execute https://github.com/owner/repo/issues/42
```

## Dependencies

- **pi** — with the `subagent` tool enabled.
- **gh CLI** — authenticated and scoped to the target repo.
- **GitHub native sub-issues** — the map issue must have sub-issues and blocking links set up.
- **`docs/agents/issue-tracker.md`** — project-level conventions for native sub-issues, blocking links, and `gh` gotchas (referenced by the skill at runtime).
- **Clean working tree** — resumes automatically drop partial work, but dispatch starts from a clean slate.

## Design notes

- **Fresh context per ticket.** Each sub-agent sees only the issue body and the codebase — no accumulated context from prior tickets. This keeps implementations focused and avoids cross-ticket contamination.
- **The snapshot pattern.** Before each dispatch, `git status --porcelain` records the exact file set. On failure, only files added/changed *since* that snapshot are cleaned — pre-existing uncommitted work is untouched.
- **Progress file is the source of truth.** `/tmp/sequential-execute/issue-<N>.progress.md` survives across invocations. Close tickets in the map are auto-marked `done` at the gate; the file only grows, never shrinks (except on resume, which discards the current run's partial line).
- **The DAG gate is a hard gate.** No dispatch happens until you explicitly approve the order. Cycles are surfaced here with a proposed resolution (typically "break at X, re-link later").
