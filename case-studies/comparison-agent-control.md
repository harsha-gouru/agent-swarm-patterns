# Comparison: How Agentic CLIs Control Subagents

The hard part of multi-agent orchestration isn't the *pattern* (parent spawns children, waits, reduces) — every tool agrees on that. The hard part is **control**: stopping two agents from corrupting the same file, deciding what context a child gets, and giving the parent a way to check / stop / inspect a running child.

This compares four tools on exactly those mechanics:

- **Claude Code** 2.1.x — Anthropic
- **Cursor "Grind"** 3.4.x — Anysphere
- **Grok CLI** 0.1.21x — xAI
- **Codex CLI** 0.121.x — OpenAI

All facts below are verified against the installed tools' configs, docs, and (for Cursor and Claude Code) the RE'd binary. Verbatim snippets are quoted with their source path.

---

## TL;DR — two camps

**Codex is not a multi-agent orchestrator.** It's a single-agent tool with the strongest *blast-radius* control of the four: OS-level sandboxing (Seatbelt on macOS, Landlock on Linux) plus an approval policy. It controls one agent tightly. It does not spawn-and-monitor a fleet.

**Claude Code, Cursor, and Grok** are real orchestrators. They differ mostly in how much control they hand the parent.

| | Claude Code | Cursor Grind | Grok CLI | Codex CLI |
|---|---|---|---|---|
| Multi-agent? | Yes (Task tool + Workflow runtime) | Yes (Grind swarm) | Yes (`task` tool) | Not really |
| File-conflict strategy | opt-in worktree | always-isolated git clones | opt-in git worktree | single agent + OS sandbox |
| Context to child | fresh **or** fork | prompt + AGENTS.md + handoff | fresh / fork / resume | AGENTS.md |
| Mid-run inspect | no for subagents; task board + events for teammates | continuation hooks | **live Tasks Pane** | n/a |
| Recursion | teammates can't recurse; workflows capped at 1000 agents | coordinator only | depth-limited | n/a |
| Config format | YAML + JSON | bundle + manifests | TOML | TOML + JSON |

---

## 1. File locking / write-conflict avoidance

**None of them use a lock.** A lock across LLM agents doesn't work — an agent can hold a file open for minutes mid-reasoning. They all use **isolation** instead.

### Claude Code — opt-in worktree

The `Task` tool takes an `isolation` parameter:

> With `isolation: "worktree"`, the worktree is automatically cleaned up if the agent makes no changes; otherwise the path and branch are returned in the result.

Default is **no isolation** — subagents share the parent's `cwd`. Two write-capable subagents in the same directory **can and will** corrupt each other. The fix is explicit: pass `isolation: "worktree"` on every spawn that writes.

```
Agent({
  description: "...",
  prompt: "...",
  isolation: "worktree"      // ← temp git worktree, isolated working tree
})
```

### Cursor Grind — always-isolated git clones

Every worker gets its own git clone. Coordination is git itself. From the worker system prompt (RE'd from `cursor-agent-exec/main.js`):

```
git pull --rebase
git push

When Push Fails: Other workers are pushing too.
1. Don't panic — this is normal
2. Wait with backoff before retrying (1st: 2–4s, 2nd: 4–8s, 3rd: 8–16s, 4th+: 16–32s)
3. Rebase again: git pull --rebase
4. Resolve any conflicts
5. Push again: git push
```

Conflict resolution is delegated to the worker LLM, with an explicit instruction *not* to take the lazy path:

```
Do NOT blindly accept your changes or theirs. Do NOT leave conflict markers.
Do NOT force push. Always rebase, never merge.
```

Cursor also has `shared` and `shared-no-git` isolation modes for lighter swarms, but the default Grind mode is full-clone isolation.

### Grok CLI — opt-in git worktree, tracked in a DB

From `~/.grok/docs/user-guide/15-subagents.md`:

> For tasks that modify files, subagents can run in an isolated git worktree. This prevents the child from conflicting with the parent's file changes:
> - Each child gets its own copy of the working tree
> - Changes are isolated until explicitly merged back
> - The parent can review and apply changes via `x.ai/git/worktree/apply`

Worktrees are tracked in a SQLite DB (`~/.grok/worktrees.db`):

```sql
CREATE TABLE worktrees (
    id TEXT PRIMARY KEY,
    path TEXT UNIQUE NOT NULL,
    source_repo TEXT NOT NULL,
    session_id TEXT,
    creator_pid INTEGER,
    status TEXT NOT NULL DEFAULT 'alive',
    ...
);
```

So the parent has a queryable registry of every child's working tree, its owning session, and whether it's still alive.

### Codex — OS sandbox, single agent

Codex doesn't isolate *agents from each other* — it isolates *the agent from your machine*. `codex --help`:

```
-s, --sandbox <SANDBOX_MODE>
    Select the sandbox policy to use when executing model-generated shell commands
--dangerously-bypass-approvals-and-sandbox
    Skip all confirmation prompts and execute commands without sandboxing.
    EXTREMELY DANGEROUS.
```

`sandbox_mode` in `~/.codex/config.toml` is one of `read-only` / `workspace-write` / `danger-full-access`. There's a dedicated `codex sandbox` subcommand. This is Seatbelt/Landlock process confinement — it stops the agent escaping the workspace, not two agents stomping one file.

### Verdict

The convergent answer is **one git worktree per writing subagent**. If you're hand-rolling orchestration:

- Spawn each writer in `git worktree add ../wt-<id> -b agent/<id>`.
- Let it commit freely in its own tree.
- Reduce by merging branches (or rebasing) in the parent — never by reading dirty files.
- Cursor's lesson: make the *agent* resolve conflicts by synthesis, not the runtime by `--ours`/`--theirs`.

---

## 2. What context to give a child

### Claude Code — fresh or fork (it has both)

The `Task`/`Agent` tool description (2.1.148) is explicit: *"specify a `subagent_type` to use a specialized agent, **or omit it to fork yourself — a fork inherits your full conversation context.**"* So there are two modes:

- **Fresh** (`subagent_type` set) — the child gets **only the prompt you write**. No history, no parent state. Anthropic's guidance: *"Brief the agent like a smart colleague who just walked into the room — it hasn't seen this conversation."* No context pollution, but you carry the full briefing burden.
- **Fork** (`subagent_type` omitted) — the child inherits the parent's **full conversation history**. The prompt is then a *directive* (what to do), not background. This is the equivalent of Grok's `fork_context`.

Background spawns and named teammates are fresh-only. Earlier notes calling Claude Code "fresh only" predate the fork mode — it is no longer accurate.

### Cursor Grind — prompt + repo files + handoff

A worker gets its `prompt`, a shared `base_prompt`, and is told to read two files itself:

```
2. Read and follow AGENTS.md — contains repo-wide rules you MUST obey
3. Read <instructionsFile> — may contain workstream-specific rules and context
```

Upward context flows through `handoff.md` — the worker's one-shot dying message to the planner. Children never talk sideways.

### Grok CLI — three modes (the richest)

From `15-subagents.md`:

| Mode | What the child gets |
|---|---|
| fresh (default) | only the prompt |
| `fork_context` | a *copy* of the parent's full conversation history |
| `resume_from` | continues *another subagent's* session — picks up its full context |

> Use `fork_context` when the child needs to understand the broader conversation. Skip it when the task is self-contained.
>
> The `resume_from` parameter lets a new subagent continue from where a previous subagent left off ... Spawn a researcher to investigate a problem; spawn an implementer with `resume_from` set to the researcher's session, so it picks up with full research context.

Grok also formalizes **persona IO contracts** so chaining is typed:

> - **implementer**: Reads a `review_file` with issues to fix, writes a `summary_file` with what was changed.
> - **researcher**: Explores based on a prompt, writes findings to `summary_file`.
>
> These contracts allow chaining: a researcher's `summary_file` becomes the input context for an implementer.

### Codex — AGENTS.md

Codex reads `AGENTS.md` (or `CLAUDE.md` if no AGENTS.md) for project context, plus the global `instructions` / `developer_instructions` in `config.toml`. Single agent, so there's no parent→child handoff problem.

### Verdict

Three context strategies, pick by task shape:

- **Self-contained task** → fresh prompt (Claude Code style). Cheapest, no pollution. Brief fully.
- **Child needs the conversation** → fork/copy history (Grok `fork_context`).
- **Multi-stage pipeline** → resume/chain (Grok `resume_from`, or Cursor's handoff-file relay). Typed IO contracts (researcher writes `summary_file`, implementer reads it) make this reliable.

Never assume the child knows what you know. The most common orchestration bug is a one-line prompt to a context-less agent.

---

## 3. Control: stop / check / inspect a running child

This is where the four differ most.

### Claude Code — depends on the layer

Claude Code is no longer a single "spawn-and-wait" tool — control differs by which of its three layers you use (see [claude-code/architecture.md](claude-code/architecture.md)).

**Layer 1 — subagents: block-and-wait.** You spawn, you wait. The parent receives **only the child's final return message** — intermediate tool calls are invisible.

- **foreground** — parent blocks until the child returns.
- **`run_in_background: true`** — parent continues; gets a notification on completion. *"do NOT sleep, poll, or proactively check on its progress."*

No mid-run inspect of a running subagent. Stop control exists only for *shell* background tasks (`KillShell`).

**Layer 2 — teammates: event-driven coordination.** Named teammates coordinate through a shared **task board** — `TaskCreate / TaskGet / TaskList / TaskUpdate` — where tasks carry `status`, an `owner` (agent id), and a `blockedBy` dependency graph. Lifecycle hook events (`TeammateIdle`, `TaskCreated`, `TaskCompleted`) let the parent (or a plugin) *subscribe* instead of poll. This is much closer to Cursor's lifecycle-hook model than to block-and-wait.

**Layer 3 — workflows: a `SubagentStop` hook.** Inside the workflow runtime the parent is a script; it gates each child structurally via the `SubagentStop` hook and reads typed return values. Still no mid-*run* peek, but completion is fully programmable.

So the older "weakest, block-and-wait" verdict only holds for Layer 1. With teammates, Claude Code has a real event-driven control surface.

### Cursor Grind — continuation policy + lifecycle hooks

Cursor's runtime watches each subagent and nudges it if it stalls:

```js
continuationPolicy: {
  idleThreshold: 3,      // 3 turns with no tool call → inject a nudge
  maxLoops: 100,         // hard cap → escape hatch
  continuationMessage: ({ isEscapeHatch, idleCount }) => ...
}
```

And it fires hooks the parent (or a plugin) can intercept:

```
subagentStart { conversation_id, subagent_id, subagent_type, task,
                parent_conversation_id, model, is_parallel_worker, git_branch }
subagentStop  { subagent_id, status, duration_ms, summary, message_count,
                tool_call_count, error_message, modified_files, loop_count }
```

So the control surface is *event-driven*: you don't poll, you subscribe. `subagentStop` carries `modified_files` — the parent knows exactly what the child touched.

### Grok CLI — live Tasks Pane (the reference design)

Grok is the only one with a real-time UI for the agent tree. From `15-subagents.md`:

> In the interactive TUI, press `Ctrl+T` to toggle the TODO/task panel, which shows:
> - Active and completed subagent tasks
> - Task lineage (parent-child relationships)
> - Task status (running, completed, failed)
>
> Press `Ctrl+Shift+A` to toggle the subagent catalog.

Plus depth limits stop runaway spawning:

> Subagents can spawn their own subagents, creating a tree of workers. To prevent runaway spawning, there is a configurable depth limit. By default, subagents cannot nest beyond a few levels deep.

If mid-run visibility is your pain point, **Grok's model is the one to copy**: the parent owns a registry keyed by child ID, each entry carries `{ status, parent_id, lineage }`, and there's a view that renders the tree live.

### Codex — approval gates

Codex's control is pre-execution, not mid-run: `approval_policy` decides when the *single* agent must stop and ask a human before running a command. `~/.codex/config.toml`:

```toml
approval_policy = "never"          # or "on-failure", "on-request", "untrusted"
sandbox_mode = "danger-full-access"
approvals_reviewer = "user"
```

`on-failure` is the useful middle ground — auto-run, but escalate to the human if a command fails.

### Verdict

The control spectrum:

```
weakest ────────────────────────────────────────────────────────────► strongest
CC subagents     Codex           CC teammates / Cursor          Grok
block-and-wait   approval gate   lifecycle hooks / task board   live Tasks Pane
final msg only   (1 agent)       + idle events                  + depth limit
```

Claude Code spans the spectrum: its Layer-1 subagents are the weakest control surface of any tool here, but its Layer-2 teammates (task board + `TeammateIdle`/`TaskCreated`/`TaskCompleted` events) sit alongside Cursor's lifecycle hooks. Only Grok's live Tasks Pane is clearly ahead.

For a hand-rolled orchestrator, the minimum viable control surface is:

1. **A child registry** — `{ child_id → status, started_at, last_event, worktree_path }`.
2. **Lifecycle events** — child emits `in_progress / blocked / succeeded / failed`; parent subscribes (don't poll).
3. **An idle timer** — if no event in N seconds, nudge or kill. (Cursor's `idleThreshold`.)
4. **A loop cap** — `maxLoops` so a stuck child can't burn forever.
5. **A depth cap** — if children can recurse, limit the tree depth. (Grok.)
6. **A status view** — even just a printed table. You cannot orchestrate what you can't see.

---

## 4. Config surface — where the manifests live

### Claude Code — YAML frontmatter + JSON settings

Agent definitions are markdown files with YAML frontmatter, in `.claude/agents/*.md`:

```markdown
---
name: codex-review
description: Runs Codex CLI to review code changes, catch bugs, and get a second opinion
tools: Bash, Read, Glob, Grep
model: haiku
---

You are a bridge agent that invokes OpenAI Codex CLI to get a second opinion...
```

The frontmatter is the contract (`name`, `description`, `tools` allowlist, `model`); the markdown body is the system prompt. Hooks and permissions live in `settings.json` (JSON).

### Cursor Grind — compiled registry + plugin manifests

Built-in subagent types are compiled into the `cursor-agent-exec` bundle. Custom ones come from plugin manifests that bundle:

```
{ skills, subagents, hooks, rules, mcpServers, commands, variables }
```

A plugin custom subagent declares `preserveTaskTool`, `permissionMode`, `isBackground`, `forceDefaultModel`, `prompt`. Less user-hackable than the others — you author a plugin, not a file.

### Grok CLI — TOML everywhere

`~/.grok/config.toml`:

```toml
[subagents]
enabled = true
default_model = "grok-4.3"        # overrides everything below

[subagents.toggle]
explore = true
plan = false                      # disable a built-in type

[subagents.models]
explore = "grok-build"            # route a type to a lighter model

[subagents.roles.researcher]
description = "Deep research agent"
default_capability_mode = "read-only"
model = "grok-build"
prompt_file = ".grok/prompts/researcher.md"

[subagents.personas.concise]
instructions = "Be extremely concise. No filler words."
```

Roles and personas are also discovered from `.grok/roles/*.toml` and `.grok/personas/*.toml`. **Fail-closed**: a requested persona that doesn't exist makes the spawn *fail*, not silently degrade.

### Codex — TOML + AGENTS.md + JSON hooks

`~/.codex/config.toml` carries `approval_policy`, `sandbox_mode`, `instructions`, `multi_agent`, `child_agents_md`, per-project blocks. `AGENTS.md` is the project-context file. `hooks.json` is JSON.

### Verdict

| Tool | Agent def | Hooks | Project context |
|---|---|---|---|
| Claude Code | YAML frontmatter `.md` | JSON `settings.json` | `CLAUDE.md` |
| Cursor | compiled / plugin manifest | bundle | `AGENTS.md` |
| Grok | TOML (`config.toml` + `.grok/roles/*.toml`) | hooks | `.grok` project rules |
| Codex | TOML (`config.toml`) | JSON `hooks.json` | `AGENTS.md` / `CLAUDE.md` |

If you're designing your own: **one file per agent, with frontmatter for the machine-readable contract and a prose body for the prompt** (Claude Code's model) is the most forkable. TOML tables (Grok) are cleaner when you have many knobs per role. Avoid Cursor's compiled-in approach unless you control the whole product.

---

## 5. Guardrails — the actual knobs

| Guardrail | Claude Code | Cursor | Grok | Codex |
|---|---|---|---|---|
| Tool allowlist | `tools:` frontmatter (+ `disallowedTools` denylist) | `toolsOverride` | capability mode | — |
| Capability tiers | `permissionMode` (`default`/`plan`/`acceptEdits`/…) | `permissionMode` (readonly / agent_only) | `read-only` / `read-write` / `execute` / `all` | — |
| Recursion control | teammates can't recurse; workflow **1000-agent lifetime cap** | `preserveTaskTool` (coordinator only) | configurable **depth limit** | n/a |
| Turn / spend cap | `maxTurns`, `maxBudgetUsd`/`taskBudget`, workflow `budget.remaining()` | `maxLoops` | — | — |
| Model lock | `model:` frontmatter | `forceDefaultModel` | `default_model` (overrides all) | `model` |
| Fail-closed | invalid frontmatter / unknown `agentType` → spawn errors | — | missing persona → spawn fails | — |
| OS sandbox | Linux seccomp sandbox (`[Sandbox Linux]`) | — | `~/.grok/docs/.../17-sandbox.md` | **Seatbelt / Landlock** |
| Human approval | permission prompts; permission rules can deny `agent(agentType)` | confirmation cards | — | `approval_policy` |

Two patterns worth stealing:

- **Capability modes** (Grok): `read-only` / `read-write` / `execute` / `all`. An `explore` agent defaults to `read-only` and *cannot* write or run commands even if its prompt tells it to. This is enforced at the tool layer, not the prompt layer — the right place. Prompt-level "please don't write files" is not a guardrail.
- **Fail-closed resolution** (Grok): if a spawn references a persona/role that doesn't exist, the spawn fails. The alternative — silently spawning a generic agent — produces a child that looks right and behaves wrong. Fail loud.

---

## The synthesized recipe

If you're hand-rolling an orchestrator and getting stuck on exactly the things in this doc:

1. **Isolation** — each writing child runs in its own `git worktree`. Parent reduces by merging branches, never by reading another agent's dirty files.
2. **Context** — fresh prompt for self-contained work; copy history (`fork_context`) when the child needs the conversation; chain via typed handoff files (`summary_file` → next agent's input) for pipelines.
3. **Control** — keep a child registry `{ id → status, worktree, last_event }`. Children emit lifecycle events (`in_progress / blocked / succeeded / failed`); parent subscribes, doesn't poll. Add an idle timer + a loop cap + a recursion-depth cap.
4. **Config** — one file per agent type: machine-readable frontmatter (`name`, `tools`/capability, `model`) + a prose system prompt. Hooks separate.
5. **Guardrails** — enforce capability at the *tool layer* (an explore agent literally lacks Write), not the prompt layer. Fail-closed on missing roles. Lock the model if cost matters.

The parent's job is not to do the work. It's to **own the registry, own verification, and own the merge**. Children are disposable; the parent is the one source of truth.

---

*Verified against: Claude Code 2.1.148 (RE'd from the native binary — see [claude-code/](claude-code/README.md)), Cursor 3.4.20 (`cursor-agent-exec` bundle), Grok CLI 0.1.214 (`~/.grok/docs/user-guide/16-subagents.md` + RE'd binary — see [grok-goal-mode/](grok-goal-mode/README.md)), Codex CLI 0.121.0 (`~/.codex/config.toml`, `codex --help`). Claude Code, Cursor, and Grok Goal-mode internals are RE'd; the rest are from shipped docs/configs.*

> **For the full Claude Code design** — the 3-layer architecture, agent-definition format, workflow runtime, and hook/guardrail surface — see the dedicated [Claude Code case study](claude-code/README.md). For Grok's productized verifier-gated swarm, see the [Grok Goal Mode case study](grok-goal-mode/README.md). This comparison covers only the cross-tool axes.
