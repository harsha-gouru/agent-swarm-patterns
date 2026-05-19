# Git-as-Bus Swarm

A new pattern observed in Cursor's Grind. Distinct from filesystem-based coordination (Antigravity's `.agents/` directory) and from in-memory message passing: **git is the coordination primitive**. Workers push commits; planners observe results by pulling rebased clones; conflicts are resolved by the workers themselves via rebase-and-synthesize.

## Shape

```text
                          ┌──────────────────┐
                          │     Planner      │  reads only
                          │  (own clone)     │
                          └────────┬─────────┘
                                   │ submit_task
                  ┌────────────────┼────────────────┐
                  ▼                ▼                ▼
            ┌──────────┐    ┌──────────┐    ┌──────────┐
            │ Worker 1 │    │ Worker 2 │    │ Worker 3 │
            │ own clone│    │ own clone│    │ own clone│
            │ + branch │    │ + branch │    │ + branch │
            └────┬─────┘    └────┬─────┘    └────┬─────┘
                 │               │               │
                 │  git push --rebase (retry)    │
                 │               │               │
                 └───────────────▼───────────────┘
                       ┌──────────────────┐
                       │  shared branch   │  ← single source of truth
                       │   on origin      │
                       └──────────────────┘
                                ▲
                                │ git pull --rebase
                                │
                          (Planner reads
                           rebased state)
```

## Use when

- Workers can be **parallelized and need to commit code** (the dominant agentic-IDE workload).
- You want **safety by isolation** — each worker can't break another's working tree.
- You want **mergeability by design** — git's three-way merge is the conflict resolver of last resort, but workers do their own synthesis first.
- Persistence requirements are exactly "what got committed survives; everything else is throwaway."
- You're OK with the overhead of N git clones (Grind accepts this).

## Prompt shape

```text
You are a Worker in <swarm-name>.
You implement. You push. Your commit is your contribution.

Your isolated git clone is at <path>. Your branch is <branch>.

Workflow:
  1. Read AGENTS.md (mandatory repo-wide rules).
  2. Read <instructionsFile> (workstream-specific rules).
  3. Implement your task.
  4. Verify the build passes. NEVER commit broken code.
  5. Make ONE commit containing all changes.
  6. Push to <branch> with rebase-retry:
       git pull --rebase
       git push
     On push failure (other workers pushed first):
       Wait with exponential backoff (2-4s, 4-8s, 8-16s, 16-32s)
       Rebase again, resolve conflicts, retry.
       Never give up.
  7. On conflict: synthesize both contributions. Read both sides,
     preserve valuable work from both, never accept blindly,
     never leave conflict markers, never force push.
  8. Before finishing, write ./handoff.md (NEVER committed).

What survives:
  Your commit message.
  The tests you wrote.
  The comments you left.
  Nothing else. Encode insights in the codebase.
```

## Guardrails

- **One commit per worker.** Multiple commits make merging downstream changes ambiguous. Squash before push if needed.
- **Always rebase, never merge.** Merge commits pollute history and make planner reasoning harder. Grind forbids merges by policy.
- **Never force push.** A force-push wipes out another worker's commit that landed first.
- **Build verification is mandatory before commit.** A broken commit blocks every following worker (they'll rebase onto it and fail too).
- **Handoff files are local-only.** `handoff.md`, `scratchpad.md`, planner notes are explicitly never committed and never referenced from any other artifact. Treat them as RAM, not disk.
- **AGENTS.md is the contract.** Repo-wide rules live there. Workers MUST read it before implementing. Equivalent to Claude Code's CLAUDE.md.
- **Synthesize, don't pick.** Conflict resolution is a creative act, not a `git checkout --ours` shortcut. Workers explicitly read both sides and write the merged version that respects both intents.
- **Workers don't `git pull` other workers' branches.** They only ever interact with the shared target branch via rebase. No cross-worker chatter.

## Failure modes

- **The poison commit.** One worker commits broken code; every subsequent rebase fails. Mitigation: mandatory build verification + CI gate before push (Grind relies on the worker prompt's "DO NOT commit code that doesn't compile" rule, which is weaker than a hard CI gate).
- **Conflict resolution at scale.** When many workers touch the same area, late workers do increasing amounts of rebase work. Mitigation: planner partitions work to minimize overlap.
- **Lost handoffs.** If a worker dies between writing `handoff.md` and the planner reading it, the handoff is gone. Mitigation: planner polls for worker completion; runtime should preserve handoff out-of-band if the worker is killed mid-step.
- **Eternal retry.** "Never give up — keep retrying until you succeed" can loop forever if the shared branch is hot. Mitigation: continuation policy `maxLoops: 100` eventually trips an escape hatch.
- **Branch churn.** N parallel workers across M tasks means M clones, M branches, M pushes. Disk and bandwidth cost real money on cloud agents. Mitigation: Grind has a `shared-no-git` mode for lighter swarms where git isolation isn't worth the cost.
- **No durable cross-worker communication.** Workers can't message each other — only the planner can mediate. Forces strict tree topology; cycles are impossible by design (sometimes you want one).

## Differences from existing patterns

- **vs `best-of-n-worktree`** — best-of-n is competition (N attempts, pick one winner). Git-as-bus is collaboration (N workers, all commits land via rebase).
- **vs `flat-map-reduce`** — flat-map-reduce reduces in the parent's context. Git-as-bus reduces in git's history graph; the planner just reads the result.
- **vs `pipeline-handoff-swarm`** — pipeline handoff is sequential (research → impl → review). Git-as-bus is concurrent.
- **vs Antigravity's `.agents/` filesystem** — filesystem-as-bus carries rich metadata (BRIEFING, progress.md, plan.md, handoff.md) but has no built-in conflict resolution. Git-as-bus carries only what you commit but inherits 20 years of merge tooling.

## Runtime primitives needed

- Per-agent git clone (or worktree) creation + cleanup.
- Shared origin with a known branch as the integration target.
- Reliable `git pull --rebase + push` retry loop with exponential backoff.
- A way to run the build inside each clone before commit (subprocess + exit-code check).
- Conflict-resolution prompt fragment in the worker system prompt (LLM does the synthesis, not the runtime).
- Optional: pre-push CI gate (Grind doesn't have one; relies on the prompt).
- Optional: locality-aware task partitioning by the planner (Grind plays this implicitly via `submit_task`'s scope-specification).

## Hybrid variants

- **`shared` mode** — single working dir, full git access. Less isolation but no clone overhead. Use when workers won't clobber each other (well-partitioned tasks).
- **`shared-no-git`** — single working dir, NO git ops allowed. Workers just write files; planner commits them at the end. Lightest mode; gives up the "synthesize conflicts" property.
- **Best-of-N worker swarm** — combine git-as-bus with `best-of-n-runner` for areas where you want competitive attempts (each in own worktree) AND collaborative implementation (all land via rebase). Grind doesn't ship this combination but the primitives exist.

## Source

Recovered from `cursor-agent-exec/dist/main.js` in Cursor 3.4.20. Key symbol: `v1({branch, instructionsFile})` produces the Worker system prompt; the verbatim rebase-retry protocol with the `2-4s, 4-8s, 8-16s, 16-32s` backoff table lives inside it.

The pattern name **"git-as-bus"** is mine — Cursor doesn't name it. Cursor names the swarm "Grind" but uses git-as-bus as a primitive without giving the primitive itself a name.
