# Case Study: Claude Code

> Reverse-engineered from Claude Code **2.1.148** — the JavaScriptCore-statically-linked native Mach-O at `~/.local/share/claude/versions/2.1.148` (~210 MB). The application JavaScript ships as plain UTF-8 in the binary's `__cstring` data; every quoted string below was recovered statically and is reproducible (see [Reproducibility](#reproducibility)).

Claude Code is the reference orchestrator for this repo — the other case studies (Cursor, Warp, Antigravity) are measured against it. It is also the one that changed the most: as of 2.1.x it is no longer a single "parent spawns subagents" tool. It is **three stacked orchestration layers**, and the newest one ships a *programmable* swarm runtime.

## The three layers

| Layer | What it is | Spawned by | Lifetime |
|---|---|---|---|
| **Subagent** | One-shot delegated task, returns a string | `Task`/`Agent` tool with `subagent_type` | Dies on return |
| **Teammate** | Named, persistent peer that coordinates over a shared task board | same tool + `name` / `team_name` / `mode` | Lives across turns |
| **Workflow** | Deterministic JS script that *programmatically* spawns agents | `Workflow()` tool → `agent()/parallel()/pipeline()` | Resumable across runs |

Layer 1 is the classic pattern. Layer 2 is event-driven coordination (`TeammateIdle`, `TaskCreated`, `TaskCompleted`). Layer 3 — Workflows — is the genuinely new category: orchestration as **code, not prompts**, with enforced determinism so a run can be paused and resumed with cached agent results.

## Files

| File | What's in it |
|---|---|
| [architecture.md](architecture.md) | The 3-layer model with diagrams; the agent-definition format, folder/source precedence, the tool-granting rule, context modes (fresh vs fork), isolation |
| [workflows-runtime.md](workflows-runtime.md) | The Workflow scripting layer — `agent()/parallel()/pipeline()/phase()/budget`, the determinism contract, resume-by-caching, concurrency caps, a verbatim built-in workflow |
| [hooks-and-guardrails.md](hooks-and-guardrails.md) | The 14 hook events, 3 hook types (Command / Prompt / Agent), the hook JSON I/O contract, and the full guardrail surface |

## How it maps to this repo

| Claude Code feature | Pattern in this repo |
|---|---|
| `Task` tool, N spawns in one turn, parent reduces | [flat-map-reduce](../../patterns/flat-map-reduce.md) |
| `isolation: "worktree"` per writing subagent | [best-of-n-worktree](../../patterns/best-of-n-worktree.md) |
| `Explore` read-only subagent, fresh context | [research-scout-swarm](../../patterns/research-scout-swarm.md) |
| `pr-review-toolkit` agents (security / test / type / comment) | [reviewer-swarm](../../patterns/reviewer-swarm.md) |
| Teammate + coordinator over the shared task board | [tiered-hierarchical-swarm](../../patterns/tiered-hierarchical-swarm.md) |
| `run_in_background: true` + completion notification | [background-hybrid-swarm](../../patterns/background-hybrid-swarm.md) |
| `Workflow()` deterministic resumable orchestration script | **NEW** — see [workflows-runtime.md](workflows-runtime.md) |
| Agent hook (a hook that itself spawns an agent to gate a tool call) | [verifier-loop](../../patterns/verifier-loop.md) as a hook |

## What's novel here (vs the other case studies)

| Axis | Claude Code |
|---|---|
| **Orchestration as code** | Workflows are sandboxed JS scripts, not prompt templates — the only case study where the swarm topology is a versionable artifact in `.claude/workflows/` |
| **Determinism contract** | `Date.now()` / `Math.random()` are *removed* from the workflow sandbox so runs are byte-reproducible → resume works by caching `agent()` results keyed on `(prompt, opts)` |
| **Tool granting** | Allowlist-by-subtraction: declare `tools:` → you get exactly that list + a fixed baseline; declare nothing → `["*"]` (inherit all). `disallowedTools` subtracts on top |
| **Agent-as-hook** | A hook can be of `type: "agent"` — the guardrail is itself a spawned agent with tools (verifier-loop collapsed into the hook layer) |
| **Three context modes** | Fresh (`subagent_type` set, zero context) vs fork (`subagent_type` omitted, inherits full conversation) — Claude Code *does* have fork, contrary to older notes |
| **Plugin-agent demotion** | Plugin-supplied agents have `permissionMode` / `hooks` / `mcpServers` silently ignored — only `.claude/agents/` files get that control |

## How it differs from Cursor Grind

| Axis | Cursor Grind | Claude Code |
|---|---|---|
| Coordination bus | Git (commits + rebase-retry push) | Shared in-memory **task board** (`TaskCreate/Get/List/Update`) with a `blockedBy` dependency graph |
| Write isolation | Always — one git clone per worker | Opt-in — `isolation: "worktree"` per spawn; default shares parent `cwd` |
| Swarm definition | Compiled-in registry + plugin manifests | One Markdown file per agent (`.claude/agents/*.md`) **or** a JS workflow script |
| Stall handling | `continuationPolicy { idleThreshold, maxLoops }` | `TeammateIdle` hook event + per-agent `maxTurns` / `maxBudgetUsd` |
| Recursion | Coordinator-only (`preserveTaskTool`) | Teammates cannot spawn teammates; workflows capped at 1000 lifetime agents |

## Reproducibility

```sh
# The binary (native Mach-O, JSC-linked — NOT an npm cli.js anymore)
BIN=~/.local/share/claude/versions/2.1.148
file "$BIN"                              # Mach-O 64-bit executable arm64

# App JS lives as plain cstrings. Locate the app-code region:
python3 - "$BIN" <<'EOF'
import sys; d=open(sys.argv[1],'rb').read()
print('TodoWrite @', d.find(b'TodoWrite'))   # app region starts ~here
EOF

# Dump strings once, then grep the cache (see RE rules):
strings -a "$BIN" > /tmp/cc.str

# Recover the key facts:
grep -aF 'Launch a new agent'   /tmp/cc.str   # Task tool description
grep -aF 'Hook JSON Output'     /tmp/cc.str   # hook I/O contract
grep -aF "phase('Plan')"        /tmp/cc.str   # a built-in workflow script
grep -aF 'agent() calls are capped' /tmp/cc.str   # concurrency caps
grep -aF '.claude/agents'       /tmp/cc.str   # agent source precedence
```

Minified symbol names (`sG`, `Vi`, `ee`, `T9`, …) are stable within a release but shift between releases. All quoted English strings are stable. Version-diff workspace: `~/Developer/cc-re-2148/` (binaries 2.1.144 → 2.1.148).
