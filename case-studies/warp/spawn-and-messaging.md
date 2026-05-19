# Spawn + Messaging Primitives

All quoted from `warp-proto-apis/apis/multi_agent/v1/task.proto`. Line refs are for the version pinned at `9a2d2425c5d1eb40fb547956e659511906e03f9e` (referenced from `warp/Cargo.toml`).

## The four spawn primitives (live in `ToolCall.tool` oneof)

| Tag | Tool | Status | Purpose |
|---:|---|---|---|
| 19 | `Subagent` | deprecated v0 | Pre-StartAgent spawn with 7 baked-in variants (research / advice / computer_use / summarization / etc.) |
| 29 | `StartAgent` | v1 | Single-child spawn, simpler `ExecutionMode` (Local empty / Remote{environment_id}) |
| 33 | `StartAgentV2` | current | Single-child spawn with per-child harness override + skills + computer_use + worker_host |
| 35 | `RunAgents` | current | Batch swarm spawn with shared run-wide config (model, harness, execution_mode, skills, base_prompt) |

### Why four

The progression `Subagent → StartAgent → StartAgentV2` is a schema-evolution trail:
- `Subagent` over-specialized — 7 metadata variants embedded the type-of-work into the wire format.
- `StartAgent` collapsed to a generic spawn (`name + prompt + lifecycle subscription + execution mode`).
- `StartAgentV2` added per-child overrides (skills, model_id, harness, computer_use_enabled, title, worker_host).
- `RunAgents` is the batch variant of `StartAgentV2`, with run-wide fields hoisted out.

## `StartAgentV2`: single-child spawn

```protobuf
message StartAgentV2 {
  string name = 1;                                    // display name
  string prompt = 2;                                  // initial prompt
  LifecycleSubscription lifecycle_subscription = 3;   // optional
  ExecutionMode execution_mode = 4;                   // Local default

  message LifecycleSubscription {
    repeated LifecycleEventType event_types = 1;
    // Omitted → subscribe to all.
    // Present with empty list → subscribe to none.
  }

  message ExecutionMode {
    oneof mode { Local local = 1; Remote remote = 2; }
    message Local { Harness harness = 1; }           // optional harness override
    message Remote {
      string environment_id = 1;
      repeated SkillRef skills = 2;                   // pre-load skills
      string model_id = 3;                            // override model
      bool computer_use_enabled = 4;                  // need browser/visual?
      string worker_host = 5;                         // pin to a worker
      Harness harness = 6;
      string title = 7;
    }
    message Harness { string type = 1; }              // "oz" | "claude" | ...
  }
}

message StartAgentV2Result {
  oneof result {
    Success success = 1;                              // { agent_id }
    Error error = 2;
  }
}
```

Verbatim comment on the lifecycle subscription:

> Optional lifecycle event subscription for the new child agent. If omitted, defaults to all lifecycle event types. If present with an empty event_types list, subscribes to none.

Three states for the same field:
- **omitted** → subscribe to all (default)
- **present with empty list** → subscribe to none (fire-and-forget)
- **present with specific types** → subscribe to those events only

This three-state pattern (default vs explicit-none vs explicit-some) is the standard protobuf workaround for the lack of nullable-list semantics.

## `RunAgents`: batch swarm spawn

```protobuf
message RunAgents {
  // Shared by all children
  string summary = 1;                               // human-readable batch desc
  string base_prompt = 2;                           // base prompt prepended
  repeated SkillRef skills = 3;                     // shared skills
  string model_id = 4;
  Harness harness = 5;                              // ENUM form (not string)
  oneof execution_mode { Local local = 6; Remote remote = 7; }
  string plan_id = 9;                               // for auto-launch matching

  // Per-child
  repeated AgentRunConfig agent_run_configs = 8;
  message AgentRunConfig {
    string name = 1;                                // unique within batch
    string prompt = 2;
    string title = 3;
  }

  message Local {}
  message Remote {
    string environment_id = 1;
    string worker_host = 2;
    bool computer_use_enabled = 3;
  }
}
```

Compared to `StartAgentV2`:
- Harness uses the **enum** type (`orchestration.proto`'s `Harness`), not the string variant. So a `RunAgents` batch is uniform-harness.
- Per-child config carries **only** name + prompt + title. No per-child model or harness override — everyone in the batch shares.
- `computer_use_enabled` is **batch-wide** on `Remote`. The LLM decides this per-call: "set per call based on whether the children need computer use (browser automation, visual inspection, etc.). Not a user preference."
- `plan_id` links the batch to a plan-attached `OrchestrationConfigSnapshot`. If the snapshot's status is `Approved` and the batch matches, **auto-launch** without confirmation prompt.

### `RunAgentsResult`: three top-level outcomes, per-child sub-outcomes

```protobuf
message RunAgentsResult {
  oneof outcome {
    Launched launched = 1;                          // call accepted; per-agent below
    Denied denied = 2;                              // user/server declined non-error
    Failure failure = 3;                            // validation rejected
  }
  message Launched {
    string resolved_model_id = 1;                   // ← actual model used
    Harness resolved_harness = 2;                   // ← actual harness
    oneof resolved_execution_mode { ... }
    repeated AgentOutcome agents = 5;
  }
  message AgentOutcome {
    string name = 1;
    oneof result {
      LaunchedAgent launched = 2;                   // { agent_id }
      FailedAgent failed = 3;                       // { error }
    }
  }
  message Denied { string reason = 1; }
  message Failure { string error = 1; }
}
```

Three-level outcome encoding is worth borrowing:
- **Failure**: the call never started (validation, schema rejection). All-or-nothing.
- **Denied**: user declined, or some non-error refusal. Carries a reason string.
- **Launched**: call accepted. **Resolved** fields are returned so client knows what config actually ran (e.g., if it inherited from the plan's `OrchestrationConfig`).
- Within Launched, each `AgentOutcome` independently succeeded or failed. Partial-failure is normal.

## Inter-agent messaging: `SendMessageToAgent`

```protobuf
message SendMessageToAgent {
  repeated string addresses = 1;                    // RECIPIENTS — broadcast
  string subject = 2;
  string message = 3;
}

message SendMessageToAgentResult {
  oneof result {
    Success success = 1;                            // { message_id }
    Error error = 2;
  }
}
```

**`addresses` is repeated** — one call broadcasts to N children. Each child sees the same `subject` + `message`. The pattern:

```
parent ──► SendMessageToAgent { addresses: ["child-A", "child-B", "child-C"],
                                 subject: "scope change",
                                 message: "now also handle the auth path" }
```

No reply pump in the API. Children respond by emitting their own lifecycle events or by calling `SendMessageToAgent` back at the parent's `agent_id`. Async message passing.

## Cross-conversation reads: `FetchConversation`

```protobuf
// A tool call to fetch a conversation's tasks from the client.
// Used by subagents that need to access a different conversation's data.
message FetchConversation {
  string conversation_id = 1;
}
```

A subagent calls this to **read another conversation's task history**. Use cases inferred from the schema:
- A reviewer subagent reading the original implementer's tasks.
- A coordinator reading a child's progress without messaging it.
- A spawned analyzer reading the conversation that produced a result.

This is structured cross-agent context sharing — Cursor handles this via shared git branches, Antigravity via the `.agents/` filesystem. Warp does it via an explicit API call routed through the client.

## Agent event flow

```protobuf
message AgentEvent {
  string event_id = 1;                              // idempotency key
  google.protobuf.Timestamp occurred_at = 2;
  oneof event { LifecycleEvent lifecycle_event = 3; }

  message LifecycleEvent {
    string sender_agent_id = 1;
    oneof detail {
      Errored errored = 2;                          // pipeline-stage failure
      Empty cancelled = 6;
      Blocked blocked = 7;                          // { blocked_action }
      Empty in_progress = 8;
      Empty succeeded = 9;
      Failed failed = 10;                           // logical failure
    }

    message Errored {
      string stage = 1;                             // which stage of the pipeline
      string reason = 2;
      string error_message = 3;
    }
    message Blocked { string blocked_action = 1; }
    message Failed {
      string reason = 1;
      string error_message = 2;
    }
  }
}
```

`event_id` carries idempotency — clients dedupe on this. `occurred_at` timestamps when the agent emitted the event, not when the parent received it.

The `Blocked` variant is the actionable one: child says "I'm stuck on `blocked_action`", parent can choose to unblock via `SendMessageToAgent` or kill via cancel.

## Putting it together — what a swarm run looks like

```text
1. User chats with lead agent.
2. Lead agent emits RunAgents tool call:
       summary: "build the API layer"
       base_prompt: "..."
       skills: [SkillRef("claude-api"), SkillRef("oz-platform")]
       model_id: "" (inherit from plan config)
       harness: ClaudeCode
       execution_mode: Remote { environment_id: "env-prod-1",
                                 worker_host: "",
                                 computer_use_enabled: false }
       agent_run_configs: [
         { name: "auth", prompt: "implement /auth endpoints", title: "Auth worker" },
         { name: "data", prompt: "implement /data endpoints", title: "Data worker" },
         { name: "test", prompt: "write integration tests",  title: "Test worker" }
       ]
       plan_id: "plan-42"

3. Server checks matches_active_config("plan-42", request) — matches → auto-launch.

4. Server responds with RunAgentsResult.Launched {
       resolved_model_id: "claude-opus-4-7",
       resolved_harness: ClaudeCode,
       resolved_execution_mode: Remote { ... },
       agents: [
         { name: "auth", launched: { agent_id: "agent-001" } },
         { name: "data", launched: { agent_id: "agent-002" } },
         { name: "test", failed: { error: "no environment capacity" } }
       ]
   }

5. Each launched child runs in a remote ClaudeCode environment. Each emits
   IN_PROGRESS → ... → SUCCEEDED (or BLOCKED, FAILED, etc.) AgentEvents
   back to the lead.

6. Lead agent receives events for whichever lifecycle types it subscribed to
   on each child. If the lead emitted RunAgents (which has no per-child
   subscription), default subscription = all events.

7. Lead can SendMessageToAgent { addresses: ["agent-001", "agent-002"], ... }
   to broadcast clarifications.

8. Lead can FetchConversation { conversation_id: <child's conv ID> } to
   inspect what a child did.
```

## Anti-patterns flagged in the schema comments

- **Don't put specialization in the wire format.** The deprecated `Subagent` with 7 metadata variants is a cautionary example — moved to prompt-encoded variants.
- **Names are correlation keys, not display strings.** `AgentRunConfig.name` must be unique within a batch; the server uses it to route `AgentOutcome` back. Don't repurpose for UI.
- **`computer_use_enabled` is LLM-decided per-call, not user preference** (verbatim comment). A child needing browser access marks itself; not a setting.
- **`is_read_only` and `is_risky` on `RunShellCommand` are deprecated** — use `RiskCategory` instead. Finer-grained classification beats booleans.
- **Three-state lifecycle subscription** (omitted / empty / specific) — pattern worth borrowing for any "subscribe to events" field.
