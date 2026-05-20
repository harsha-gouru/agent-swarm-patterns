# Index: Agent Swarm Techniques

## Mental model

```text
parent agent
  emits N tasks in one turn
    -> bash task | background process | subagent | nested subtree
  waits on task IDs
  verifies truth in parent context
  applies/merges only after verification
```

## Pattern catalog

| Pattern | Use when | File |
|---|---|---|
| Flat map-reduce | Independent items can be analyzed in parallel | [flat-map-reduce.md](patterns/flat-map-reduce.md) |
| Best-of-N worktree | Multiple implementations should compete safely | [best-of-n-worktree.md](patterns/best-of-n-worktree.md) |
| Research scout | Different evidence sources need separate context | [research-scout-swarm.md](patterns/research-scout-swarm.md) |
| Reviewer swarm | One patch needs security/perf/test/design review | [reviewer-swarm.md](patterns/reviewer-swarm.md) |
| Tiered hierarchy | Many subtasks need supervisors and local reduction | [tiered-hierarchical-swarm.md](patterns/tiered-hierarchical-swarm.md) |
| Race / first-success | Latency matters more than comparing all results | [race-first-success.md](patterns/race-first-success.md) |
| Pipeline handoff | Work moves through research -> implementation -> review | [pipeline-handoff-swarm.md](patterns/pipeline-handoff-swarm.md) |
| Background hybrid | Agents coordinate with long-running shell tasks | [background-hybrid-swarm.md](patterns/background-hybrid-swarm.md) |
| Verifier loop | Parent needs an independent second pass | [verifier-loop.md](patterns/verifier-loop.md) |
| Runtime primitives | Design notes for building a swarm runtime | [runtime-primitives.md](patterns/runtime-primitives.md) |

## Case studies

| Subject | Patterns it composes | File |
|---|---|---|
| Google Antigravity IDE | tiered-hierarchical, flat-map-reduce, reviewer-swarm, verifier-loop, pipeline-handoff, background-hybrid + new sentinel-cron supervisor | [case-studies/google-antigravity](case-studies/google-antigravity/README.md) |
| Cursor "Grind" | tiered-hierarchical, flat-map-reduce, best-of-n-worktree, background-hybrid + new git-as-bus coordination | [case-studies/cursor](case-studies/cursor/README.md) |
| Warp (Multi-Harness Orchestration) | flat-map-reduce, pipeline-handoff, background-hybrid, tiered-hierarchical + new multi-harness dispatch (Oz / ClaudeCode / OpenCode / Gemini / Codex) | [case-studies/warp](case-studies/warp/README.md) |
| Google Co-Scientist | tiered-hierarchical, research-scout, reviewer-swarm, best-of-n, verifier-loop, pipeline-handoff + new Elo-tournament-as-fitness + in-context meta-learning | [case-studies/co-scientist](case-studies/co-scientist/README.md) |
| Cognition (Managed Devins) | tiered-hierarchical, flat-map-reduce, best-of-n-worktree, verifier-loop — structured "map-reduce-and-manage", single-threaded writes | [case-studies/cognition](case-studies/cognition/README.md) |
| Genspark (Mixture-of-Agents) | per-task routing, reviewer-swarm, verifier-loop — emergent "less control, more tools" (the structured-swarm counterpoint) | [case-studies/genspark](case-studies/genspark/README.md) |

## Comparisons

| Comparison | Covers | File |
|---|---|---|
| Agent control mechanics | Claude Code / Cursor / Grok CLI / Codex — file-conflict isolation, context passing, mid-run stop/check/inspect, config surface, guardrails | [comparison-agent-control.md](case-studies/comparison-agent-control.md) |

## Safety rules

1. Use absolute paths in child prompts.
2. Use read-only capability for exploration.
3. Use worktree isolation for writes.
4. Parent owns final verification.
5. Never trust child summaries without checking artifacts.
6. Persist per-child logs: summary, chat history, events, updates, terminal output.
