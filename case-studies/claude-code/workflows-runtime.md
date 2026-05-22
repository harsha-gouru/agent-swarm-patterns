# Claude Code: the Workflow Runtime

This is the new category. Every other case study in this repo treats a swarm as a *pattern the parent improvises at runtime*. Claude Code 2.1.x makes the swarm a **first-class, versionable, resumable artifact**: a JavaScript file that *is* the orchestration.

## What a workflow is

A workflow is a self-contained `.js` script that lives in `.claude/workflows/` (or is shipped by a plugin, or is built in). It is invoked through the `Workflow()` tool:

```text
Workflow({ scriptPath: '.claude/workflows/implement.js',
           resumeFromRunId: 'run_abc123'   // optional — resume a paused run
        })
```

The script does not *describe* a swarm to a model. It **runs** — in a sandboxed JavaScriptCore VM context (`vmScript.runInContext`) — and the swarm happens as a side effect of the functions it calls.

## The injected API

The sandbox injects these globals into the script's scope. Nothing else (no `require`, no `fs`, no network):

```text
agent(prompt, opts?)   → Promise<string | structuredObject>   spawn one subagent
parallel([fn, fn, …])  → Promise<any[]>     run fns concurrently, collect results
pipeline(items, …)     → Promise<any[]>     map items through staged fns
phase(name)            → void               label the next agents in the progress UI
log(msg)               → void               append to the run log
budget                 → { total, spent(), remaining() }   token budget for the turn
args                   → the params passed in by the caller
console                → mapped to log()
```

`agent()` signature, recovered verbatim:

```text
agent(prompt: string, opts?: {
  label?: string,        // groups calls under a title in the progress display
  phase?: string,
  schema?: object,       // a JSON Schema — see below
  model?: string,
  isolation?: ...,        // 'remote' available only in cloud builds
  agentType?: string,     // pick a .claude/agents/ definition
}): Promise<any>
```

- **Without `schema`** → resolves to the subagent's final text as a `string`.
- **With `schema`** (a JSON Schema) → the subagent is *forced* to call `StructuredOutput`; the result is parsed and validated. If the subagent finishes without calling it, the runtime nudges it up to 2× then errors.

`parallel()` and `pipeline()` guardrails (verbatim error strings):

```text
parallel() expects an array of functions, not promises.
            Wrap each call:  () => agent(...)
pipeline() stages must be functions: pipeline(items, item => ..., result => ...)
```

The "not promises" rule matters: passing `agent(...)` (a promise) starts the agent *immediately* and defeats scheduling. You pass `() => agent(...)` (a thunk) so the runtime controls *when* each one starts.

## The determinism contract — why resume works

Inside a workflow script, **two things are removed**:

```text
Date.now() / new Date()  → unavailable.   "breaks resume.
                            Stamp results after the workflow returns,
                            or pass timestamps via args."
Math.random()            → unavailable.   "breaks resume. For N independent
                            samples, include the index in the agent label
                            or prompt."
```

This is deliberate. Because the script is **deterministic**, the runtime can replay it and know that an `agent()` call with the same `(prompt, opts)` *must* produce the same logical step. So:

```text
RESUME =  re-run the script from the top
          │
          ├─ agent() call with UNCHANGED (prompt, opts)
          │     → return the cached result instantly, do NOT re-spawn
          │
          └─ agent() call that is NEW or EDITED
                → actually re-run it
```

> Recovered: *"`agent()` calls with unchanged (prompt, opts) return their cached results instantly; only edited or new calls re-run. Same-session only. Stop the prior run first before resuming."*

This is the part worth stealing: **if your orchestration script is pure and deterministic, "resume" is free** — it's just memoised replay. Non-determinism (clocks, RNG, ambient state) is what forces you to checkpoint manually. Claude Code's answer is to *ban* the non-determinism rather than checkpoint around it.

The pause/resume handshake the runtime prints:

```text
Resume the paused workflow by calling:
  Workflow({ scriptPath: '…', resumeFromRunId: '…' })
  — completed agents return cached results.
```

## Concurrency & runaway caps

```text
agent() concurrency  =  min(16, cpuCores − 2)   per workflow
                        ~10 run at any moment; excess calls QUEUE
                        (you may pass 100 items to parallel()/pipeline()
                         — all 100 complete, only ~10 run at once)

lifetime cap         =  1000 agents total per workflow
                        → hard backstop against a loop that calls agent()
                          forever because budget.remaining() never drops

budget cap           =  budget.total / spent() / remaining()
                        scripts branch on it:  if (budget.remaining() > 50_000) {…}
```

Telemetry confirms both as real failure modes: `tengu_workflow_agent_cap_exceeded`, `tengu_workflow_budget_cap_exceeded`. The `agent() call cap reached` error explicitly blames *"a loop using budget.remaining() that never terminates."*

## The subagent's view

A subagent spawned *by a workflow* gets this system prompt (verbatim):

> *"You are a subagent spawned by a workflow orchestration script. Your final text response is returned **verbatim** as a string to the calling script — it is your return value, not a message to a human. Output the literal result. Do NOT output confirmations like 'Done.' If asked for JSON, return ONLY the raw JSON — no code fences, no prose."*

So the agents inside a workflow are treated as **pure functions**: prompt in, value out. That is what makes `const draft = await agent(...)` compose like ordinary code.

## A real built-in workflow (verbatim)

Recovered from the binary — the plan phase of an implement-a-task workflow:

```js
const PLAN_SCHEMA = {
  type: 'object',
  properties: {
    summary: { type: 'string' },
    files:   { type: 'array',  items: { type: 'string' } },
    steps:   { type: 'array',  items: { type: 'string' } },
    reuse:   { type: 'array',  items: { type: 'string' } },
    risks:   { type: 'array',  items: { type: 'string' } },
    // ...
    lintPassed: { type: 'boolean' }, typecheckPassed: { type: 'boolean' },
  },
}

// ░░░ Phase 1: Plan ░░░
phase('Plan')

const draft = await agent(
  "Scope this task against the codebase and draft an implementation plan.\n\n" +
  "## Task\n" + TASK + "\n\n" +
  "## Instructions\n" +
  "1. Explore — find relevant files, existing patterns, utilities to reuse. " +
  "   Actively search for existing functions; avoid proposing new code when " +
  "   suitable implementations already exist.\n" +
  "2. Read CLAUDE.md at project root and in parent dirs of relevant files.\n" +
  "3. Draft a concrete plan: files to touch, what edits, in what order.\n" +
  "4. Call out existing code to reuse with file:line.\n" +
  "5. List risks and describe verification (test command, manual check).\n\n" +
  "Be concrete — file paths and function names, not vague intentions.",
  { label: 'plan:draft', schema: PLAN_SCHEMA }
)

if (!draft) return { error: 'Plan draft skipped.' }
log('Draft: ' + draft.files.length + ' files, ' + draft.steps.length + ' steps')
// ... draft.summary / draft.files / draft.steps feed the next phase
```

Note what this gives you that a prompt cannot:

- `draft` is a **typed object** (schema-validated) — the next phase reads `draft.files`, not a paragraph.
- `phase('Plan')` is a real statement — phases are code, not headings a model might ignore.
- `if (!draft) return` is real control flow — the workflow can abort deterministically.

## Built-in workflow examples in the binary

| Workflow | What it does |
|---|---|
| debug-root-cause | Collects evidence, runs **parallel hypothesis agents that try to *refute* each hypothesis**, writes up the surviving root cause. Produces a report, not a PR. |
| dashboard-builder | Finds available data + existing dashboard patterns, specs panels, implements, validates queries, opens a PR. |
| docs-writer | Finds relevant code + doc patterns, drafts an outline, writes, checks examples run and links resolve, opens a PR. |
| issue-triage / CI-fix | GitHub-action-triggered. |

## How it maps to this repo

The Workflow runtime is not one of the 11 existing patterns — it is the **substrate** they could all be written *in*. `parallel()` is [flat-map-reduce](../../patterns/flat-map-reduce.md); `pipeline()` is [pipeline-handoff](../../patterns/pipeline-handoff-swarm.md); the refute-each-hypothesis debug workflow is a [verifier-loop](../../patterns/verifier-loop.md) fanned out. The contribution is the *form*: a deterministic script with memoised resume. See [runtime-primitives.md](../../patterns/runtime-primitives.md) — this is a concrete, shipping implementation of those design notes.

**Takeaway for a hand-rolled orchestrator:** make your orchestration a pure, deterministic function. Ban clocks and RNG inside it. Then "resume" costs nothing — re-run from the top, memoise each task by its input. Checkpointing is what you do *instead* when you can't guarantee purity; Claude Code shows the cheaper path is to guarantee purity.
