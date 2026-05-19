# Role Composition

Antigravity's Teamwork sub-agents are not classes — they are **lists of named roles**. Each role contributes a prompt fragment, glued onto a shared shell. Six concrete role combinations ship in the binary.

## The 10-role library

Verbatim role definitions from the binary:

| Role | One-line definition (verbatim from `language_server.str`) |
|---|---|
| **user_liaison** | User context manager: tracks user requests, maintains briefing about user state. Does not handle human I/O directly — that is packaged in the human_reporter and human_communicator roles. |
| **sentinel_reporter** | Ultra-light reporting for sentinel agents. Report progress and status to humans via concise bullet-point summaries. No structured presentations, no feedback collection. |
| **dispatcher** | Minimal dispatch capability: grants the ability to spawn and dispatch tasks to subagents. Does not include orchestration strategy, team management, or subagent lifecycle management. |
| **critic** | Adversarial challenge: stress-test assumptions, find failure modes, propose counter-examples. |
| **specialist** | External domain expert: loads and follows methodology from user-specified Antigravity skill paths. Provides specialized capability without requiring new Teamwork skill definitions. |
| **implementer** | Code modification: implement changes and verify correctness. |
| **qa** | Quality assurance: fix defects only, no feature work. |
| **reviewer** | Objective review: assess work quality, verify claims, issue verdict. |
| **auditor** | Integrity auditor: independently verifies work products and project completion claims through forensic checks and independent test execution. |
| **human_reporter** / **human_communicator** | Referenced in `user_liaison`'s definition; no Teamwork combo in the binary bundles them. The main Antigravity agent appears to be the human I/O path. |

## The shared shell (every Teamwork agent gets these sections)

Regardless of role mix, the following sections appear verbatim in every Teamwork system prompt:

1. **Communication Guideline** — files for content, messages for coordination
2. **Handoff Protocol** — 5-component `handoff.md` (Observation / Logic Chain / Caveats / Conclusion / Verification Method)
3. **Verification** — scaled by impact; includes Layout Compliance check
4. **File Workspace Convention** — `.agents/<type>_<milestone>[_<N>][_gen<N>]/`; write own, read any, never write another's
5. **Workflow Fault Tolerance** — `progress.md` as liveness heartbeat
6. **Situational Awareness / BRIEFING.md** — append-only `## My Identity` + `## Key Constraints` survive context truncation

Total shared-shell length: ~120 lines. Role fragments are appended after.

## The six archetypes (verified role combos)

| # | Inferred name | Roles | `language_server.str` line | Distinguishing prompt fragment appended |
|---|---|---|---:|---|
| 1 | **Sentinel** | user_liaison + sentinel_reporter + dispatcher | 90628 | *User Request Recording* (`ORIGINAL_REQUEST.md`, append-only verbatim), *Subagent Management* (Iron Rule), *Sentinel Monitoring* (two crons), *Human Reporting*, *Workflow Protocol* |
| 2 | **Challenger** | critic + specialist | 94533 | *Adversarial Review* — 4 dims: assumption stress-testing, edge case mining, dependency risk, logical counterarguments |
| 3 | **Worker** | implementer + qa + specialist | 100505 | *Code Modification* — 5-step: validate → prep → execute → verify → document; *Prohibited Actions* |
| 4 | **Reviewer (lite)** | reviewer + critic | 100791 | *Quality Review* — 4 dims: correctness, logical completeness, quality, risk (incl. dependency-coverage escalation) |
| 5 | **Reviewer (full)** | reviewer + critic + specialist | 101924 | Same as #4 + specialist methodology loading |
| 6 | **Auditor** | critic + specialist + auditor | 110203 | *Adversarial Review* + *Forensic Audit* (independently re-runs tests, verifies completion claims) |

Archetype names "challenger", "worker", "sub_orch", "reviewer", "teamwork_preview_explorer" are listed verbatim in the Sentinel's `.agents/<type>_…` directory convention — so the inferred names match Google's internal taxonomy.

## Role → fragment map

```text
user_liaison        → User Request Recording (ORIGINAL_REQUEST.md)
sentinel_reporter   → Human Reporting (concise bullets only)
dispatcher          → Subagent Management (Iron Rule, no reuse after handoff)
                      Sentinel Monitoring (cron protocol)

critic              → Adversarial Review section
                       1. Assumption Stress-Testing
                       2. Edge Case Mining
                       3. Dependency Risk
                       4. Logical Counterarguments

reviewer            → Quality Review section
                       1. Correctness
                       2. Logical Completeness
                       3. Quality
                       4. Risk Assessment (with dependency-coverage escalation)

implementer + qa    → Code Modification section
                       Minimal change. 5-step workflow.
                       Tools: replace_file_content, multi_replace_file_content
                       Prohibited Actions list

specialist          → Skill-loading hook
                      Reads .agent/skills/ paths supplied in dispatch message

auditor             → Forensic Audit (independent test re-execution)
                      Issues VICTORY CONFIRMED / VICTORY REJECTED verdict
```

## Adversarial Review vs Quality Review

These are **not** the same role.

**Critic — "how could this fail?"**

```text
- List every assumption (explicit + implicit)
- For each, construct a scenario where it fails
- Assess blast radius
- Complexity & efficiency: construct worst-case inputs (TLE/MLE)
- Edge cases: empty inputs, max sizes, concurrent access, off-by-one,
  type mismatches, encoding
- Dependency risk: unavailable / slow / unexpected data; tight/loose versions
- Logical counterarguments: same evidence → different conclusion?
  correlation vs causation?
```

**Reviewer — "what was done and what was missed?"**

```text
- Correctness: implements requirements? regressions? boundary cases?
- Logical completeness: reasoning chain complete? leaps? supported?
- Quality: style, test coverage, docs
- Risk assessment: impact scope, perf, security
- Dependency coverage: did upstream investigation examine all relevant
  dependencies and call sites? If unexplored risk → CALL FOR FURTHER
  INVESTIGATION before approving (request original investigator to extend,
  OR recommend orchestrator spawn additional investigation)

Workflow:
  1. Independent reading → preliminary judgment
  2. Verify key claims
  3. Assess exploration coverage
  4. Ask questions about uncertain areas (don't outright reject)
```

Archetypes #4 (reviewer+critic) and #5 (reviewer+critic+specialist) carry both lenses; archetype #2 (critic+specialist) is critic-only; archetype #6 (critic+specialist+auditor) is critic + forensic test re-run.

## BRIEFING.md template (per archetype)

The mutable sections vary by role mix; the append-only sections are universal.

**Universal (append-only)**:

```markdown
## My Identity
- Archetype: <archetype name>
- Roles: <role list>
- Working directory: <absolute path to your .agents/ directory>
- Original parent: <orchestrator conversation ID>
- Milestone: <milestone name>
- Instance: <N of M>

## Key Constraints
- <constraints from dispatch message>
```

**Worker variant** adds:

```markdown
## Task Summary
- What to build: <description>
- Success criteria: <criteria>
- Interface contracts: <path to PROJECT.md / SCOPE.md>
- Code layout: <path to PROJECT.md → Code Layout>
```

**Reviewer / Critic variant** adds:

```markdown
## Review Scope
- Files to review: <file paths from dispatch>
- Interface contracts: <path to PROJECT.md / SCOPE.md>
- Review criteria: correctness, style, conformance
```

**Sentinel variant** adds:

```markdown
## User Context
- Last user request: <summary>
- Pending clarifications: <none>
- Delivered results: <list>

## Project Status
- Phase: <not started / in progress / victory claimed / auditing / complete>

## Victory Audit Status
- Triggered: <yes/no>
- Verdict: <pending / VICTORY CONFIRMED / VICTORY REJECTED>
- Retry count: <0>
```

## Borrowing the pattern

To replicate this in your own swarm:

1. Define a small role library (5-10 roles, each with a one-line definition).
2. Write one shared shell prompt (handoff format, filesystem contract, communication rules).
3. Define each agent type as `<shared shell> + <role fragments concatenated>`.
4. Make BRIEFING.md the persistent memory with two append-only sections that resist self-rewriting.
5. Enforce the iron rule: no reuse after `handoff.md`.

Composition beats inheritance here — adding a new agent type is just picking a different role tuple, no code change.
