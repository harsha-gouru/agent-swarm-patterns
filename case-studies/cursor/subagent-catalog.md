# Cursor Subagent Catalog

Cursor exposes a `Task` tool that spawns a subagent of a chosen `subagent_type`. Subagent types are registered as static config objects in `cursor-agent-exec/dist/main.js` with the shape:

```js
{
  subagent_type: new nd._L({ type: { case: "...", value: new ... } }),
  description: "<one-liner shown to the parent agent>",
  preserveTaskTool: boolean,     // can THIS subagent itself spawn Task subagents?
  permissionMode: READONLY | AGENT_ONLY | undefined,
  systemReminder: () => "...",   // short prompt fragment appended to base
  systemPromptOverride: (...) => "...",  // full prompt replacement (rare)
  toolsOverride: (...) => [...], // restrict tool surface
  defaultModelIds: ["composer-2", ...],
  forceDefaultModel: boolean,    // block parent's model override
  resumeModeOverride: NONE | LAST_AGENT_SAME_TYPE,
  conversationStateMapper: ...,
  messageHistoryModifier: e => filteredHistory,
  continuationPolicy: { idleThreshold, maxLoops, continuationMessage },
  resultSuffix: "<text appended to subagent's result>"
}
```

What follows are the **verified built-in subagent types** in Cursor 3.4.20, recovered from the same file.

---

## 1. `worker-agent` (custom)

> "Worker that implements a single task and reports back. Uses a shared workspace; do not run git commands unless explicitly requested. Use for concrete implementation work."

- `preserveTaskTool: false` — cannot recursively spawn
- `systemPromptOverride: D1` — gets the full "You Are a Worker" prompt
- `forceDefaultModel: true` — composer-2 only, no overrides
- `continuationPolicy: { idleThreshold: 3, maxLoops: 100 }`

**Spawn pattern:** `Task(subagent_type="worker-agent", prompt="...", run_in_background=true)`

## 2. `coordinator-agent` (custom)

> "Autonomous planner that explores the codebase and delegates work to workers and sub-planners. Uses a shared workspace; avoid git operations unless explicitly requested. Use to start a grind-swarm or to delegate a scoped area (include scope in the prompt)."

- `preserveTaskTool: true` — **CAN** recursively spawn workers + sub-planners
- `systemPromptOverride: Q1` — full "You Are the Planner" prompt
- `forceDefaultModel: true`
- Same continuation policy as worker

**Spawn pattern:** `Task(subagent_type="coordinator-agent", prompt="...", run_in_background=true)`

These two are the **Grind primitives**. Everything else is more specialized.

---

## 3. `explore` (built-in type)

> "Fast, readonly agent specialized for exploring codebases. Use this when you need to quickly find files by patterns (eg. 'src/components/**/*.tsx'), search code for keywords (eg. 'API endpoints'), or answer questions about the codebase (eg. 'how do API endpoints work?'). This agent operates in read-only mode and cannot modify files. When calling this agent, specify the desired thoroughness level: 'quick' for basic searches, 'medium' for moderate exploration, or 'very thorough' for comprehensive analysis."

- `permissionMode: READONLY` — hard-blocks all writes

**System reminder (verbatim):**

```
You are a file search specialist for Cursor, an application to write code with AI.
You excel at thoroughly navigating and exploring codebases.

=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
This is a READ-ONLY exploration task. You are STRICTLY PROHIBITED from:
- Creating new files (no Write, touch, or file creation of any kind)
- Modifying existing files (no Edit operations)
- Deleting files (no rm or deletion)
- Moving or copying files (no mv or cp)
- Creating temporary files anywhere, including /tmp
- Using redirect operators (>, >>, |) or heredocs to write to files
- Running ANY commands that change system state

Your role is EXCLUSIVELY to search and analyze existing code.
```

## 4. `generalPurpose` (built-in type, alias: `L1`)

> "General-purpose agent for researching complex questions, searching for code, and executing multi-step tasks. Use when searching for a keyword or file and not confident you'll find the match quickly."

The catch-all when nothing more specific fits.

## 5. `computerUse`

> "Perform manual testing of built applications and code. This subagent has access to the computer and browser to test the application. This subagent_type is stateful; if a computerUse subagent already exists, the previously created subagent will be resumed if you reuse the Task tool with subagent_type set to computerUse."

- `resumeModeOverride: LAST_AGENT_SAME_TYPE` — stateful, reuses the prior session
- `defaultModelIds: [cuaModel, ...]` — model is overridable via `cuaModel` config
- `forceDefaultModel: true`
- `systemReminder: () => "Enter COMPUTER USE mode. Follow the provided instruction above."`
- Filters message history to remove `Reflect` tool calls

## 6. `browserUse`

> "Perform browser-based testing and web automation. This subagent can navigate web pages, interact with elements, fill forms, and take screenshots. Use this for testing web applications, verifying UI changes, or any browser-based tasks. ... This subagent_type is stateful; if a browserUse subagent already exists, the previously created subagent will be resumed if you reuse the Task tool with subagent_type set to browserUse."

- `permissionMode: AGENT_ONLY` — requires agent mode (browser MCP unavailable in readonly)
- `resumeModeOverride: LAST_AGENT_SAME_TYPE` — stateful
- Requires `subagentInstanceId`; throws if missing

## 7. Debug subagent (built-in type)

> "You are a debugging specialist operating in **DEBUG MODE**. You must debug with **runtime evidence**."

**Critical Constraints (verbatim, rendered as JSX into the prompt):**
- NEVER fix without runtime evidence first
- ALWAYS rely on runtime information + code (never code alone)
- Do NOT remove instrumentation before post-fix verification logs prove success and caller confirms there are no more issues
- Fixes often fail — iteration is expected and preferred. Taking longer with more data yields better, more precise fixes

- `resumeModeOverride: LAST_AGENT_SAME_TYPE`
- `resultSuffix:` "The debug subagent has finished.
  - After reproducing the bug: 'Issue reproduced, please proceed. <additional notes>'
  - To clean up debug logs: '<The issue has been fixed.> Please clean up the instrumentation.'"

## 8. `pastConversationExplorer` (custom)

> "MUST USE for any question about agent behavior, failures, patterns, or user work history. Trigger immediately when user asks: 'what have you been failing at', 'what are your problems', 'what mistakes do you make', 'what have I been working on', 'what did I do last week', 'show me recent work', 'how do you usually handle X', 'what are your common errors', 'why do you keep failing at Y'."

Searches transcripts at `/opt/cursor/past-transcripts/` (cloud agent sessions for the user/team).

**Index metadata fields (verbatim):**
- `bcId` — unique conversation identifier
- `name` — conversation name/title
- `status` — FINISHED, RUNNING, FAILED, etc.
- `createdAtMs` — timestamp
- `filePath` — path to the transcript file
- `messageCount`
- `source` — `"user"` (personal) or `"team"` (shared team conversations)

Use cases (verbatim): Find prior solutions · Understand patterns · Project context · Learn conventions · Reference past work · Self-reflection queries.

## 9. `cursorBlameLearning` (custom)

> "Produces story-style learning reports about how code was built — who changed it, why, what tradeoffs were discussed, and what future engineers should know. Use when the user explicitly asks about history, evolution, authorship, rationale, or deep file/module context."

Reads git history + `CODE_LINEAGE` evidence to write narrative reports.

## 10. `vmSetup` (likely custom)

> "Codebase analysis helper for VM environment setup. Use this to explore the codebase structure, find setup scripts, discover dependencies, and analyze configuration. Ideal for parallel discovery tasks."

## 11. Video subagent

> "Describe or analyze videos with an expert video description and analysis model."

Distinct model for video analysis.

## 12. Command execution subagent

> "Command execution specialist for running bash commands. Use this for git operations, command execution, and other terminal tasks."

**System reminder (verbatim):**

```
You are a command execution specialist. Your role is to execute shell commands
efficiently and safely.

Guidelines:
- Execute commands precisely as instructed
- For git operations, follow git safety protocols
- Report command output clearly and concisely
- If a command fails, explain the error and suggest solutions
- Use command chaining (&&) for dependent operations
- Quote paths with spaces properly
- For clear communication, avoid using emojis

Complete the requested operations efficiently.
```

Tools filtered to `SHELL` only (see `F1 = new Set(["SHELL"])` and `P1(e,t,n)` filter function).

## 13. `cursorGuide` (built-in type)

> "Read Cursor product documentation to answer questions about how Cursor Desktop, IDE, CLI, Cloud Agents, Bugbot, and other features work. Use when the user asks 'In Cursor, how do I...?' or similar questions about Cursor products."

- `permissionMode: READONLY`

**System reminder workflow (verbatim):**
1. ALWAYS start by fetching `https://cursor.com/llms.txt` using the web fetch tool. This page contains an overview of all Cursor documentation pages and their URLs.
2. Based on the user's question, identify which documentation pages are relevant.
3. Fetch those specific pages using the web fetch tool to get detailed information.
4. Synthesize the information and provide a clear, accurate answer.

## 14. CI investigator

> "Investigate a single failing PR CI check and return a short root-cause summary. Use when the user asks to summarize, explain, diagnose, or investigate a specific failed check from a pull request."

## 15. `best-of-n-runner`

> "Run a task in an isolated git worktree. Each best-of-n-runner gets its own branch and working directory. Use for best-of-N parallel attempts or isolated experiments."

This is Cursor's first-class implementation of the `best-of-n-worktree` pattern from this repo. Each instance gets a real `git worktree add` directory.

---

## Plugin / Marketplace custom subagents

Plugins distributed via Cursor's marketplace can register additional `custom` subagents. The plugin manifest schema (from `cursor-agent-exec/main.js`):

```js
{
  manifest: { variables, ... },
  skills: [...],
  subagents: [...],
  hooks: [...],
  rules: [...],
  mcpServers: [...],
  commands: [...]
}
```

A custom subagent registered by a plugin gets:

```js
{
  subagent_type: { case: "custom", value: { name: "<plugin-defined name>" } },
  description: "...",
  preserveTaskTool: <plugin choice>,
  permissionMode: <plugin choice>,
  isBackground: <plugin choice>,
  userRequestedModelId: <plugin choice or 'inherit'>,
  forceDefaultModel: <plugin choice>,
  plugin: <plugin ref>,
  marketplace: <marketplace ref>,
  pluginId, marketplaceId,
  prompt: <plugin-provided system prompt>
}
```

So plugins can ship **fully custom subagent types with their own system prompts, permission modes, and tool restrictions** — Cursor's analogue to Antigravity's workspace `.agent/skills/` system, except distributed via a marketplace rather than via Firebase.

## Universal subagent reminder

Every subagent — built-in or custom — gets prepended:

> "You are running as a subagent under a parent agent. Do not spawn additional subagents unless requested by the user or by your instructions."

(`O1` in the bundle.) This prevents accidental fan-out. Only `coordinator-agent` (with `preserveTaskTool: true`) and certain custom subagents can spawn further subagents.

## The fork-and-resume primitive

The Task tool docs reference a separate pattern:

> "with `run_in_background` set to true and resume set to `"self"`. Use the prompt 'You are the forked subagent...'"

This is **agent self-forking**: the agent spawns a copy of itself as a subagent, runs it in the background, and the original continues. Different from spawning a `worker-agent` — the fork inherits the parent's full state and context. Used for "do this side-quest while I keep doing the main thing."

## Hooks: subagentStart / subagentStop

The runtime emits telemetry events at the start and end of every subagent invocation:

```js
case "subagentStart": {
  conversation_id, generation_id, model, subagent_id, subagent_type,
  task, parent_conversation_id, tool_call_id, subagent_model,
  is_parallel_worker, git_branch
}

case "subagentStop": {
  conversation_id, generation_id, model, subagent_id, subagent_type,
  status, duration_ms, summary, parent_conversation_id,
  message_count, tool_call_count, error_message, modified_files,
  git_branch, loop_count, task, description
}
```

Plugin hooks can intercept both events (`hookExecutor.executeHookForStep(_E.subagentStart, t)` / `.subagentStop`). This is the extension point for things like "log every subagent run to my own dashboard" or "block subagents that match a pattern."

## Readonly enforcement

```
const he = "This operation is not allowed in readonly mode. The subagent was
launched with readonly: true, which restricts write operations.";
```

When a subagent is launched with `permissionMode: READONLY`, every write tool returns this error string. Used by `explore`, `cursorGuide`, and any plugin subagent that requests it.
