---
name: sequential-execute
description: Execute the sub-issues of a main GitHub issue (the map) one at a time in dependency order — one blocking sub-agent per ticket, one commit per ticket.
disable-model-invocation: true
---

# Sequential Execute

Execute the tickets of a main issue (the map) sequentially: build the dependency DAG from the native sub-issue and blocking links, confirm the order with the user, then run one blocking sub-agent per ticket in a fresh context, one commit per ticket, stopping at the first failure.

Argument: the map issue number (a URL is accepted).

The human intervenes at exactly two points: the DAG gate before the first dispatch, and the final report. Every other path runs autonomously — a resume drops its partial work without asking, and a failure stops the run with a report, not a question.


## 1. Resolve the map

- Derive the repo owner/name from `gh repo view --json owner,name`.
- Progress file: `/tmp/sequential-execute/issue-<N>.progress.md` — one line per ticket: `done|failed|pending <number> [<commit-hash>]`.

**Done when:** the issue number, owner/name, and progress-file path are fixed.

## 2. Build the DAG

Use `gh` CLI to collect every ticket and its blocking edges:

```bash
# List sub-issues of the map issue
gh api repos/<owner>/<repo>/issues/<N>/sub-issues \
  --jq '.[] | {number, title, state}'

# For each sub-issue <NN>, list its blockers
gh api repos/<owner>/<repo>/issues/<NN>/blocking \
  --jq '.[].number'
```

- Keep only `blockedBy` edges whose target is also a ticket of this map; list any open blocker outside the map for the gate.
- Topologically order the tickets (Kahn's algorithm; break ties by issue number ascending).
- Closed tickets count as finished: mark them `done` in the progress file at the gate.
- If the progress file already exists, this is a **resume**: read it, keep its `done`/`failed` lines, and drop the uncommitted partial work from the previous run automatically (`git restore . && git clean -fd`).

**Done when:** the order is ready, the graph is acyclic (a cycle is surfaced at the gate with a proposed resolution), and a resume's partial work is dropped.

## 3. DAG gate

Show the resolved order — number, title, blockers — with closed/resumed tickets marked, any outside open blockers flagged, and, if the graph cycled, the resolution being proposed. This is the only pre-dispatch intervention.

**Done when:** the user explicitly approves the order.

## 4. Sequential dispatch

For each pending ticket in topological order, one at a time:

1. **Snapshot** the tree: `git status --porcelain` — record the exact set of untracked and modified files.
2. **Dispatch one sub-agent** (subagent tool, fresh context) with the task template below. A second sub-agent starts only after the first returns.
3. **Verify**: the sub-agent reported a commit hash and `git show <hash>` exists on the current branch.
4. **Success** → record `done <number> <hash>` in the progress file; continue to the next ticket.
5. **Failure** (reported failure or missing commit) → clear the tree against the snapshot: `git restore` the tracked files changed since the snapshot, delete the untracked files created since the snapshot; record `failed <number>`; stop the run and report the ticket, the failure, and the clear.

**Done when:** every ticket is `done` (or the run has stopped on a failure).

### Sub-agent task template

> Implement GitHub issue #<NN> in <owner>/<repo>.
> 1. Read the requirements: `gh issue view <NN> --json title,body,comments` — work from that body and its acceptance criteria only.
> 2. Implement: use /tdd at pre-agreed seams; run typechecking and single test files regularly; run the full test suite once at the end.
> 3. Commit the work as a single commit, message `FIX|IMPROVE|NEW: <ticket title> (#<NN>)` — pick the type by the kind of change.
> 4. Close the issue: `gh issue close <NN>`.
> 5. Report: the commit hash, a summary of the change, the test results, and anything left incomplete.

## 5. Wrap-up

Run the full typecheck and test suite once. Report: every ticket with its commit, any failures, and anything left open.

**Done when:** the full suite is green and the summary is printed.
