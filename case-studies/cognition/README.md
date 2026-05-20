# Case Study: Cognition (Devin / Managed Devins)

> Sources, all public:
> - [cognition.ai/blog/dont-build-multi-agents](https://cognition.ai/blog/dont-build-multi-agents) — "Don't Build Multi-Agents" (Jun 2025)
> - [cognition.ai/blog/multi-agents-working](https://cognition.ai/blog/multi-agents-working) — "Multi-Agents: What's Actually Working"
> - [cognition-labs.com/blog/devin-can-now-manage-devins](https://www.cognition-labs.com/blog/devin-can-now-manage-devins) — Managed Devins
> - [cognition-labs.com/blog/devin-can-now-schedule-devins](https://www.cognition-labs.com/blog/devin-can-now-schedule-devins) — Scheduled Devins
> - [claude.com/customers/cognition](https://claude.com/customers/cognition) — Anthropic customer story

This is a **stub** — a short, source-cited entry, not a full RE write-up.

## What it is

Cognition builds **Devin**, an autonomous AI software engineer. Their orchestration
position is the most rigorously argued of any case study here, because they
published it as it evolved.

## The orchestration thesis

**"Don't Build Multi-Agents" (Jun 2025).** Reframe prompt engineering →
**context engineering**. Two principles:
1. Share context — full agent *traces*, not just messages.
2. Every action carries implicit decisions; conflicting decisions across agents
   produce fragile systems.

Conclusion at the time: parallel multi-agent collaboration in 2025 is fragile —
decision-making too dispersed, cross-agent context-passing unsolved.

**"Multi-Agents: What's Actually Working" (later).** Softened to a narrow
working class: **multiple agents contribute intelligence, but writes stay
single-threaded.** Unstructured swarms ("arbitrary networks of agents
negotiating") are "mostly a distraction." The practical shape is
**map-reduce-and-manage**: a manager splits work, children execute, the manager
synthesizes and reports.

## Managed Devins (the shipped product)

- **Main Devin = coordinator.** Scopes the work, assigns pieces, monitors
  progress, resolves conflicts, compiles results.
- **Each child = a full Devin in its own isolated VM** — own terminal, browser,
  dev environment, test runner, and session link.
- Coordination runs over an **internal MCP**.
- Parent controls: message a child mid-task, monitor per-child compute (ACU)
  consumption, **sleep or terminate** a child that is done or going off-track,
  schedule self-messages as checkpoints.
- The coordinator can read children's **full trajectories** to improve how it
  breaks down the next task.
- Rationale: "context accumulates, focus degrades" — each child gets a clean
  slate, narrow focus.
- Composes with **Scheduled Devins** — recurring runs that carry state between
  runs via the agent's own notes.

## How it maps to this repo

| Cognition construct | Pattern in this repo |
|---|---|
| Manager splits → children execute → manager synthesizes | [tiered-hierarchical-swarm](../../patterns/tiered-hierarchical-swarm.md) + [flat-map-reduce](../../patterns/flat-map-reduce.md) |
| One isolated VM per child Devin | [best-of-n-worktree](../../patterns/best-of-n-worktree.md) (isolation half) |
| Single-threaded writes; extra agents add *intelligence*, not actions | [reviewer-swarm](../../patterns/reviewer-swarm.md) |
| Coordinator reads child trajectories, sleeps/terminates strays | [verifier-loop](../../patterns/verifier-loop.md) + parent-owned control |
| Scheduled Devins carrying state between runs | [background-hybrid-swarm](../../patterns/background-hybrid-swarm.md) |

## Why it matters

Cognition independently arrived at the same convergent recipe seen in the
[Cursor](../cursor/README.md) and [Warp](../warp/README.md) case studies:
**isolate every writing child, keep writes single-threaded, give the parent
explicit control (message / sleep / terminate / inspect trajectory).** Their
explicit rejection of unstructured swarms is the strongest published argument
for the structured patterns in this repo.
