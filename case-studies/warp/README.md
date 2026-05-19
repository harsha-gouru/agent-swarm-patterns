# Case Study: Warp (Multi-Harness Orchestration)

> Sources, all public:
> - `github.com/warpdotdev/warp` — full Rust app, ~59k stars (`crates/ai/src/agent/`)
> - `github.com/warpdotdev/warp-proto-apis` — protobuf API definitions for the agent control plane (`apis/multi_agent/v1/`)
> - `Warp.app/Contents/Resources/bundled/` — Apache-2.0 skill bundle

No reverse engineering required — the entire agentic surface is open-source. This case study is built from `task.proto` (1963 lines), `orchestration.proto` (67 lines), `crates/ai/src/agent/`, and a handful of supporting Rust files.

## What makes Warp different

Antigravity and Cursor are each tied to **one** agent runtime. Warp is a **meta-orchestrator** that dispatches across **five different agent harnesses**:

```protobuf
// orchestration.proto:14
message Harness {
  oneof variant {
    Oz oz = 1;             // Warp's own
    ClaudeCode claude_code = 2;
    OpenCode open_code = 3;
    Gemini gemini = 4;
    Codex codex = 5;
  }
}
```

You pick the harness as part of an `OrchestrationConfig`. The same Warp app then spawns child agents that run inside that chosen harness — whether the user wants Anthropic's Claude Code loop, Google's Gemini CLI, OpenAI's Codex CLI, or Warp's own Oz harness.

## Files

| File | What's in it |
|---|---|
| [architecture.md](architecture.md) | The 5-harness orchestration model, two execution modes (Local / Remote{environment, worker_host}), the lead/child agent topology, where each piece lives in the repo |
| [spawn-and-messaging.md](spawn-and-messaging.md) | The four spawn primitives (`Subagent`, `StartAgent`, `StartAgentV2`, `RunAgents`), lifecycle event subscription, inter-agent `SendMessageToAgent` broadcast, cross-conversation `FetchConversation` |
| [tool-catalog.md](tool-catalog.md) | The 35-tool agent surface from `task.proto`, with which tools are read-only, which spawn subagents, which interact with running shells, and which are deprecated |
| [skills-bundle.md](skills-bundle.md) | The Apache-2.0 skill bundle (`Resources/bundled/skills/`), still worth documenting because that's the user-extensible surface |

## How it maps to this repo

| Warp construct | Pattern in this repo |
|---|---|
| `RunAgents` (batch spawn, shared run-wide config) | [flat-map-reduce](../../patterns/flat-map-reduce.md) |
| `StartAgentV2` with `LifecycleSubscription` | [background-hybrid-swarm](../../patterns/background-hybrid-swarm.md) with explicit event subscription |
| `OrchestrationConfig` + `Approved` status → auto-launch matching `RunAgents` | [pipeline-handoff-swarm](../../patterns/pipeline-handoff-swarm.md) (the plan is the handoff carrier) |
| Per-call harness selection across Oz / ClaudeCode / OpenCode / Gemini / Codex | **NEW** — see [architecture.md](architecture.md) |
| `SendMessageToAgent.addresses` repeated string + `FetchConversation` | A weak [tiered-hierarchical-swarm](../../patterns/tiered-hierarchical-swarm.md) (no enforced tree shape) |
| `RunAgents.Remote.computer_use_enabled` set per-call by the LLM | Capability negotiation, runtime decision |

## What's novel here (the actual agentic-flow distinctives)

1. **Multi-harness dispatch.** Warp is the only one of the three case studies that treats the agent loop itself as pluggable. An `OrchestrationConfig` carries a `Harness` oneof: `Oz | ClaudeCode | OpenCode | Gemini | Codex`. Each child agent in a `RunAgents` batch runs in the configured harness.

2. **Plan-attached orchestration config.** Approval is captured once via `OrchestrationConfigSnapshot` carried in conversation history. Subsequent `RunAgents` calls that match the active config **auto-launch without re-prompting** (`matches_active_config` in `crates/ai/src/agent/orchestration_config.rs`). Empty/unset fields on the call are treated as inheriting from the config.

3. **Lifecycle event subscription.** When spawning a child via `StartAgentV2`, the parent declares which lifecycle events it wants notifications for: `IN_PROGRESS | SUCCEEDED | FAILED | ERRORED | CANCELLED | BLOCKED`. Empty subscription = subscribes to none. This is selective backpressure — the parent decides whether to wait on any signal at all.

4. **Two execution modes baked into the orchestration model:**
   - `Local` — child runs in the user's machine, same harness, same env.
   - `Remote { environment_id, worker_host }` — child runs in a managed cloud environment with explicit worker pinning. `RunAgents.Remote.computer_use_enabled` is set by the LLM per-call based on whether the children need browser/visual capability.

5. **`SendMessageToAgent.addresses repeated string`.** Inter-agent messaging is **broadcast-by-default** — one tool call delivers the same message to N child agents. Each child sees `subject` + `message`. This is multi-cast, not unicast.

6. **Cross-conversation reads via `FetchConversation`.** A subagent can read another conversation's tasks by ID. "Used by subagents that need to access a different conversation's data" (verbatim comment). This is structured cross-agent context sharing — no shared filesystem (Antigravity) or shared git branch (Cursor) needed.

7. **The agent surface is in `.proto`, not in the binary.** Tool definitions, agent-spawn primitives, lifecycle events — all live in `warp-proto-apis/apis/multi_agent/v1/`. The Rust client is generated from these. So the agentic flow is wire-stable across releases, and ChannelVersions-aware — see `crates/channel_versions/`.

## Reproducibility

```sh
# Clone both repos
git clone --depth 1 https://github.com/warpdotdev/warp.git
git clone --depth 1 https://github.com/warpdotdev/warp-proto-apis.git

# The agent control plane (1963 lines of .proto)
ls warp-proto-apis/apis/multi_agent/v1/

# The 5 harnesses
grep -A 8 "^message Harness" warp-proto-apis/apis/multi_agent/v1/orchestration.proto

# The 35 client tools
awk '/^    message ToolCall \{/,/^    \}/' warp-proto-apis/apis/multi_agent/v1/task.proto

# The spawn primitives
grep -A 30 "^message StartAgentV2 \{\|^message RunAgents \{" \
  warp-proto-apis/apis/multi_agent/v1/task.proto

# Rust-side orchestration config + approval logic
cat warp/crates/ai/src/agent/orchestration_config.rs
```

## What I had wrong before

An earlier draft of this case study focused on Warp's **skills** as the headline. That was the wrong axis. Skills (and the bundled `create-skill` meta-skill, and the optimization loop) are the **user-extensibility layer** — they don't tell you anything about how Warp itself spawns and coordinates agents. The agentic flow lives in the protobuf API: `OrchestrationConfig`, `RunAgents`, `StartAgentV2`, `LifecycleEventType`, `Harness`. That's the system. Skills are content shipped on top of it.
