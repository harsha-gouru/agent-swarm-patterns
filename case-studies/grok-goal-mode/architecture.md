# Grok Goal Mode: Architecture

`/goal "<objective>"` hands an objective to a fixed three-role swarm that runs unsupervised until the objective is **observably** met. This file reconstructs the topology, the state machine, the id-chaining rule, and the verdict system — all from the 0.1.214 binary.

## The topology

```text
   /goal "<objective>"
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  GOAL ORCHESTRATOR                                            │
│  "You are the goal orchestrator."                             │
│  - owns the deliverable list; never does the work itself      │
│  - calls update_goal() after every step (powers a live UI)    │
│  - spawns worker + verifier; loops until GOAL ACHIEVED        │
└───────┬───────────────────────────────────────────┬──────────┘
        │ spawn_subagent (fresh, NEVER fork_context) │
        ▼                                            ▼
┌───────────────────────┐                ┌───────────────────────┐
│  WORKER subagent       │                │  VERIFIER subagent     │
│  - ONE worker for the  │  ── output ──► │  - SEPARATE agent      │
│    whole goal          │                │  - MUST physically run │
│  - accumulates context │                │    the goal end-to-end │
│    across deliverables │ ◄── FAIL ────  │  - produces concrete   │
│  - resumed via         │   gap details  │    runtime evidence    │
│    resume_from         │                │  - verdict: ACHIEVED / │
│                        │                │    GAP / BLOCKED       │
└───────────────────────┘                └───────────────────────┘
        ▲                                            │
        └──────── loop until verifier says ──────────┘
                  GOAL ACHIEVED
```

Three rules pin this shape, recovered verbatim from the orchestrator prompt:

- **`NEVER self-verify by reading worker output — always use a verifier.`** The orchestrator cannot grade the worker. A separate verifier subagent must.
- **`NEVER cancel a running worker.`**
- **`spawn subagents (never use fork_context)`** — workers and verifiers start fresh, not from the orchestrator's conversation. Context flows through `resume_from`, not history-copying.

## Phase 1 — planning

The orchestrator's **first action** is to register a typed deliverable list via the `update_goal` tool:

```text
You are the goal orchestrator. Your tools:
- <update_goal>: call DIRECTLY (built-in, NOT via a subagent)
- <spawn_subagent>: spawn subagents (never use fork_context)

FIRST ACTION — plan and register the deliverable list:
  update_goal(phase: "planning", deliverables: [
    {id: 1, title: "...", status: "pending"}, ...
  ])
```

`update_goal` is a real built-in tool — the binary leaks its Rust path: `crates/codegen/xai-grok-tools/src/implementations/grok_build/update_goal/mod.rs`. Its signature, from recovered examples:

```text
update_goal(phase: "planning",  deliverables: [{id, title, status: "pending"}, …])
update_goal(                    deliverables: [{id: N, …, status: "in_progress"}])
update_goal(phase: "executing", deliverables: [{id: N, …, status: "verified"}])
update_goal(completed: true,    message: "All deliverables verified")
```

## The deliverable state machine

```text
   pending ──► in_progress ──► (verifier runs) ──┬──► verified
      ▲                                          │
      │                                          └──► needs_rework
      └──────────── resume worker ◄───────────────────────┘

   phases:  planning ──► executing
```

> `unknown goal phase from update_goal, defaulting to Idle` — recovered runtime guard; `needs_rework` is a recovered status. State is carried as tool-call arguments, never as prose the model might drift from.

## Phase 2 — the per-deliverable loop

Recovered orchestrator process block (`MANDATORY for EVERY deliverable (no exceptions)`):

```text
PROCESS for each deliverable:
1. mark it (status: "in_progress")
2. spawn / resume the WORKER to implement it
3. spawn the VERIFIER to check it
4. PASS  →  mark it (status: "verified")
5. FAIL  →  resume the worker with the FAIL details, fix,
            then resume the verifier to re-review. Loop until PASS.

Do NOT mark "verified" without a verifier PASS.
AFTER ALL VERIFIED → run a final whole-goal verification.
```

The worker is **persistent**: *"Keep the LATEST worker `subagent_id` alive across the whole goal — the worker accumulates context across deliverables."* One worker, many deliverables, resumed each time.

## The `subagent_id` chain rule — the subtle one

This is the most interesting recovered detail and worth stealing directly. Every `spawn_subagent` (including a resume) returns a **new** `subagent_id`. The orchestrator must keep overwriting its saved id:

```text
Track the LATEST worker subagent_id across the whole goal. Every spawn_subagent
call returns a NEW subagent_id — overwrite your saved id with the latest one each
time. Resuming with a stale (older) id silently restarts the worker with no
memory of prior fix rounds.
```

```text
   spawn worker         → id_A   (save: id_A)
   resume_from id_A     → id_B   (save: id_B)   ← MUST overwrite
   resume_from id_B     → id_C   (save: id_C)
   resume_from id_A  ✗            ← stale id → silent fresh restart, context lost
```

The failure mode is **silent**: a stale id doesn't error, it just gives you a worker that forgot everything. The same rule applies to the **final verifier** — it must be resumed (`resume_from` the latest verifier id, *not* a fresh spawn) so it *"remembers its prior gap-findings."*

This is a general lesson for any resume-based orchestrator: **resume handles must be treated as a moving chain, not a fixed pointer**, and a stale handle should fail loud, not restart silent.

## The verifier — reality, not theory

The verifier prompt is the strictest part of the design. Recovered verbatim:

```text
It MUST verify the goal is met IN REALITY, not in theory:
- "build a working CLI tool"      → the verifier MUST actually run the CLI
- "add a feature to an app"       → MUST start the app, exercise the feature
                                    end-to-end, confirm user-visible behaviour
- "make the test suite green"     → MUST run the test suite and observe the pass
- "improve performance by X%"     → MUST run a benchmark demonstrating X%

"Theoretically achieved" (the code looks right, tests pass in isolation, the
implementation seems plausible) is NOT enough. The verifier must produce concrete
evidence that the goal-as-the-user-asked-it is observably true.
```

## The three-verdict system

The verifier returns exactly one of three outcomes — recovered verbatim:

| Verdict | Meaning | Orchestrator action |
|---|---|---|
| **(a) GOAL ACHIEVED** | Observably met in reality, *at most cosmetic nits remaining* | `update_goal(completed: true)` |
| **(b) GAP FOUND** | Not met, but a worker can fix it (missing feature, wrong behaviour, failed test). *"This is the COMMON case."* | `resume_from` the LATEST worker id with gap details → fix → re-verify |
| **(c) TRULY BLOCKED** | Not achievable in this environment — fundamental mismatch (Windows binary on macOS; no deploy credentials; *internally contradictory goal*) | call with `blocked_reason` + `message`; pause the goal |

The prompt actively fights verdict inflation in both directions:

```text
VERDICT RULE: PASS (GOAL ACHIEVED) means the goal is observably met with at most
cosmetic nits. Any issue more serious than a nit means GAP FOUND, not GOAL
ACHIEVED. Do not lower the bar because "most things work" or "it's close enough."

The bar for (c) is HIGH. Almost everything you encounter is (b), not (c).
"Build broke once", "test failed", "I'm not sure how to fix this" are NOT (c) —
they are (b) and you must hand them back to a worker.

VERIFICATION LIMITATIONS ARE GAPS, NOT BLOCKS: if you cannot fully verify the
goal because of your own tooling limitations, that is a GAP, not a block.
```

So the orchestrator can never "give up" by mislabeling a fixable gap as blocked, and can never "declare victory" on a partial result.

## Pause states — named, not generic

The runtime classifies *why* a goal paused (recovered from the pager view `crates/codegen/xai-grok-pager/src/views/agent_status.rs`):

```text
Paused (doom loop)              ← repeated failure / no progress detected
Paused (back-off)               ← rate / retry back-off
Paused (verification blocked)   ← verifier returned TRULY BLOCKED
```

The verification-blocked pause is surfaced to the user with a human-readable reason:

```text
Goal paused — verification blocked.
Reason: <concrete one-line reason>
Type /goal resume to continue after you've addressed it.
```

`/goal pause` and `/goal resume` are the manual controls; the changelog confirms doom-loop auto-pause and that the old `--budget` flag was removed in favour of stricter verification.

## The end-to-end flow

```text
1. /goal "<objective>"
2. orchestrator: update_goal(phase:"planning", deliverables:[…])      ← register
3. for each deliverable, in order:
     a. update_goal(status:"in_progress")
     b. spawn/resume WORKER  → save the returned subagent_id  (chain!)
     c. spawn VERIFIER       → it physically runs the deliverable
     d. PASS → update_goal(status:"verified")
        FAIL → resume_from latest worker id with gap → goto (b)
4. AFTER ALL VERIFIED: spawn a FINAL verifier over the whole goal
     GOAL ACHIEVED → update_goal(completed:true)   ← done
     GAP FOUND     → resume_from latest worker id  → loop
     TRULY BLOCKED → pause (verification blocked) + reason to user
5. doom-loop / back-off → auto-pause; /goal resume to continue
```

## Takeaways for a hand-rolled orchestrator

1. **Make the verifier a separate agent and forbid self-verification.** The orchestrator that did (or watched) the work is the worst judge of it. Grok hard-codes the split.
2. **Verify in reality, not in theory.** "Tests pass in isolation" is not "the goal is met." Make the verifier *run the thing* and produce runtime evidence.
3. **Three verdicts, with explicit anti-inflation rules.** ACHIEVED / GAP / BLOCKED — and spell out that "close enough" is a GAP and "I'm stuck" is a GAP, not a BLOCK. A two-state pass/fail lets the agent weasel.
4. **Treat resume handles as a moving chain.** Every resume returns a new id; the orchestrator must overwrite. A stale handle that *silently* restarts a fresh agent is the nastiest bug in a resume-based swarm — make it fail loud.
5. **One persistent worker beats N fresh ones** when deliverables share context. Resume it; don't re-brief from scratch each time.
6. **Name your pause states.** `doom loop` / `back-off` / `verification blocked` tell the user (and you) what to do next; a generic "paused" does not.
