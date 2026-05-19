# Case Study: Google Antigravity

> Reverse-engineered from `Antigravity IDE.app` 2.0.1 (`com.google.antigravity-ide`) — specifically the 124 MB stripped `bin/language_server_macos_arm` agent binary. All quoted material was recovered as static string literals via `strings -a` and is reproducible.

Antigravity is Google Deepmind's agentic coding IDE (VSCode fork, codename "Jetski"). Its multi-agent design composes most of the patterns already cataloged in this repo and adds one new one (the **Sentinel cron supervisor**). This case study shows how those pieces fit in a shipping product.

## Files

| File | What's in it |
|---|---|
| [architecture.md](architecture.md) | Full agent ladder — main agent, Planning Mode, Browser subagent, Teamwork sub-agents, background KI extractor, Critique/Gerrit specialist |
| [role-composition.md](role-composition.md) | The 10-role library and the 6 verified Teamwork archetypes (the "six agent types"). Lists which prompt fragments each role appends to the shared shell |
| [sentinel-cron-pattern.md](sentinel-cron-pattern.md) | New pattern proposal: a supervisor that uses **literal cron jobs** to monitor the orchestrator it spawned, with kill-and-respawn semantics |

## How it maps to this repo

| Antigravity feature | Pattern in this repo |
|---|---|
| Sentinel → Orchestrator → Workers/Reviewers/Auditor | [tiered-hierarchical-swarm](../../patterns/tiered-hierarchical-swarm.md) |
| Worker (implementer+qa+specialist) parallel instances `_<N>` | [flat-map-reduce](../../patterns/flat-map-reduce.md) |
| Reviewer + Critic + Auditor on a completion claim | [reviewer-swarm](../../patterns/reviewer-swarm.md) + [verifier-loop](../../patterns/verifier-loop.md) |
| Sentinel's mandatory **VICTORY AUDIT** before reporting done | [verifier-loop](../../patterns/verifier-loop.md) (escalated: rejection resumes the team) |
| Planning Mode (research → plan artifact → user-approval gate → execute → verify) | [pipeline-handoff-swarm](../../patterns/pipeline-handoff-swarm.md) |
| Soft handoff on context overflow + successor agents `_gen<N>` | [pipeline-handoff-swarm](../../patterns/pipeline-handoff-swarm.md) |
| Background KI extractor (jailed post-conversation agent) | [background-hybrid-swarm](../../patterns/background-hybrid-swarm.md) |
| Sentinel's two crons (`*/8`, `*/10`) on orchestrator | **NEW** — see [sentinel-cron-pattern.md](sentinel-cron-pattern.md) |

## What's novel here

1. **Role composition over class inheritance.** An agent is not a class — it is a list of named roles. Each role contributes a prompt fragment. Six concrete combinations ship in the binary.
2. **Append-only memory sections survive context truncation.** `BRIEFING.md` has two sections (`## My Identity`, `## Key Constraints`) the agent is forbidden from rewriting; the rest is mutable.
3. **The `.agents/` filesystem is the agent bus.** Coordination, state, handoffs all happen as files at conventional paths. Messages are for "I'm done / I'm stuck"; files are for content.
4. **Iron rule: no agent reuse after handoff.** Once a sub-agent writes `handoff.md`, it is permanently retired. Successor generations are tracked via `_gen<N>` suffix.
5. **VICTORY AUDIT is mandatory.** The Sentinel must spawn an Auditor archetype between "orchestrator says done" and "tell the user." On `VICTORY REJECTED`, the audit goes back to the orchestrator and the team resumes.

## Reproducibility

```sh
# Cache strings once (28 MB output)
strings -a "/Applications/Antigravity IDE.app/Contents/Resources/app/extensions/antigravity/bin/language_server_macos_arm" > /tmp/ag.str

# Find the six Teamwork archetypes
grep -n "You are a Teamwork agent with roles:" /tmp/ag.str

# Find the top-level personas
grep -n "^You are Antigravity\|^You are operating as\|^You are a Knowledge Items\|^You are in Planning Mode\|^You are an AI assistant knowledgeable about Google" /tmp/ag.str
```

Line numbers cited in the other docs are stable across builds within a major version; for 2.0.1 they match the table in [architecture.md](architecture.md).
