# Claude Code: Hooks & Guardrails

A swarm is only as safe as the gates around each spawn and each tool call. Claude Code's gate layer has two halves: **hooks** (user-defined interception points) and **guardrails** (runtime-enforced limits). Both were recovered from the 2.1.148 binary.

## Hooks — the 14 events

A hook is a user-configured handler that fires on a lifecycle event. The full event set:

```text
TOOL EVENTS              SESSION EVENTS         SWARM / TASK EVENTS
─────────────            ──────────────         ───────────────────
PreToolUse               SessionStart           SubagentStop
PostToolUse              SessionEnd             TeammateIdle
PermissionRequest *      Stop                   TaskCreated
UserPromptSubmit         PreCompact             TaskCompleted
UserPromptExpansion      PostCompact
Notification
```
(* `PermissionRequest` is the matcher target for Prompt/Agent hooks; the broad event list also includes it.)

The swarm-relevant ones:

| Event | Fires when | Use for |
|---|---|---|
| `PreToolUse` | before any tool call — **including a spawn** | block a spawn, vet a child's prompt |
| `PostToolUse` | after a tool call returns | validate a child's output, inject context |
| `SubagentStop` | a subagent finishes | gate completion, run a verifier |
| `TeammateIdle` | a teammate runs out of work | reassign, or wind the teammate down |
| `TaskCreated` | a task is added to the board | log / approve new work |
| `TaskCompleted` | a task is marked done | trigger the next stage, audit |
| `Stop` | the end of *every* turn (read-only ones too) | end-of-turn checks |
| `PreCompact` / `PostCompact` | around history compaction | a `PreCompact` hook can *block* compaction |

## Hook types — Command, Prompt, Agent

```json
// 1. Command hook — run a shell command
{ "type": "command", "command": "prettier --write $FILE", "timeout": 30 }

// 2. Prompt hook — an LLM evaluates a condition   (tool events only)
{ "type": "prompt", "prompt": "Is this safe? $ARGUMENTS" }

// 3. Agent hook — spawn a whole AGENT with tools to decide   (tool events only)
{ "type": "agent", "prompt": "Verify tests pass: $ARGUMENTS" }
```

The **Agent hook is the notable one for this repo**: the guardrail is itself a spawned agent with `Bash, Write, Edit, Read, Glob, Grep`. A [verifier-loop](../../patterns/verifier-loop.md) doesn't have to be something the parent remembers to run — it can be wired into `PostToolUse` / `SubagentStop` so it fires *structurally* on every relevant event. The gate becomes part of the runtime, not the prompt.

## Hook I/O contract

**Input** — every hook receives JSON on stdin:

```json
{
  "session_id": "abc123",
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/to/file.txt", "content": "..." },
  "tool_response": { "success": true }   // PostToolUse only
}
```

**Output** — a hook returns JSON that *controls the runtime*:

```json
{
  "systemMessage": "Warning shown to the user in the UI",
  "continue": false,                       // false → block / stop the action
  "stopReason": "Message shown when blocking",
  "suppressOutput": false,                 // hide stdout from the transcript
  "decision": "block",                     // block (PostToolUse/Stop/UserPromptSubmit)
  "reason": "Explanation for the decision",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Context injected back into the model"
  }
}
```

Two control levers that matter for orchestration:

- **`continue: false`** — hard stop. On `PreToolUse` this kills a spawn before it starts; on `SubagentStop` it refuses to accept the child's completion.
- **`hookSpecificOutput.additionalContext`** — feeds text *back into the model*. A `PostToolUse` hook can inspect a child's artifact and inject "the child claimed X but file Y is empty" — turning the hook into a fact-checker whose findings the parent must see.

> Matchers can filter tool events by `tool_name` but **not by command content** — there is no way to target only `git commit` via a `Bash` matcher. The binary explicitly routes that to a real `.git/hooks/pre-commit`.

## Guardrails — runtime-enforced limits

These are not user hooks; they are caps the runtime enforces unconditionally.

| Guardrail | Mechanism | Layer |
|---|---|---|
| **Tool allowlist** | `tools:` frontmatter → exactly that + 7 baseline; absent → `["*"]` | per agent |
| **Tool denylist** | `disallowedTools` — subtracted from the granted set | per agent |
| **Permission mode** | `permissionMode` (`default` / `plan` / `acceptEdits` / …) | per agent |
| **Spawn permission** | `agent(agentType)` is itself subject to permission rules — a rule can *deny spawning* a given agent type by source | per spawn |
| **Turn cap** | `maxTurns` — child is killed after N turns | per agent |
| **Spend cap** | `maxBudgetUsd` / `taskBudget`; workflows expose `budget.remaining()` | per agent / workflow |
| **Concurrency** | workflow `agent()` capped at `min(16, cores−2)`, ~10 run at once | per workflow |
| **Runaway backstop** | **1000-agent lifetime cap** per workflow | per workflow |
| **Recursion** | teammates cannot spawn teammates → tree depth capped at 1 | structural |
| **Isolation** | `isolation: "worktree"` → own git worktree; default shares `cwd` | per spawn |
| **Determinism** | `Date.now` / `Math.random` removed inside workflow scripts | per workflow |
| **Spawn discipline** | system-prompt rule: *"Do not spawn agents unless the user asks"* | soft / prompt |

## The gate sequence around one spawn

```text
parent decides to spawn
        │
        ▼
  ┌─────────────────┐   permission rule:  agent(agentType) denied?  ──► refuse
  │  PreToolUse     │   PreToolUse hook:  continue:false?           ──► refuse
  │  gate           │   PreToolUse hook:  Agent hook vets prompt    ──► refuse / amend
  └────────┬────────┘
           ▼
  child runs   ── bounded by maxTurns, maxBudgetUsd, tool allowlist,
           │       permissionMode, isolation
           ▼
  ┌─────────────────┐   SubagentStop hook: continue:false?  ──► reject completion
  │  SubagentStop   │   SubagentStop hook: Agent hook runs a verifier pass
  │  gate           │   PostToolUse hook:  additionalContext ──► inject findings
  └────────┬────────┘
           ▼
  parent receives final text  → verifies → merges
```

## Takeaways for a hand-rolled orchestrator

1. **Enforce capability at the tool layer, never the prompt.** `tools:` / `disallowedTools` make an `Explore` agent *physically unable* to write. "Please don't write files" is not a guardrail.
2. **Make verification structural.** An Agent hook on `SubagentStop` means the verifier fires every time — the parent cannot forget. A verifier-loop the parent has to remember is a verifier-loop that gets skipped under pressure.
3. **Cap everything that can loop.** `maxTurns`, `maxBudgetUsd`, concurrency, and a hard lifetime count. Claude Code ships *all four* — because each is a real failure mode (`tengu_workflow_agent_cap_exceeded` exists for a reason).
4. **Block compaction when state is fragile.** A `PreCompact` hook that returns `continue:false` keeps history intact through a delicate step — orchestration state often lives in the transcript.
5. **Hooks return data, not just exit codes.** `additionalContext` injecting findings back into the model is what turns a hook from a yes/no gate into a participant in the loop.
