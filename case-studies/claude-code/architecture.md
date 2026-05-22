# Claude Code: Agent Architecture

Claude Code's orchestration is three layers. This file covers the shape of all three, then the mechanics shared by every spawn: the agent-definition format, where definitions live, how tools are granted, what context a child gets, and isolation.

## The 3-layer shape

```text
┌─────────────────────────────────────────────────────────────────┐
│  MAIN LOOP  (the conversation you type into)                      │
│                                                                   │
│   Task / Agent tool                Workflow tool                  │
│        │                                │                         │
│        │ subagent_type?                 │ scriptPath               │
│        ▼                                ▼                         │
│  ┌───────────────┐              ┌─────────────────────┐           │
│  │ LAYER 1       │              │ LAYER 3             │           │
│  │ SUBAGENT      │              │ WORKFLOW            │           │
│  │ one-shot      │              │ sandboxed JS script │           │
│  │ returns string│              │ runInContext()      │           │
│  └───────────────┘              │   agent()           │           │
│        │                        │   parallel()        │──┐        │
│        │ + name/team_name/mode   │   pipeline()        │  │ spawns │
│        ▼                        │   phase()           │  │ many   │
│  ┌───────────────┐              └─────────────────────┘  │        │
│  │ LAYER 2       │◄──── shared task board ◄───────────────┘        │
│  │ TEAMMATE      │      TaskCreate/Get/List/Update                 │
│  │ persistent    │      blockedBy dependency graph                 │
│  │ peer          │      owner = agent_id                           │
│  └───────────────┘                                                 │
└─────────────────────────────────────────────────────────────────┘
       teammates CANNOT spawn teammates  (depth capped at 1)
       workflows capped at 1000 agents lifetime
```

- **Layer 1 — Subagent.** The `Task`/`Agent` tool. Spawn, the parent blocks (or `run_in_background: true` → notified on completion). The parent receives **only the child's final text** — intermediate tool calls are invisible. This is the classic map-reduce primitive.
- **Layer 2 — Teammate.** Same tool, plus `name` / `team_name` / `mode`. A teammate is a *named, persistent* agent that lives across turns and coordinates with peers through a shared task board. The internal runner is logged as `[inProcessRunner] Starting agent loop`; teammates are `in-process-teammate`, and a `team-lead` coordinates. **A teammate cannot spawn another teammate** — `name`/`team_name`/`mode` are unavailable to a child, so the tree depth is hard-capped at 1.
- **Layer 3 — Workflow.** The `Workflow()` tool runs a JS script that *calls* `agent()` itself. See [workflows-runtime.md](workflows-runtime.md).

## The agent definition

Every spawn resolves a static **agent definition** object. Recovered shape:

```js
{
  agentType,        // unique id, e.g. "Explore", "code-reviewer"
  whenToUse,        // routing hint shown to the parent model
  getSystemPrompt,  // () => the agent's system prompt (the file body)
  tools,            // granted tool list — see "Tool granting" below
  disallowedTools,  // denylist, subtracted from `tools`
  source,           // "projectSettings" | "userSettings" | plugin
  permissionMode,   // "default" | "plan" | "acceptEdits" | ...
  model,            // model override
  effort,           // reasoning effort
  maxTurns,         // hard turn cap
  background,       // run detached by default
  memory,           // agent-memory mode
  isolation,        // "inherit" | "worktree"
  hooks,            // agent-scoped hooks
  mcpServers,       // which MCP servers this agent sees
}
```

## Where definitions live — source precedence

Agent definitions are Markdown files with YAML frontmatter. They are loaded from three sources; a later source overrides an earlier one on `agentType` collision:

```text
   Plugin agents      from installed plugins
        ▼  (overridden by)
   Personal agents    ~/.claude/agents/*.md      source: userSettings
        ▼  (overridden by)
   Project agents     .claude/agents/*.md        source: projectSettings
   + built-ins        general-purpose, Explore, Plan, ...
```

The file format — frontmatter is the machine-readable contract, the body is the system prompt:

```markdown
---
name: codex-review
description: Runs Codex CLI to review code changes and get a second opinion
tools: Bash, Read, Glob, Grep
model: haiku
permissionMode: default
isolation: worktree
maxTurns: 20
---

You are a bridge agent that invokes the Codex CLI ...   ← becomes getSystemPrompt()
```

> ⚠️ **Plugin-agent demotion.** For agents supplied by a *plugin*, the `permissionMode`, `hooks`, and `mcpServers` fields are **silently ignored** — the binary says verbatim: *"which is ignored for plugin agents. Use `.claude/agents/` for this level of control."* A plugin cannot grant itself elevated permissions; only a project- or user-authored file can.

## Tool granting — the one rule that matters

This is the single most important line recovered from the binary:

```js
tools: T?.tools ? T9([...T.tools, sG, Vi, ee, qX, XQ, GZ, tG]) : ["*"]
```

Read it as a decision:

```text
            does the definition declare `tools:` ?
                   │                        │
                  YES                       NO
                   │                        │
                   ▼                        ▼
   exactly that list  +  7 fixed       ["*"]  = inherit EVERY tool
   baseline tools always appended      the parent currently has
   (StructuredOutput, task-mgmt,
    etc. — so a child can always
    return a result)
                   │
                   ▼
        then subtract `disallowedTools`
```

Consequences worth stealing for a hand-rolled orchestrator:

- **Declaring `tools:` is a deny-by-default switch.** The moment a definition lists *anything*, every unlisted tool is gone.
- **You cannot accidentally make a useless agent** — the 7-tool baseline (structured output + task primitives) is always injected, so a child can always report back.
- Capability is enforced at the **tool layer**, not the prompt. An `Explore` agent with a read-only tool set *cannot* write even if its prompt tells it to.

## Context modes — fresh vs fork

The `Task`/`Agent` tool description states it directly:

> *"specify a `subagent_type` to use a specialized agent, **or omit it to fork yourself — a fork inherits your full conversation context.**"*

```text
   subagent_type SET            subagent_type OMITTED
   ┌──────────────────┐         ┌──────────────────────┐
   │ FRESH            │         │ FORK                 │
   │ zero context     │         │ inherits the parent's│
   │ "brief like a    │         │ full conversation    │
   │  stranger who    │         │ history              │
   │  walked in"      │         │ prompt = a directive,│
   │ prompt carries   │         │ not background       │
   │ the full brief   │         │                      │
   └──────────────────┘         └──────────────────────┘
```

Use **fresh** for self-contained work (no context pollution, but you carry the briefing burden). Use **fork** when the child genuinely needs the conversation — the prompt then says *what to do*, not *what the situation is*. Background spawns and named teammates are **fresh only**.

## Isolation

`isolation` is per-spawn:

| Value | Effect |
|---|---|
| `inherit` (default) | Child shares the parent's `cwd`. Two write-capable children here **will corrupt each other.** |
| `worktree` | Child gets its own temporary `git worktree` + branch. Auto-cleaned if the child makes no changes; otherwise the path and branch are returned in the result for the parent to merge. |

The repo rule [best-of-n-worktree](../../patterns/best-of-n-worktree.md) is exactly this: every *writing* child gets `isolation: "worktree"`, and the parent reduces by merging branches — never by reading another agent's dirty files.

## The flow, end to end

```text
1. parent model decides to delegate
2. resolve agent definition  (plugin ◄ user ◄ project precedence)
3. compute tool set          (declared list + baseline  OR  ["*"])  − disallowedTools
4. pick context mode         (subagent_type → fresh | omitted → fork)
5. pick isolation            (inherit | worktree)
6. PreToolUse hook fires     → may block the spawn entirely
7. child runs its agent loop (bounded by maxTurns / maxBudgetUsd)
8. SubagentStop hook fires   → may inject context or block completion
9. parent receives ONLY the final text  (or structured output if a schema was set)
10. parent verifies, then merges  ← the parent is the single source of truth
```

Step 9 is the hard constraint: the parent never sees the child's intermediate work. Everything the swarm needs to coordinate on must flow through the **task board** (Layer 2) or **return values / worktree branches** — never through hoping the parent watched.
