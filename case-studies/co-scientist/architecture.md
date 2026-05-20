# Co-Scientist Architecture

Agent-by-agent breakdown of Google's Co-Scientist, sourced from the Methods
section of Gottweis *et al.*, *Nature* (2026), DOI
[10.1038/s41586-026-10644-y](https://doi.org/10.1038/s41586-026-10644-y).

## Full flow

```
 ┌───────────┐
 │ Scientist │  writes a research goal in natural language: question,
 └─────┬─────┘  known constraints, desired output attributes, prior PDFs.
       │        Can also: add own hypotheses, review, steer mid-run.
       ▼
 ┌──────────────────────┐
 │ Research plan config │  Co-Scientist parses the goal into a config:
 │                      │  default criteria = alignment, plausibility,
 │                      │  novelty, testability, safety. Customizable.
 └──────────┬───────────┘
            ▼
 ┌──────────────────────┐         ┌────────────────────────┐
 │  SUPERVISOR agent    │◄───────►│  Context memory         │
 │  - builds task queue │  state  │  - agent + system state │
 │  - assigns agents to │  every  │  - tournament progress  │
 │    worker processes  │  cycle  │  - enables restart on   │
 │  - allocates compute │         │    component failure    │
 │  - computes summary  │         └────────────────────────┘
 │    statistics        │
 └──────────┬───────────┘
            │  async, continuous worker queue
            ▼
   ╔════════════════════ TOURNAMENT LOOP ════════════════════╗
   ║                                                          ║
   ║  GENERATION ──► hypotheses ──► REFLECTION ──► reviewed    ║
   ║   strategies:                   review types:  hyps       ║
   ║   - literature web search       - initial (no tools)      ║
   ║   - simulated scientific debate - full (web search)       ║
   ║   - assumption decomposition    - deep verification       ║
   ║   - research expansion          - observation             ║
   ║                                 - simulation              ║
   ║                                 - recurrent/tournament    ║
   ║         │                             │                   ║
   ║         ▼                             ▼                   ║
   ║  PROXIMITY ──► similarity graph ──► RANKING                ║
   ║   clusters / dedups                Elo tournament:         ║
   ║                                    - init Elo 1200         ║
   ║                                    - top hyps: multi-turn  ║
   ║                                      pairwise debate       ║
   ║                                    - low hyps: single-turn ║
   ║                                    - similar hyps paired   ║
   ║                                      (via Proximity graph) ║
   ║         │                             │                   ║
   ║         ▼                             ▼                   ║
   ║  EVOLUTION ◄── top-ranked hyps ── tournament state         ║
   ║   refine strategies (emit NEW hyps, never mutate in place):║
   ║   - enhancement through grounding                          ║
   ║   - coherence / feasibility fixes                          ║
   ║   - inspiration from existing hyps                         ║
   ║   - combination of top hyps                                ║
   ║   - simplification                                         ║
   ║   - out-of-box thinking                                    ║
   ║         │                                                  ║
   ║         ▼                                                  ║
   ║  new hypotheses re-enter the tournament ──────────┐        ║
   ║                                                    │        ║
   ║  META-REVIEW ◄── all reviews + debate patterns ────┘        ║
   ║   - synthesizes recurring issues into a critique           ║
   ║   - critique APPENDED to every agent's prompt next cycle   ║
   ║   - => feedback propagation without fine-tuning/RL         ║
   ║   - also: research overview + suggested expert contacts    ║
   ╚════════════════════════════╤═════════════════════════════╝
                                │ terminal state reached
                                ▼
                    ┌────────────────────────┐
                    │ Research overview +    │──► Scientist reviews,
                    │ ranked hypotheses      │    selects for wet lab
                    └────────────────────────┘
```

## The agents in detail

### Supervisor
Not one of the "six specialized agents" — it is the orchestrator. Parses the
research plan config, builds and runs an **asynchronous task queue**, assigns
specialized agents to worker processes, and allocates compute. Periodically
computes summary statistics (hypotheses generated, hypotheses awaiting review,
tournament progress, effectiveness of each generation method) and writes them to
context memory. Those statistics drive the next round's agent weighting and the
decision of whether a terminal state is reached.

### Generation
Produces the initial hypothesis pool. Four strategies, each a prompt:
- **Literature exploration** — iterative web search, reads articles, builds a
  fact base, then proposes novel directions.
- **Simulated scientific debate** — self-play multi-turn debate among simulated
  experts, refined hypothesis emitted at the end.
- **Iterative assumption identification** — finds testable intermediate
  assumptions via conditional reasoning hops, aggregates them into hypotheses.
- **Research expansion** — uses prior hypotheses + meta-review feedback to find
  unexplored regions of the hypothesis space.

### Reflection
The peer reviewer. Six review modes:
- **Initial review** — fast, no tools, discards obviously flawed/non-novel ideas.
- **Full review** — web search for grounding; scrutinizes assumptions, judges
  novelty against literature.
- **Deep verification** — decomposes a hypothesis into assumptions →
  sub-assumptions, evaluates each independently to find invalidating elements.
- **Observation review** — can the hypothesis explain long-tail prior results?
- **Simulation review** — mentally simulates the hypothesis / proposed
  experiment step-by-step to surface failure modes.
- **Recurrent/tournament review** — adapts reviews based on tournament results.

> Ablation result: Reflection *without* search rated a known non-novel idea
> 6.14/10 novelty; *with* search, 2.38/10. The external tool is what makes the
> critic trustworthy.

### Ranking
Runs the Elo tournament that is the system's fitness function. New hypotheses
start at **Elo 1200**. Top-ranked hypotheses meet in **multi-turn scientific
debates**; lower-ranked ones get cheaper single-turn pairwise comparisons.
Match-making is prioritized by (1) similarity — similar hypotheses (from the
Proximity graph) are compared, and (2) recency + current rank.

### Proximity
Builds a similarity graph over hypotheses. Does not generate anything — it feeds
the Ranking agent's match-making and lets the system cluster/dedup and surface a
diverse set to the scientist.

### Evolution
Continuously refines top-ranked hypotheses. Strategies: grounding enhancement,
coherence/feasibility fixes, inspiration, combination of multiple top
hypotheses, simplification, out-of-box divergence. **Key invariant:** it always
emits *new* hypotheses that must compete in the tournament — it never modifies or
replaces existing ones, so a bad "improvement" cannot overwrite a good idea.

### Meta-review
Closes the self-improvement loop. Synthesizes all reviews + tournament-debate
patterns into a meta-review critique. That critique is **appended to the prompts
of all agents** in the next iteration — feedback propagation with no
back-propagation, fine-tuning, or RL. Also generates the final research overview
(can be formatted to e.g. NIH Specific Aims via constrained decoding) and
suggests qualified expert contacts.

## Why the design matters for swarm engineering

1. **Fitness without a reward model.** Pairwise debate + Elo gives a relative
   quality ordering using only the base LLM. No training signal needed.
2. **In-context meta-learning.** Long context turns "append the lesson to the
   prompt" into a viable substitute for weight updates across iterations.
3. **Compute is the dial.** The Supervisor's async queue means quality scales
   with wall-clock/compute spent, not with model size.
4. **Mutation isolation.** Evolution emitting-only-new + tournament re-entry is
   the same safety property as best-of-N worktrees: improvements compete, they
   don't clobber.
5. **The critic needs a tool.** Every reflection quality result in the paper
   hinges on web search. A hypothesis swarm is only as honest as its verifier.

## Reproducibility

The system code is not released. The paper provides:
- Multi-agent orchestration + tournament pseudocode — Supplementary Note 8.
- Verbatim agent prompts — Supplementary Note 9.
- Worked examples (ALS recurring example) — Supplementary Note 10.

Base model: Gemini 2.0 (Pro / Flash) — architecture stated as model-agnostic.
