# Case Study: Cursor "Grind"

> Reverse-engineered from `Cursor.app` 3.4.20 (VSCode 1.105.1) — specifically the `cursor-agent-exec` extension's 8.3 MB minified `dist/main.js`. All quoted material was recovered as static JSX/template-literal strings and is reproducible.

Cursor ships a named multi-agent system called **Grind**. Compared to Antigravity (see `../google-antigravity/`), the design point is opposite on the coordination-bus axis: **Grind uses git itself as the bus**. Each agent runs in its own git clone; workers push commits to a shared branch with rebase-and-retry; everything that isn't a commit vanishes when the agent ends.

## Files

| File | What's in it |
|---|---|
| [architecture.md](architecture.md) | The Grind hierarchy (Orchestrator → Planner → Sub-planners + Workers), isolation modes, what survives vs what vanishes |
| [subagent-catalog.md](subagent-catalog.md) | The 13+ built-in subagent types invokable via the `Task` tool, with verbatim descriptions, permission modes, model overrides, continuation policies |
| [git-as-bus-pattern.md](git-as-bus-pattern.md) | New pattern proposal: using git itself as the swarm coordination primitive (rebase-retry, "only your commit survives") |

## How it maps to this repo

| Cursor feature | Pattern in this repo |
|---|---|
| Orchestrator → Planner → Workers tree | [tiered-hierarchical-swarm](../../patterns/tiered-hierarchical-swarm.md) |
| Workers push commits in parallel to shared branch | [flat-map-reduce](../../patterns/flat-map-reduce.md) with git-merge as reducer |
| `best-of-n-runner` subagent (isolated git worktree per attempt) | [best-of-n-worktree](../../patterns/best-of-n-worktree.md) |
| `explore` readonly subagent + `pastConversationExplorer` | [research-scout-swarm](../../patterns/research-scout-swarm.md) |
| `Reflect` tool (meta-troubleshooting when stuck) | [verifier-loop](../../patterns/verifier-loop.md) variant |
| `run_in_background=true` + later resume | [background-hybrid-swarm](../../patterns/background-hybrid-swarm.md) |
| Rebase-retry-on-push with exponential backoff | **NEW** — see [git-as-bus-pattern.md](git-as-bus-pattern.md) |

## What's novel here (vs Antigravity)

| Axis | Antigravity (Google) | Cursor (Grind) |
|---|---|---|
| **Coordination bus** | `.agents/` filesystem with `handoff.md` files | Git itself — workers push commits, planners pull rebased clones |
| **Agent persistence** | Files persist; agents retire after `handoff.md` | "When you finish, EVERYTHING vanishes. Only your commit survives." |
| **Agent definition model** | Role composition (10-role library, 6 archetype combos) | Subagent-type registry with per-type system-prompt override, tool override, model override, continuation policy |
| **Stateful resume** | New `_gen<N>` directory per generation | `resumeModeOverride: LAST_AGENT_SAME_TYPE` — Task tool resumes the prior subagent of the same type |
| **Supervisor model** | Sentinel runs two crons against orchestrator's `progress.md` | Continuation policy: `idleThreshold=3, maxLoops=100` with continuation messages |
| **Adversarial review** | Auditor archetype + mandatory VICTORY AUDIT | No mandatory verifier — relies on `Reflect` tool self-triggering |
| **Custom-agent extension model** | Workspace `.agent/skills/` + Firebase-distributed skill packs | Plugin manifests with `skills + subagents + hooks + rules + mcpServers + commands + variables` |
| **Customer-facing swarm** | Sentinel + Orchestrator + Workers | "Grind" — Orchestrator + Planner + Workers + Sub-planners |
| **User-visible model** | Single chat with subagents transparent | Subagents are first-class: `Task(subagent_type=..., prompt=..., run_in_background=true)` |
| **Reflective tool** | None named | `Reflect` tool — explicit meta-reasoning step |

## Reproducibility

```sh
# Extract the agent extension's bundled JS
APP=/Applications/Cursor.app/Contents/Resources/app/extensions/cursor-agent-exec/dist/main.js

# Naive beautify (the file is one minified line)
node -e "const fs=require('fs');const s=fs.readFileSync('$APP','utf8');fs.writeFileSync('/tmp/agent-exec.js',s.replace(/;/g,';\n').replace(/\{/g,'{\n').replace(/\}/g,'\n}\n'));"

# Find Grind architecture
grep -n '"Grind"\|"Architecture"\|"Information Flow"' /tmp/agent-exec.js

# Find the subagent catalog
grep -n 'subagent_type:new nd._L' /tmp/agent-exec.js

# Find Planner + Worker prompts
grep -n '"You Are the Planner"\|"You Are a Worker"' /tmp/agent-exec.js
```

Line refs cited in the other docs are for Cursor 3.4.20; they shift between releases but the symbol names (Q1, D1, C1, w1, v1, E1, x1, B1, S1, R1) are stable within a release.
