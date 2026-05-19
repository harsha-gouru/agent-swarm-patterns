# Antigravity Agent Architecture

## Shape

```text
                                  USER
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  Antigravity (main)  │   ls.str:48394
                         │  Google Deepmind     │
                         │  "Advanced Agentic   │
                         │  Coding"             │
                         └──┬─────────┬─────────┘
                            │         │
                  delegates │         │ delegates
                            ▼         ▼
            ┌────────────────────┐  ┌─────────────────────┐
            │  Planning Mode     │  │  Browser Subagent   │
            │  ls.str:60205      │  │  own model, own     │
            │  → impl_plan.md    │  │  browser_* tools    │
            │  → user approval*  │  │  talks back via     │
            │  → task.md         │  │  send_message       │
            │  → walkthrough.md  │  └─────────────────────┘
            └────────────────────┘
              *gated on IsAutonomousEvalMode template flag
                            │
                            ▼
            ┌─────────────────────────────────────────┐
            │  Teamwork sub-agent system              │
            │  (role-composed; see role-composition)  │
            └─────────────────────────────────────────┘
                            │
                            ▼
                ┌─────────────────────────┐
                │  Sentinel (archetype 1) │  ls.str:90628
                │  user_liaison +         │
                │  sentinel_reporter +    │
                │  dispatcher             │
                │  • owns ORIGINAL_REQUEST.md
                │  • cron */8 progress
                │  • cron */10 liveness
                └────────────┬────────────┘
                             ▼
                ┌─────────────────────────┐
                │  Orchestrator           │  (own prompt, not in
                │  (sub_orch archetype)   │   the 6 leaf combos)
                └──┬──────┬──────┬────────┘
        ┌─────────┘      │      └────────┐
        ▼                ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Worker   │    │ Reviewer │    │Challenger│
  │ #3       │    │ #4 / #5  │    │ #2       │
  │ impl+qa+ │    │ reviewer+│    │ critic+  │
  │ specialist│   │ critic   │    │ specialist│
  └────┬─────┘    │ [+spec]  │    └──────────┘
       │          └──────────┘
       │   parallel _1, _2, _3 ...
       ▼
   handoff.md
       │
       ▼
  Orchestrator says "DONE" ─────┐
                                │
                                ▼
                  ┌─────────────────────────┐
                  │  Auditor (archetype 6)  │  MANDATORY
                  │  critic + specialist +  │
                  │  auditor                │
                  │  • re-runs tests        │
                  │  • forensic verify      │
                  └────────────┬────────────┘
                               │
                       VICTORY CONFIRMED ──→ Sentinel tells user
                       VICTORY REJECTED ──→ resume team with audit

  After conversation ends ───►  Background KI Extractor  (ls.str:51426)
                                  • no user interaction
                                  • jailed to KI dirs
                                  • spawns KI Specialist (ls.str:56850)
                                    if consolidation needed

  Internal Googler only ─────►  Critique / Gerrit specialist  (ls.str:69697)
                                  • read-only; defers edits to main agent
                                  • knows CL flow, Louhi CI, Sponge logs
```

## Verified persona line refs (Antigravity IDE 2.0.1)

| Persona | `language_server.str` line | Verbatim opening |
|---|---:|---|
| Main agent | 48394 | "You are Antigravity, a powerful agentic AI coding assistant designed by the Google Deepmind team working on Advanced Agentic Coding." |
| Main agent (alt build) | 46778 | "You are Antigravity Agent, a powerful agentic AI coding assistant designed by the Google engineering team." |
| Planning Mode | 60205 | "You are in Planning Mode. Exercise judgement on whether a user's request warrants a plan before taking action." |
| Background KI | 51426 | "You are operating as a **background agent** that runs automatically after conversations to extract and preserve knowledge." |
| KI Specialist | 56850 | "You are a Knowledge Items (KI) specialist responsible for managing the knowledge base." |
| Critique/Gerrit | 69697 | "You are an AI assistant knowledgeable about Google's code review tools: Critique and Gerrit." |
| Teamwork #1 Sentinel | 90628 | "You are a Teamwork agent with roles: user_liaison, sentinel_reporter, dispatcher." |
| Teamwork #2 Challenger | 94533 | "You are a Teamwork agent with roles: critic, specialist." |
| Teamwork #3 Worker | 100505 | "You are a Teamwork agent with roles: implementer, qa, specialist." |
| Teamwork #4 Reviewer (lite) | 100791 | "You are a Teamwork agent with roles: reviewer, critic." |
| Teamwork #5 Reviewer (full) | 101924 | "You are a Teamwork agent with roles: reviewer, critic, specialist." |
| Teamwork #6 Auditor | 110203 | "You are a Teamwork agent with roles: critic, specialist, auditor." |

## Top-level persona contracts

| Persona | Trigger | Cannot do | Outputs |
|---|---|---|---|
| Main agent | User chat turn | — | Tool calls + message stream |
| Planning Mode | Major arch change / extensive research / ambiguity / significant deviation | Run modifying commands during research phase | `implementation_plan.md` (with `request_feedback=true`), then `task.md` + `walkthrough.md` |
| Browser Subagent | Main agent calls `browser_subagent` | Drive main agent's tools | `send_message` to parent |
| Background KI | Post-conversation, automatic | Talk to user, use main-agent tools, edit outside KI dirs | KI files only |
| KI Specialist | Spawned by Background KI | Same constraints | Merged KIs, `delete_knowledge` calls |
| Critique/Gerrit | Internal Googler workflow | Modify CLs via Cider Webapp | Guidance; defers edits to main agent |

## Filesystem contract (every Teamwork agent)

```text
project_root/
├── .agents/                          ← metadata only; never source/tests/data
│   ├── ORIGINAL_REQUEST.md           ← Sentinel-owned; verbatim user msgs, append-only
│   ├── orchestrator/
│   │   ├── plan.md
│   │   ├── progress.md               ← liveness heartbeat (≥ every 5 min during long runs)
│   │   ├── context.md
│   │   ├── BRIEFING.md               ← ≤ ~100 lines; "who am I, what do I know"
│   │   ├── BRIEFING_ARCHIVE.md       ← spillover; never archive append-only sections
│   │   └── original_prompt.md        ← UTC-timestamped append log of inbound messages
│   ├── explorer_1/                   ← _<N> = parallel instance number
│   │   ├── analysis.md
│   │   └── handoff.md
│   ├── implementer_1_gen2/           ← _gen<N> = successor generation
│   │   ├── changes.md
│   │   └── handoff.md
│   ├── reviewer_lexer/               ← _<milestone>
│   │   └── handoff.md
│   ├── challenger_parser/
│   └── sub_orch_phase2/
└── src/                              ← code lives here, NOT in .agents/
```

Directory pattern: `.agents/<type>_<milestone>[_<N>][_gen<N>]/`
- `<type>` ∈ `{teamwork_preview_explorer, worker, reviewer, challenger, sub_orch, …}`
- Each agent owns ONE directory.
- Write own folder; read any folder; never write another agent's folder.
- Reference other agents' files by path — don't copy content.
- Source/tests/data in `.agents/` is a layout-compliance violation reviewers must flag.

## Communication contract

```text
| Use FILE for                    | Use MESSAGE for           |
|---------------------------------|---------------------------|
| Reports, handoffs, analysis     | "I'm done, check report"  |
| Code changes, proposals         | "I'm stuck, need help"    |
| Progress updates (progress.md)  | Clarification questions   |
```

Message format:

```text
Context: [what you are working on]
Content: [what you want to communicate]
Action:  [what you expect the recipient to do]
```

## The 5-component handoff report

```text
handoff.md:
  1. Observation       — file paths, line numbers, verbatim errors, quote directly
  2. Logic Chain       — step-by-step reasoning, each step refs an observation
  3. Caveats           — areas not investigated, assumptions, alt interpretations
  4. Conclusion        — supported by logic chain, actionable, scoped
  5. Verification Method — specific commands (pytest, cargo test), files, invalidation
```

Three handoff types:

| Type | When | Extra |
|---|---|---|
| Hard | Task complete | All five sections populated |
| Soft | Task transferred (context overflow, replacement) | Add "Remaining Work" with concrete next steps |
| Partial | Agent stuck or replaced mid-task | Fill what you can, explain where/why stuck |

## Iron rule

> Once a sub-agent has delivered `handoff.md` or a completion report, it is **permanently retired**. Always spawn fresh agents for new work.

No agent state survives across tasks. Successor generations get a new `_gen<N>` directory; the original is dead.

## Guardrails worth borrowing

- BRIEFING.md keeps two append-only sections (`## My Identity`, `## Key Constraints`) — survives context truncation.
- `progress.md` is the liveness heartbeat, separate from any user-visible task tracker.
- Verification scales by impact: High (production / security) → verify everything. Low (docs / style) → spot-check.
- `Layout Compliance` reviewers actively flag source/tests/data leaking into `.agents/`.

## Failure modes observed in the design

- Sub-agents collapse into shell commands when leaves look trivial → repo's tiered-hierarchical-swarm pattern warns about this.
- Context loss across levels hides evidence → 5-component handoff report exists exactly to defeat this.
- Successor confusion → `_gen<N>` suffix + Sentinel's "completed handoff is NOT death" rule.
