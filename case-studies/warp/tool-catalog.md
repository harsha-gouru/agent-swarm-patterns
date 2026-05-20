# Warp Agent Tool Catalog

The lead agent's tool surface is `ToolCall.tool` — a oneof in `task.proto` line 367. **35 tool variants.** Field tags are stable wire tags.

## All 35 tools

| Tag | Tool | Read-only? | Notes |
|---:|---|:-:|---|
| 2 | `RunShellCommand` | ✗ | Carries `RiskCategory`; `wait_until_complete` optional; deprecated `is_read_only` / `is_risky` booleans |
| 3 | `SearchCodebase` | ✓ | Semantic / embedding search |
| 4 | `Server` | — | Server-side opaque tool; client just round-trips the `payload` string |
| 5 | `ReadFiles` | ✓ | |
| 6 | `ApplyFileDiffs` | ✗ | Write |
| 7 | `SuggestPlan` | ✓ | Plan creation/edit suggestion to user |
| 8 | `SuggestCreatePlan` | ✓ | |
| 9 | `Grep` | ✓ | |
| 10 | `FileGlob` | ✓ | **DEPRECATED** — use FileGlobV2 |
| 11 | `ReadMCPResource` | ✓ | |
| 12 | `CallMCPTool` | varies | Wraps an arbitrary MCP tool call |
| 13 | `WriteToLongRunningShellCommand` | ✗ | Send input to a running PTY; modes: `raw` / `line` |
| 14 | `SuggestNewConversation` | — | Suggest a fresh conversation/handoff |
| 15 | `FileGlobV2` | ✓ | Replaces tag 10 |
| 16 | `SuggestPrompt` | — | Suggest a follow-up prompt to the user |
| 17 | `OpenCodeReview` | — | Open code review UI |
| 18 | `InitProject` | ✗ | First-time project init |
| 19 | `Subagent` | varies | **DEPRECATED v0** — 7 metadata variants (research/advice/computer_use/summarization/conversation_search/warp_documentation_search/cli) |
| 20 | `ReadDocuments` | ✓ | |
| 21 | `EditDocuments` | ✗ | |
| 22 | `CreateDocuments` | ✗ | |
| 23 | `ReadShellCommandOutput` | ✓ | Read PTY scrollback |
| 24 | `UseComputer` | ✗ | Computer-use (browser/visual) |
| 25 | `InsertReviewComments` | ✗ | |
| 26 | `ReadSkill` | ✓ | Load a skill into context |
| 27 | `RequestComputerUse` | — | Ask for computer-use enablement |
| 28 | `FetchConversation` | ✓ | **Cross-agent read** — fetch another conversation's tasks |
| 29 | `StartAgent` | ✗ | **v1 spawn** (single child) |
| 30 | `SendMessageToAgent` | ✗ | **Broadcast** to N children by `addresses[]` |
| 31 | `TransferShellCommandControlToUser` | — | Hand a PTY back to the user |
| 32 | `AskUserQuestion` | — | Modal user question |
| 33 | `StartAgentV2` | ✗ | **Current single-child spawn** with skill + harness + computer_use overrides |
| 34 | `UploadFileArtifact` | ✗ | Upload artifact file |
| 35 | `RunAgents` | ✗ | **Batch swarm spawn** with shared run-wide config |

## Categorization

### Code-editing surface

`ReadFiles`, `ApplyFileDiffs`, `Grep`, `FileGlobV2`, `SearchCodebase`, `EditDocuments`, `CreateDocuments`, `ReadDocuments`, `ReadShellCommandOutput`, `RunShellCommand`, `WriteToLongRunningShellCommand`, `InitProject`

Standard set. Comparable to Claude Code's Read/Edit/Write/Bash/Grep/Glob.

### Spawn primitives (the four)

`Subagent` (deprecated) · `StartAgent` (v1) · `StartAgentV2` (current single) · `RunAgents` (current batch). See `spawn-and-messaging.md`.

### Inter-agent communication

`SendMessageToAgent` (broadcast) · `FetchConversation` (read another agent's history).

### User-interaction

`AskUserQuestion` · `SuggestPrompt` · `SuggestPlan` · `SuggestCreatePlan` · `SuggestNewConversation` · `TransferShellCommandControlToUser` · `OpenCodeReview` · `RequestComputerUse`.

These are all **suggestions** to the UI layer rather than agent-side commits. The agent says "I'd like to open a code review"; the client renders it as an interactive card.

### MCP integration

`ReadMCPResource` · `CallMCPTool`. Two-tool MCP surface, same shape as Anthropic's MCP spec.

### Computer use

`UseComputer` · `RequestComputerUse`. Browser / visual interaction. Per-call enablement via `computer_use_enabled` on Remote execution modes.

### Skills

`ReadSkill` is the only direct skill tool. Skills are otherwise passed by reference (`SkillRef`) in `StartAgentV2.Remote.skills` and `RunAgents.skills`, and loaded into context when the agent decides to consult them.

### Review & artifacts

`OpenCodeReview` · `InsertReviewComments` · `UploadFileArtifact`. Review-loop primitives.

## Tool-result mapping

Each tool has a corresponding result message. Results are matched to calls by `tool_call_id`:

| Tool | Result |
|---|---|
| RunShellCommand | RunShellCommandResult (+ ShellCommandFinished async event) |
| ReadFiles | ReadFilesResult |
| SearchCodebase | SearchCodebaseResult |
| ApplyFileDiffs | ApplyFileDiffsResult |
| SuggestCreatePlan | SuggestCreatePlanResult |
| SuggestPlan | SuggestPlanResult |
| Grep | GrepResult |
| FileGlobV2 | FileGlobV2Result |
| ReadMCPResource | ReadMCPResourceResult (carries MCPResourceContent) |
| CallMCPTool | CallMCPToolResult |
| WriteToLongRunningShellCommand | WriteToLongRunningShellCommandResult |
| SuggestNewConversation | SuggestNewConversationResult |
| SuggestPrompt | SuggestPromptResult |
| OpenCodeReview | OpenCodeReviewResult |
| InitProject | InitProjectResult |
| ReadDocuments | ReadDocumentsResult |
| EditDocuments | EditDocumentsResult |
| CreateDocuments | CreateDocumentsResult |
| ReadShellCommandOutput | ReadShellCommandOutputResult |
| UseComputer | UseComputerResult |
| InsertReviewComments | InsertReviewCommentsResult |
| ReadSkill | ReadSkillResult |
| RequestComputerUse | RequestComputerUseResult |
| FetchConversation | FetchConversationResult |
| StartAgent | StartAgentResult |
| SendMessageToAgent | SendMessageToAgentResult |
| TransferShellCommandControlToUser | TransferShellCommandControlToUserResult |
| AskUserQuestion | AskUserQuestionResult |
| StartAgentV2 | StartAgentV2Result |
| UploadFileArtifact | UploadFileArtifactResult |
| RunAgents | RunAgentsResult |

`ShellCommandFinished` is special — it arrives asynchronously after the initial `RunShellCommandResult`, carrying final stats. Long-running commands stream multiple events.

## Risk classification (`RiskCategory`)

`RunShellCommand` carries a `RiskCategory` field that replaces the deprecated booleans. The schema fragment from `task.proto` shows this is the finer-grained risk axis:
- `is_read_only` boolean ✗ deprecated
- `is_risky` boolean ✗ deprecated
- `risk_category: RiskCategory` ✓ current

The enum values aren't in the snippet I read, but the pattern matches industry standard: read-only / write / dangerous / external-effect. Lets the client prompt the user differently for each category.

## What's missing

Notable absences compared to Antigravity and Cursor:
- **No `Reflect` tool.** Cursor has one for meta-troubleshooting. Warp doesn't.
- **No `TodoWrite` or task-list manipulation tool.** There's a `Task` message and `todo.proto` (35 lines), but the agent doesn't have a direct write-the-todo-list tool. Plans are user-suggested via `SuggestPlan`.
- **No git-specific tools.** Git is just `RunShellCommand`.
- **No browser tools** beyond the abstract `UseComputer` / `RequestComputerUse`. Cursor has 13+ specialized browser subagent types; Antigravity has a full Chrome DevTools MCP. Warp delegates to the harness.
- **No web search tool.** Either via MCP or via the agent's reasoning.

The story Warp tells with its tool catalog: **stay thin, push specialization to harnesses and MCP servers**. The agent surface is generic primitives + the spawn machinery + the user-interaction primitives. Everything domain-specific lives in:
- the **harness** (Oz / ClaudeCode / OpenCode / Gemini / Codex) — picked per-spawn
- the **skills** (Apache 2.0 markdown bundle) — user-authored
- the **MCP servers** — third-party

That's an inversion of Cursor's approach (rich first-party subagent types) and Antigravity's (rich first-party role compositions).
