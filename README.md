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
- [Runtime primitives](patterns/runtime-primitives.md)

## Case studies

- [Google Antigravity](case-studies/google-antigravity/README.md) — how Google Deepmind's agentic IDE composes these patterns in a shipping product (six role-composed sub-agents, Sentinel cron supervisor, mandatory VICTORY AUDIT)
- [Cursor "Grind"](case-studies/cursor/README.md) — how Cursor's swarm uses git itself as the coordination bus (Orchestrator → Planner → Workers, isolated clones, rebase-retry push, "only your commit survives")
- [Warp (Oz / Skills)](case-studies/warp/README.md) — Anthropic-format skills shipped as Apache-2.0 plaintext, agent-driven skill optimization loop with stratified train/test split, subprocess delegation to `claude -p`

## Core idea

```text
Swarm = parent creates many independent task IDs, waits, then reduces.

task_id can be:
  - shell command
  - background process
  - subagent session
  - nested subtree
```
