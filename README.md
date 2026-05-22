# Agent Swarm Patterns

A compact field guide to practical agent-swarm orchestration patterns: parent/child sessions, task IDs, worktree isolation, verifier loops, map-reduce, best-of-N, and background command swarms.

Start here: [index.md](index.md)

## Patterns

- [Flat map-reduce swarm](patterns/flat-map-reduce.md)
- [Best-of-N worktree swarm](patterns/best-of-n-worktree.md)
- [Research scout swarm](patterns/research-scout-swarm.md)
- [Reviewer swarm](patterns/reviewer-swarm.md)
- [Tiered hierarchical swarm](patterns/tiered-hierarchical-swarm.md)
- [Race / first-success swarm](patterns/race-first-success.md)
- [Pipeline handoff swarm](patterns/pipeline-handoff-swarm.md)
- [Background hybrid swarm](patterns/background-hybrid-swarm.md)
- [Verifier loop](patterns/verifier-loop.md)
- [Security investigation swarm](patterns/security-investigation-swarm.md)
- [Runtime primitives](patterns/runtime-primitives.md)

## Case studies

- [Claude Code](case-studies/claude-code/README.md) — the reference orchestrator, RE'd from the 2.1.148 binary: three stacked layers (one-shot subagents, persistent teammates over a shared task board, and a deterministic resumable JS **workflow runtime**), the tool-granting allowlist rule, 14 hook events incl. agent-as-hook, and the full guardrail surface
- [Grok CLI — Goal Mode](case-studies/grok-goal-mode/README.md) — new and undocumented in Grok 0.1.214: a productized verifier-gated swarm (orchestrator → persistent worker → mandatory independent verifier), a typed deliverable state machine via `update_goal`, a three-verdict system (ACHIEVED / GAP / BLOCKED), and the `subagent_id` chain rule for resume-based swarms
- [Google Antigravity](case-studies/google-antigravity/README.md) — how Google Deepmind's agentic IDE composes these patterns in a shipping product (six role-composed sub-agents, Sentinel cron supervisor, mandatory VICTORY AUDIT)
- [Cursor "Grind"](case-studies/cursor/README.md) — how Cursor's swarm uses git itself as the coordination bus (Orchestrator → Planner → Workers, isolated clones, rebase-retry push, "only your commit survives")
- [Warp (Multi-Harness Orchestration)](case-studies/warp/README.md) — open-source meta-orchestrator that dispatches across 5 agent harnesses (Oz / ClaudeCode / OpenCode / Gemini / Codex), with plan-attached orchestration config + auto-launch matching, `LifecycleEventType` subscription, and a 35-tool agent surface
- [Google Co-Scientist](case-studies/co-scientist/README.md) — published (Nature 2026) scientific-reasoning swarm: a Supervisor fans hypotheses to 6 specialized agents, an Elo tournament of pairwise scientific debates is the fitness function, and the Meta-review agent self-improves the system by appending learned critiques to every agent's prompt — no fine-tuning
- [Cognition (Managed Devins)](case-studies/cognition/README.md) — stub: the most rigorously argued orchestration position — "don't build unstructured multi-agents", keep writes single-threaded, ship "map-reduce-and-manage" (coordinator + children in isolated VMs)
- [Genspark (Mixture-of-Agents)](case-studies/genspark/README.md) — stub: the deliberate counterpoint — "less control, more tools", per-task routing across 9 LLMs + 80+ tools, no enforced tree or write isolation

## Comparisons

- [How agentic CLIs control subagents](case-studies/comparison-agent-control.md) — Claude Code vs Cursor vs Grok CLI vs Codex on the mechanics that actually matter: file-conflict isolation, context passing, mid-run stop/check/inspect, config surface, guardrails

## Core idea

```text
Swarm = parent creates many independent task IDs, waits, then reduces.

task_id can be:
  - shell command
  - background process
  - subagent session
  - nested subtree
```
