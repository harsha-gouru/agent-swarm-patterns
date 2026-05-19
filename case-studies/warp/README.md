# Case Study: Warp (Oz / Skills)

> Extracted from `Warp.app` v0.2026.05.18.05.32.stable_02. Unlike the other case studies in this repo, Warp's agent layer is **partially open-sourced**: the entire `Resources/bundled/skills/` directory ships as plaintext markdown + Python under **Apache License 2.0** (`LICENSE.txt` next to each `SKILL.md`). No reverse engineering of the binary needed — just read the files.

Warp's design point is different from both Antigravity and Cursor:
- It ships **Anthropic's exact skills format** (YAML frontmatter + markdown + scripts/references/assets bundled-resources tree, 3-level progressive disclosure).
- It delegates skill execution to **`claude -p`** (Claude Code's headless mode) as a subprocess.
- It ships an **agent-driven skill optimization loop** — an LLM iteratively rewrites a skill's description, scored against a stratified train/test split of trigger evals.
- Cloud-agent orchestration lives in a separate CLI called **`oz`**.

## Files

| File | What's in it |
|---|---|
| [architecture.md](architecture.md) | The Skills format, three-level progressive disclosure, Oz cloud-agent platform, subprocess-to-`claude -p` delegation, what ships in the bundle |
| [skill-optimization-loop.md](skill-optimization-loop.md) | New pattern: agent-driven skill description optimization with stratified train/test split, eval HTML viewer, blind comparison subagents |
| [skill-catalog.md](skill-catalog.md) | The 11 bundled skills + the 1 bundled MCP skill, with verbatim descriptions and what each does |

## How it maps to this repo

| Warp feature | Pattern in this repo |
|---|---|
| `grader` / `comparator` / `analyzer` subagents in create-skill | [reviewer-swarm](../../patterns/reviewer-swarm.md) |
| Spawn with-skill AND baseline runs **in the same turn** for each test case | [flat-map-reduce](../../patterns/flat-map-reduce.md) |
| Blind A/B comparison (comparator doesn't know which side is which) | [verifier-loop](../../patterns/verifier-loop.md) with anti-bias scaffold |
| Iterate up to 5 rounds, pick best by test (not train) score | **NEW** — see [skill-optimization-loop.md](skill-optimization-loop.md) |
| `oz` CLI for cloud-agent runs | [background-hybrid-swarm](../../patterns/background-hybrid-swarm.md) |
| Skills bundling scripts that execute without loading into context | [runtime-primitives](../../patterns/runtime-primitives.md) — progressive disclosure |

## What's novel here

1. **Skills are Apache 2.0 plaintext.** Unlike Antigravity (skills shipped via Firebase) or Cursor (subagent types compiled into a minified bundle), Warp ships its agent's instructions as readable markdown the user can edit, fork, or write from scratch. The bundled `create-skill` SKILL.md is essentially a publishable manual for the entire format.
2. **Anthropic's skills format, verbatim.** YAML frontmatter (`name`, `description`, optional `license`, `compatibility`). Bundled resources tree: `scripts/` (executable), `references/` (load-on-demand), `assets/` (output templates). Three-level progressive disclosure (~100-word metadata always loaded → SKILL.md body when triggered → resources as needed). This is Claude Skills — Warp adopted the standard.
3. **Agent ↔ Claude Code subprocess delegation.** `improve_description.py` and `run_eval.py` shell out to `claude -p --output-format text`. Warp's agent invokes Claude Code headless for the actual skill execution and grading. This is the first agent product I've seen explicitly composing on top of another agent product.
4. **Train/test split on trigger queries.** The skill optimization loop generates ~20 eval queries (8-10 should-trigger, 8-10 should-not-trigger near-misses), splits 60/40 stratified, runs each query 3x for reliable trigger rates, then picks the best description by held-out test score — not train score — to avoid overfitting.
5. **Skill-creation-by-skill.** `create-skill` is itself a skill. The agent uses a skill to write skills.
6. **Cloud agent platform as a separate CLI.** `oz` is its own binary at `Resources/bin/oz` — agent · environment · mcp · run · model commands. Warp clearly separates "agent in your terminal" from "cloud-orchestrated agents."

## Reproducibility

No tools needed — just open the bundle directory:

```sh
# All the skills
open /Applications/Warp.app/Contents/Resources/bundled/skills/

# Each skill is plaintext markdown + scripts:
cat /Applications/Warp.app/Contents/Resources/bundled/skills/create-skill/SKILL.md
ls /Applications/Warp.app/Contents/Resources/bundled/skills/create-skill/scripts/
ls /Applications/Warp.app/Contents/Resources/bundled/skills/create-skill/agents/

# Three subagent prompts that drive the optimization loop:
cat /Applications/Warp.app/Contents/Resources/bundled/skills/create-skill/agents/grader.md
cat /Applications/Warp.app/Contents/Resources/bundled/skills/create-skill/agents/comparator.md
cat /Applications/Warp.app/Contents/Resources/bundled/skills/create-skill/agents/analyzer.md

# Cloud-agent CLI:
/Applications/Warp.app/Contents/Resources/bin/oz --help
```

The system prompts that run *inside the Warp app itself* are not in the bundle — Warp's main system prompt is fetched at runtime from the server. So the agentic surface that's visible to us is: skills + subagent prompts + the `oz` CLI. That's enough to map the design.
