# Warp Agent Architecture

All quoted material is verbatim from `warp-proto-apis/apis/multi_agent/v1/` and `warp/crates/ai/src/agent/`.

## Shape

```text
                    ┌──────────────────────────────────┐
                    │  Lead Agent (user-facing)        │
                    │  Runs in user's Warp terminal    │
                    └──────────┬───────────────────────┘
                               │
                  emits ToolCall.tool = ...
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
    StartAgent             StartAgentV2           RunAgents
   (single, v1)            (single, current)   (batched swarm)
        │                      │                      │
        │  with optional       │  with optional       │  with N AgentRunConfigs
        │  LifecycleSubscription                      │  sharing run-wide
        │  (which events to be │                      │  model_id, harness,
        │   notified about)    │                      │  execution_mode, skills
        │                      │                      │
        ▼                      ▼                      ▼
      ┌────────────────────────────────────────────────────┐
      │  OrchestrationConfig                               │
      │  • model_id                                        │
      │  • harness ∈ { Oz, ClaudeCode, OpenCode,           │
      │                Gemini, Codex }                     │
      │  • execution_mode ∈ { Local, Remote{env, host} }   │
      └────────────────────────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
       ┌──────────────┐                  ┌──────────────┐
       │ Local child  │                  │ Remote child │
       │ same machine │                  │ environment_id│
       │ same harness │                  │ worker_host   │
       └──────────────┘                  │ computer_use? │
                                         └──────────────┘
              │                                 │
              └────────────────┬────────────────┘
                               │
              child emits LifecycleEvent ──► parent (only if subscribed)
                  IN_PROGRESS, SUCCEEDED, FAILED, ERRORED,
                  CANCELLED, BLOCKED

           parent can:
              SendMessageToAgent (broadcast to N children by addresses[])
              FetchConversation (read another agent's tasks)
```

## The orchestration config (verbatim)

`warp-proto-apis/apis/multi_agent/v1/orchestration.proto:14`:

```protobuf
message Harness {
  oneof variant {
    Oz oz = 1;
    ClaudeCode claude_code = 2;
    OpenCode open_code = 3;
    Gemini gemini = 4;
    Codex codex = 5;
  }
  message Oz {}
  message ClaudeCode {}
  message OpenCode {}
  message Gemini {}
  message Codex {}
}

message OrchestrationConfig {
  message Local {}
  message Remote {
    string environment_id = 1;
    string worker_host = 2;
  }
  string model_id = 1;
  Harness harness = 2;
  oneof execution_mode {
    Local local = 3;
    Remote remote = 4;
  }
}

message OrchestrationStatus {
  message Approved {}
  message Disapproved {}
  oneof status {
    Approved approved = 1;
    Disapproved disapproved = 2;
  }
}
```

**The five harnesses are empty oneof variants** — they don't carry per-variant config (yet). The Rust side dispatches based on which variant is set; the harness-specific behavior is implemented client-side. Anyone adding a harness adds one line to this oneof.

## The plan-attached snapshot pattern

```protobuf
message OrchestrationConfigUpdate {
  string plan_id = 1;
  OrchestrationConfig config = 2;
  OrchestrationStatus status = 3;
}

message OrchestrationConfigSnapshot {
  string plan_id = 1;
  OrchestrationConfig config = 2;
  OrchestrationStatus status = 3;
}
```

- The user edits an orchestration card attached to a plan → client emits `OrchestrationConfigUpdate` on its next outbound request.
- The server records an `OrchestrationConfigSnapshot` in conversation history.
- The active config = the most-recent snapshot's config + status.

## The auto-launch rule

From `warp/crates/ai/src/agent/orchestration_config.rs`:

```rust
/// Returns `true` when the `run_agents` call's run-wide fields match
/// the active approved `OrchestrationConfig`, meaning the confirmation
/// card can be skipped (auto-launch).
///
/// Empty/unset fields on the call are treated as inheriting from the
/// config (and therefore matching). Fields not in the config
/// (`computer_use_enabled`, `skills`, `base_prompt`, `agent_run_configs`,
/// per-agent `title`) are excluded from the check.
pub fn matches_active_config(request: &RunAgentsRequest, config: &OrchestrationConfig) -> bool {
    // model_id — empty on the call means "inherit from config" → matches.
    if !request.model_id.is_empty() && request.model_id != config.model_id { return false; }
    // harness_type
    if !request.harness_type.is_empty() && request.harness_type != config.harness_type { return false; }
    // execution_mode variant must agree.
    // ...
}
```

So the user approves a config **once**, attached to a plan. Subsequent `RunAgents` calls inside that plan that match the config (or leave fields blank to inherit) auto-launch without re-prompting. This is **trust by plan**, not trust by call.

Fields explicitly **excluded** from the match (so they can vary call-to-call without breaking auto-approve):
- `computer_use_enabled` — set per-call by the LLM based on whether children need browser/visual
- `skills` — list of `SkillRef`s to pre-load
- `base_prompt` — shared base prompt
- `agent_run_configs` — the per-child list itself
- per-agent `title`

## Lifecycle events (the agent state machine)

```protobuf
enum LifecycleEventType {
  LIFECYCLE_EVENT_TYPE_UNSPECIFIED   = 0;
  LIFECYCLE_EVENT_TYPE_STARTED       = 1 [deprecated = true];   // → IN_PROGRESS
  LIFECYCLE_EVENT_TYPE_IDLE          = 2 [deprecated = true];   // → SUCCEEDED
  LIFECYCLE_EVENT_TYPE_RESTARTED     = 3 [deprecated = true];   // → IN_PROGRESS
  LIFECYCLE_EVENT_TYPE_ERRORED       = 4;
  LIFECYCLE_EVENT_TYPE_CANCELLED     = 5;
  LIFECYCLE_EVENT_TYPE_BLOCKED       = 6;
  LIFECYCLE_EVENT_TYPE_IN_PROGRESS   = 7;
  LIFECYCLE_EVENT_TYPE_SUCCEEDED     = 8;
  LIFECYCLE_EVENT_TYPE_FAILED        = 9;
}
```

The three deprecated states (STARTED, IDLE, RESTARTED) collapsed into IN_PROGRESS and SUCCEEDED. "Restart is no longer distinguished from start" (verbatim comment) — a clean schema lesson.

Each event carries:

```protobuf
message AgentEvent {
  string event_id = 1;                            // idempotency
  google.protobuf.Timestamp occurred_at = 2;
  oneof event { LifecycleEvent lifecycle_event = 3; }

  message LifecycleEvent {
    string sender_agent_id = 1;
    oneof detail {
      Errored errored = 2;                        // { stage, reason, error_message }
      Empty cancelled = 6;
      Blocked blocked = 7;                        // { blocked_action }
      Empty in_progress = 8;
      Empty succeeded = 9;
      Failed failed = 10;                         // { reason, error_message }
    }
  }
}
```

**Errored vs Failed** — distinct: Errored is a pipeline stage failure (with `stage` + `reason`), Failed is a logical task failure (`reason` + `error_message`, no stage).

**Blocked** is interesting: it carries `blocked_action` describing what the agent is stuck waiting on. Parent can choose to unblock by calling `SendMessageToAgent`.

## The three spawn primitives (current)

### `StartAgentV2` — spawn one child

```protobuf
message StartAgentV2 {
  string name = 1;
  string prompt = 2;
  LifecycleSubscription lifecycle_subscription = 3;
  ExecutionMode execution_mode = 4;

  message LifecycleSubscription {
    repeated LifecycleEventType event_types = 1;
    // Empty list = subscribe to none. Omitted = subscribe to all.
  }

  message ExecutionMode {
    oneof mode { Local local = 1; Remote remote = 2; }
    message Local {
      Harness harness = 1;                        // override per-child harness
    }
    message Remote {
      string environment_id = 1;
      repeated SkillRef skills = 2;               // pre-load remote skills
      string model_id = 3;
      bool computer_use_enabled = 4;
      string worker_host = 5;
      Harness harness = 6;
      string title = 7;
    }
    message Harness { string type = 1; }          // "oz" or "claude" — STRING, not enum here
  }
}
```

Note: `ExecutionMode.Harness` is a **string-typed** harness (`"oz"`, `"claude"`, etc.), distinct from the enum `Harness` in `orchestration.proto`. This is the per-child override path; the orchestration enum is the plan-attached config path. Two parallel selection surfaces for the same concept.

### `RunAgents` — spawn a swarm

```protobuf
message RunAgents {
  message Local {}
  message Remote {
    string environment_id = 1;
    string worker_host = 2;
    bool computer_use_enabled = 3;
  }
  message AgentRunConfig {
    string name = 1;          // unique within this batch
    string prompt = 2;
    string title = 3;
  }
  string summary = 1;                              // top-level human-readable
  string base_prompt = 2;                          // shared across all children
  repeated SkillRef skills = 3;                    // shared skill bundle
  string model_id = 4;                             // shared
  Harness harness = 5;                             // shared (enum form)
  oneof execution_mode { Local local = 6; Remote remote = 7; }
  repeated AgentRunConfig agent_run_configs = 8;   // ← the N children
  string plan_id = 9;                              // for auto-launch matching
}
```

Server rejects on `RunAgentsResult.Failure { error: "agent names must be non-empty and unique" }` if names collide. **Names are the correlation key** — `AgentOutcome.name` matches `AgentRunConfig.name`.

### `RunAgentsResult` — three outcomes

```protobuf
message RunAgentsResult {
  oneof outcome {
    Launched launched = 1;                         // success — per-agent outcomes inside
    Denied denied = 2;                             // user said no
    Failure failure = 3;                           // server-side validation rejected
  }

  message Launched {
    string resolved_model_id = 1;
    Harness resolved_harness = 2;
    oneof resolved_execution_mode { ... }
    repeated AgentOutcome agents = 5;
  }
  message AgentOutcome {
    string name = 1;
    oneof result {
      LaunchedAgent launched = 2;                  // { agent_id }
      FailedAgent failed = 3;                      // { error }
    }
  }
  message Denied { string reason = 1; }
  message Failure { string error = 1; }
}
```

**Per-agent partial failure is encoded inside `Launched`** — the call succeeded overall, but some individual children failed to spawn. `Failure` is reserved for validation rejections that prevent the call from even starting. This is the right way to model swarm spawns.

## The deprecated `Subagent` (v0)

```protobuf
message Subagent {
  string task_id = 1;
  string payload = 2;
  oneof metadata {
    CLISubagent cli = 3;                           // wraps a shell command
    Empty research = 4;
    Empty advice = 5;
    Empty computer_use = 6;
    Empty summarization = 7;
    ConversationSearchMetadata conversation_search = 8;
    Empty warp_documentation_search = 9;
  }
}
```

The legacy `Subagent` had **7 specialized variants** baked into the protocol — research, advice, computer_use, summarization, etc. These all collapsed into the generic `StartAgent` → `StartAgentV2` → `RunAgents` model, where the variant is encoded in the prompt rather than the schema.

This is a useful design lesson: specialized subagent types in the wire format become technical debt. Warp moved to "subagent = name + prompt + harness + execution mode" and pushed specialization to the prompt layer.

## Where everything lives

```text
warp-proto-apis/apis/multi_agent/v1/
├── orchestration.proto       67 lines   ← Harness enum, OrchestrationConfig, snapshot/status
├── task.proto                1963       ← Task, ToolCall (35 tools), spawn primitives,
│                                          AgentEvent + LifecycleEventType
├── request.proto              577       ← Client → server request envelope
├── response.proto             333       ← Server → client response envelope
├── skill.proto                 64       ← SkillRef
├── attachment.proto           162       ← File/image/diff attachments
├── input_context.proto        149
├── conversation_data.proto    ...
├── citations.proto            ...
├── todo.proto                  35
├── lsp.proto                  ...
├── file_content.proto          37
├── document_content.proto     ...
├── options.proto              ...
└── suggestions.proto          ...

warp/crates/ai/src/agent/
├── mod.rs                              ← module root
├── orchestration_config.rs             ← matches_active_config, approval status
├── orchestration_config_tests.rs       ← test coverage of the match rule
├── action/                             ← client-side action representations
├── action_result/                      ← client-side result representations
├── convert.rs                          ← proto ↔ client-type bridges
├── citation.rs                         ← AIAgentCitation
└── file_locations.rs                   ← FileLocations, group_file_contexts_for_display

warp/crates/ai/                         25,384 lines of Rust across the AI crate
```

## What's not in the open code

A few things are referenced but their implementations live elsewhere:

- **The harness clients** (Oz, ClaudeCode, OpenCode, Gemini, Codex). Warp ships only the orchestration protocol; the actual transport for each harness is a separate concern. The `claude_code` variant presumably wraps `claude -p` headless calls (consistent with the skill-bundle's `improve_description.py` pattern); `gemini` and `codex` similarly wrap those CLIs.
- **The server-side agent runner.** The `worker_host` in `Remote` execution mode points at a Warp-managed worker; that worker's implementation isn't in the open source.
- **The system prompt** for the lead agent. Skills are bundled (Apache 2.0) but the agent's base system prompt is fetched at runtime, not in the binary plaintext or the open repo.

## Bottom line

Warp's agentic flow is:

1. User chats with a **lead agent** in the terminal.
2. Lead agent has a **35-tool surface** (see `tool-catalog.md`), four of which are spawn primitives.
3. Spawning is **harness-aware** — the parent picks Oz / ClaudeCode / OpenCode / Gemini / Codex per spawn (or inherits from a plan-attached `OrchestrationConfig`).
4. Children run **Local** or **Remote** with explicit `environment_id` + `worker_host` pinning.
5. Children emit **lifecycle events**; parents subscribe to specific event types.
6. Parents can **broadcast messages** to children (`SendMessageToAgent.addresses`) and **read other agents' conversations** (`FetchConversation`).
7. **Skills** (Apache-2.0 markdown) are user-extensible content layered on top — orthogonal to the spawn machinery.

This is more like a **kernel** for multi-agent orchestration than a single specific agent loop.
