# Grind: Cursor's Agent Architecture

Cursor's multi-agent system is named **Grind**. The name appears verbatim in the system prompt that ships in `cursor-agent-exec/dist/main.js`.

## Verbatim opening (recovered from the binary)

> "Grind is an autonomous coding system. You are one of many agents working in parallel."

## Shape

```text
                 ┌──────────────────────┐
                 │     Orchestrator     │   spawns a root planner with a goal
                 └──────────┬───────────┘
                            ▼
                 ┌──────────────────────┐
                 │       Planner        │   "You explore. You identify.
                 │  (root, or sub-)     │    You delegate. You do not implement."
                 └─┬───────────────┬────┘
                   │               │
       submit_task │      spawn_planner
                   ▼               ▼
            ┌─────────────┐  ┌─────────────┐
            │   Worker    │  │ Sub-planner │   (recursive: own clone, own workers)
            │ implements  │  └─────────────┘
            │ + pushes    │
            └──────┬──────┘
                   │
                   ▼
              git push --rebase (retry with backoff)
                   │
                   ▼
            shared branch  ←──────  other workers push concurrently
```

```text
Information Flow (verbatim from the prompt):
  Planners communicate DOWN via task specifications.
  Children communicate UP via handoffs.
  Code flows through git.
```

## The three roles

### Planner

> "You explore. You identify. You delegate. You do not implement."

Tools: `submit_task` (delegate concrete work), `spawn_planner` (delegate a whole area). Read-only on code paths.

**Philosophy (verbatim):**
- "Work autonomously. Think deeply. Get things right. Always be working — never idle."
- "If you finish one area, explore another. If blocked, try a different path."
- "Record questions in the scratchpad but keep moving. Do NOT stop or ask for clarification."
- "If a problem is hard, solve it. If solving it requires first solving three other problems, solve those."
- "Prioritize transformational changes over incremental fixes. If something fundamental is broken, address the architecture, not the surface."

### Worker

> "You implement. You push. Your commit is your contribution."

**Job (verbatim sequence):**
1. Understand your task
2. **Read and follow `AGENTS.md`** — contains repo-wide rules you MUST obey
3. **Read `<instructionsFile>`** — workstream-specific rules and context (templated per swarm)
4. Implement the solution
5. **Verify the build passes** — DO NOT commit code that doesn't compile
6. Make **ONE** commit with all your changes
7. Push to the shared branch (with rebase, retrying as needed)
8. Write `./handoff.md` before finishing

### Sub-planner

Same prompt as Planner but scoped to a specific area. The parent planner delegates the area; the sub-planner owns it recursively.

> "Once spawned, that sub-planner owns the area. Do not spawn workers for areas you've delegated to a sub-planner."

## Isolation modes (the key design choice)

```js
const x1 = {
  agentId: "",
  repoPath: ".",
  branch: "your branch",
  instructionsFile: "AGENTS.md",
  isolationMode: "shared-no-git"
};
```

Three modes referenced in `C1()` (the prompt builder):

| Mode | Each agent gets | Coordination |
|---|---|---|
| **`isolated`** (default for Grind) | Its own git clone, its own branch | Workers push commits to shared branch; planners pull rebased clones to see results |
| **`shared`** | Shared working directory, full git access | Same workspace, agents must avoid stomping each other |
| **`shared-no-git`** | Shared working directory, NO git ops allowed | Simpler; lighter swarm without git overhead |

The Grind prompt only renders when `isolationMode !== shared`. Shared modes get a different (lighter) prompt that omits the git workflow section.

## What survives vs what vanishes

This is the most-quoted insight from the Worker prompt:

> "When you finish, EVERYTHING vanishes: your conversation history, your scratchpad, your local files, any notes you made. **Only your commit survives.**"
>
> "If you discover something important — a pattern, a gotcha, architectural knowledge, a concern — you have one choice: encode it in the codebase."

What encodes survival:

| Channel | What it carries |
|---|---|
| **Tests** | Expected behavior, edge cases, invariants (most important) |
| **Comments** | Context for future readers |
| **Documentation** | README, docs/, inline if it exists |
| **Error messages** | What went wrong and why |
| **Assertions** | Documented invariants |
| **Commit message** | What you did, why, what you learned, concerns/suggestions |

## Local-only files (NEVER commit)

> "These files exist only for your working memory. **NEVER** commit, reference, or mention these files anywhere — not in code, comments, documentation, task descriptions, commit messages, or any output. They do not exist as far as the codebase is concerned."

- `scratchpad.md` — working memory; current progress, decisions, things to remember
- `handoff.md` — the worker's one-shot message to the planner (vanishes with worker)
- `.grind-plan.md` — planner's working notes

## The push retry protocol (verbatim)

```text
git pull --rebase
git push

When Push Fails: Other workers are pushing too. If git push fails:
1. Don't panic — this is normal
2. Wait with backoff before retrying:
     1st: 2–4s
     2nd: 4–8s
     3rd: 8–16s
     4th+: 16–32s
3. Rebase again: git pull --rebase
4. Resolve any conflicts
5. Push again: git push
6. Never give up — keep retrying until you succeed
```

## Conflict resolution philosophy (verbatim)

> "You are a teammate. The other worker's changes are valuable — they were working toward the same goal. Your job is to synthesize both contributions:
>
> 1. Read and understand both sides — what was the branch doing? What were you doing?
> 2. Preserve valuable work from both — don't just pick 'ours' or 'theirs'
> 3. Understand the intent — what was the other person trying to achieve?
> 4. Think holistically — how should this code look with both contributions merged?
>
> Do NOT blindly accept your changes or theirs. Do NOT leave conflict markers. Do NOT force push. Always rebase, never merge."

## Continuation policy (anti-stall)

Coordinator and worker subagents share:

```js
continuationPolicy: {
  idleThreshold: 3,
  maxLoops: 100,
  continuationMessage: ({ isEscapeHatch, idleCount, escapeToken }) =>
    isEscapeHatch ? S1(t, n, "worker") : B1("worker")
}
```

`B1` (the continuation prompt) varies by role:

- **Planner idle**: "Review your scratchpad and submit any remaining tasks. Don't check on any subagent's work unless you were already told it completed."
- **Worker idle**: "Continue working on your task if there are deliverables left for your objective. Make sure you completely implement all tasks to the fullest degree. Otherwise, report what you've done."

So if either role goes idle (no tool calls for 3 turns), the runtime injects a nudge. After 100 loops, the escape hatch activates.

## The handoff (Worker → Planner)

> "Before finishing, write `./handoff.md`. This is your ONE chance to communicate with the planner — your conversation history vanishes after this."

**Required**: What you did. What worked, what didn't. Issues or concerns.

**Also welcome**: Thoughts, ideas, opinions. Things you noticed. Questions you couldn't answer. Suggestions for the broader effort. Concerns, risks, warnings.

> "Don't filter yourself. If the planner should know it, say it."

Note this is the OPPOSITE direction from Antigravity's handoffs (which flow sideways agent-to-agent). In Grind, handoffs are one-shot upward dying messages from a worker to its parent planner.

## Tool catalog (frequency-ranked, from the bundle)

Top-level tool names registered for the main agent:

| Tool | Count | Notes |
|---|---:|---|
| Read | 66 | File read |
| Reflect | 55 | **Novel** — meta-troubleshooting step (see below) |
| Task | 49 | Spawn subagent: `Task(subagent_type=..., prompt=..., run_in_background=true)` |
| Write | 39 | File write |
| Edit | 16 | In-place edit |
| Grep | 15 | Code search |
| Bash | 11 | Shell exec |
| Glob | 8 | File globbing |
| WebSearch | 6 | |
| WebFetch | 5 | |
| TodoWrite | 3 | |

Planner-only verbs: `submit_task`, `spawn_planner`.

Cursor adopted Claude Code's tool-naming convention (Read/Edit/Bash/Grep/Task) wholesale. Their distinctive add is `Reflect`.

### The Reflect tool (verbatim)

```
name: "Reflect"
description: Use the tool to troubleshoot when your actions are not having the
  desired effect. Be sure to make a decision about how to proceed using the
  `next_steps` field. The `Reflect` tool is expensive, so use it sparingly.

Critical Rules:
- You must use the `next_steps` field to describe your decision for what to do next.
- Never use the `Reflect` tool if your previous message was a `Reflect` tool call.
```

This is meta-reasoning as a first-class tool call: the agent explicitly stops, reflects on whether its strategy is working, and writes `next_steps`. Marked "expensive" (likely uses a stronger model). The "never reflect twice in a row" rule prevents reflection loops.

## Spawning rules (verbatim, from the orientation prompt)

> Use `Task(subagent_type="worker-agent", prompt="...", run_in_background=true)` for concrete, implementable work.
>
> Use `Task(subagent_type="coordinator-agent", prompt="...", run_in_background=true)` for areas too large to plan in one pass.
>
> Once spawned, that sub-planner owns the area. Do not spawn workers for areas you've delegated to a sub-planner.
>
> **ALWAYS set `run_in_background=true` when spawning workers and sub-planners.** Never block on a subagent — spawn it and move on.
>
> **Do NOT use the `model` parameter** when spawning workers or sub-planners — swarm agents need the full default model to handle complex, multi-step implementation work.

The "no `model` parameter" rule is enforced by `forceDefaultModel: true` on the worker/coordinator subagent definitions.

## Default model

`composer-2` (and `composer-2-fast`) — Cursor's in-house frontier models. Worker/coordinator subagents force this default and reject user model overrides.

## Working-dir layout (when shared mode is active)

```text
project_root/
├── swarm-agents/
│   └── <subagentInstanceId>/      ← N1(projectFolder, subagentInstanceId)
│       ├── scratchpad.md
│       ├── handoff.md
│       └── .grind-plan.md
├── AGENTS.md                       ← repo-wide mandatory rules
└── <instructionsFile>              ← workstream-specific rules (template var)
```

In isolated mode each agent has its own git clone, so there is no shared `swarm-agents/` directory.

## Recovered prompt-builder symbols

For anyone tracing this in 3.4.20's `main.js`:

| Symbol | Role |
|---|---|
| `C1(e)` | Top-level system-prompt builder; switches on isolation mode |
| `w1()` | Renders the Grind overview section ("Grind is an autonomous coding system...") |
| `E1({instructionsFile})` | Planner-specific body ("You Are the Planner") |
| `v1({branch, instructionsFile})` | Worker-specific body ("You Are a Worker") |
| `T1({agentWorkDir, noGit})` | Shared-mode overview |
| `I1({agentWorkDir, noGit})` | Shared-mode worker body |
| `k1({agentWorkDir, noGit})` | Shared-mode planner body |
| `b1()` / `_1()` | Closing sections |
| `Q1(e,t,n)` | Wraps C1 with `role:"planner"` |
| `D1(e,t,n)` | Wraps C1 with `role:"worker"` |
| `R1(e,t)` | Computes agent work dir |
| `S1(t,n,role)` / `B1(role)` | Continuation-message generators |
| `x1` | Default isolation config object |

Search any of these to verify the prompt content here against your own copy.
