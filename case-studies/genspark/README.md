# Case Study: Genspark (Super Agent / Mixture-of-Agents)

> Sources, all public:
> - [genspark.ai/blog/genspark-less-control-more-tools](https://www.genspark.ai/blog/genspark-less-control-more-tools) — "Less Control, More Tools"
> - [anthropic.com/customers/genspark](https://anthropic.com/customers/genspark) — Anthropic customer story
> - [openai.com/index/genspark](https://openai.com/index/genspark/) — OpenAI customer story
> - [venturebeat.com/.../gensparks-super-agent](https://venturebeat.com/ai/gensparks-super-agent-ups-the-ante-in-the-general-ai-agent-race) — VentureBeat coverage

This is a **stub** — a short, source-cited entry, not a full RE write-up.

## What it is

Genspark builds **Super Agent**, a general AI agent. It is the **dissenting data
point** in this repo: where Cursor / Warp / Cognition converge on structured,
isolated, single-threaded-write topologies, Genspark argues for the opposite —
emergent orchestration with minimal structure.

## The orchestration thesis

Their stated principle: **"less control, more tools."** Overly structured
workflows limit creativity; freedom to choose and chain tools unlocks more.

Core agent = "an LLM in a while loop — plan, execute, observe, and crucially,
**backtrack**." Two design areas they call out: *model orchestration* and an
*improvement loop*.

## Architecture

- **Mixture-of-Agents**: ~9 different-sized LLMs + 80+ tools + 10+ proprietary
  datasets. Explicitly multi-model — "the future is multi-model, not one model
  to rule them all."
- **Per-task routing**: each query is routed to the best-suited model and tools
  (model routing + retrieval-based tool selection). Creative → OpenAI,
  complex reasoning → Claude, visual → Gemini.
- **Claude as master coordinator**: drives planning, reasoning, and the overall
  agent flow that selects models and tools.
- **Cross-verification**: specialized models verify each other's outputs to
  reduce errors and hallucinations.
- **Tool composability** is the explicit design driver — clean APIs, "LEGO
  blocks"; 80 tools yield tens of thousands of chains.
- **Improvement loop**: LLM-as-judge scores whole agent sessions, then
  **attributes the reward to each individual step**; this step-level signal is
  internalized via reinforcement learning + "prompt playbooks," so the system
  improves from live user traffic.

## How it maps to this repo

| Genspark construct | Pattern in this repo |
|---|---|
| Per-task model + tool routing across 9 LLMs | [race-first-success](../../patterns/race-first-success.md) / routing — no fixed tree |
| Cross-verification between specialized models | [reviewer-swarm](../../patterns/reviewer-swarm.md) (peer check) |
| Plan / execute / observe / backtrack loop | [verifier-loop](../../patterns/verifier-loop.md) |
| LLM-judge → step-level reward attribution → RL | [runtime-primitives](../../patterns/runtime-primitives.md) (feedback layer) |
| "Less control, more tools" — emergent composition | **counterpoint** — see below |

## Why it matters

Genspark is the deliberate counterexample. It has no enforced agent tree, no
worktree-style isolation, and no single-threaded-write rule — it leans on
routing + cross-verification + a step-level reward loop instead. Read it
alongside [Cognition](../cognition/README.md): the two are explicit opposites on
whether swarms should be *structured* or *emergent*, which makes the pair the
clearest framing of the central design tension in this repo.
