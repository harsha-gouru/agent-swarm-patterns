# Warp Skill Bundle

> This doc is about Warp's **content** layer (skills as Apache-2.0 markdown). For the actual **agentic flow** — how agents spawn each other, how they communicate, what tools they have — see [architecture.md](architecture.md), [spawn-and-messaging.md](spawn-and-messaging.md), and [tool-catalog.md](tool-catalog.md).

All skills shipped in `/Applications/Warp.app/Contents/Resources/bundled/`. Apache 2.0 licensed (each skill has its own `LICENSE.txt` where applicable). Skills are referenced from the agent's protobuf surface via `SkillRef` (passed in `StartAgentV2.Remote.skills` and `RunAgents.skills`) and loaded via the `ReadSkill` tool.

## Top-level skills (`bundled/skills/`)

### `create-skill` — the meta-skill (456 lines)

> Create new skills, modify and improve existing skills, and measure skill performance. Use when users want to create a skill from scratch, edit, or optimize an existing skill, run evals to test a skill, benchmark skill performance with variance analysis, or optimize a skill's description for better triggering accuracy.

Bundles:
- 9 Python scripts (`run_loop.py`, `run_eval.py`, `improve_description.py`, `aggregate_benchmark.py`, `generate_report.py`, `package_skill.py`, `quick_validate.py`, `utils.py`)
- 3 subagent prompts (`agents/grader.md`, `agents/comparator.md`, `agents/analyzer.md`)
- 1 reference (`references/schemas.md` — JSON schemas for evals, grading, benchmarks)
- HTML viewer template (`assets/eval_review.html`)

This bundle is itself a self-contained skill-optimization loop (generate → eval → grade → improve → benchmark → report).

### `claude-api` — Anthropic SDK helper (324 lines)

> Build, debug, and optimize Claude API / Anthropic SDK apps. Apps built with this skill should include prompt caching. Also handles migrating existing Claude API code between Claude model versions (4.5 → 4.6, 4.6 → 4.7, retired-model replacements). TRIGGER when: code imports `anthropic`/`@anthropic-ai/sdk`; user asks for the Claude API, Anthropic SDK, or Managed Agents; user adds/modifies/tunes a Claude feature (caching, thinking, compaction, tool use, batch, files, citations, memory) or model (Opus/Sonnet/Haiku) in a file; questions about prompt caching / cache hit rate in an Anthropic SDK project. SKIP: file imports `openai`/other-provider SDK, filename like `*-openai.py`/`*-generic.py`, provider-neutral code, general programming/ML.

The description is a **fully-specified TRIGGER/SKIP rule** rather than a sentence. This is the most prescriptive description style in the bundle — it tells the agent exactly which file imports and topics activate this skill, and which adjacent topics should NOT.

Bundles per-language code references:
```
claude-api/
├── SKILL.md
├── csharp/ curl/ go/ java/ php/ python/ ruby/ shared/ typescript/
└── LICENSE.txt (Apache 2.0)
```

Variant-by-domain pattern: agent reads only the language-relevant subdirectory.

### `oz-platform` — Cloud agent CLI guide (273 lines)

> Use Warp's REST API and command line to run, configure, and inspect Oz cloud agents.

References: `third-party-clis.md`. Documents `oz` subcommands: agent, environment, mcp, run, model, schedule, secret, federate, harness-support, artifact, provider, integration.

### `feedback` — File a Warp bug report (240 lines)

> Turn rough feedback about the Warp app into a filed GitHub issue or duplicate-issue response for `warpdotdev/warp`. Use ONLY when the user explicitly wants to report a problem with the Warp terminal/IDE/app itself — not when they're working on their own code, managing their own GitHub repos, or doing general software development tasks.

The description includes an explicit **negative-scope clause** — when NOT to use. Same pattern as `claude-api`.

Bundles: `scripts/file_feedback_issue.py`, `resolve_platform.py`, `resolve_warp_version.py`, references for examples / logs / platforms.

### `tab-configs` / `create-tab-config` / `update-tab-config` (263 + 29 + 25 lines)

The tab-config triad. `tab-configs` is the schema reference; the other two are the create/update workflows. Three skills for one feature, organized around a shared canonical reference.

### `add-mcp-server` (103 lines)

> Use this skill when helping users add MCP servers to their Warp configuration.

Frontmatter name is `agent-add-mcp` (only skill where directory name ≠ frontmatter name).

### `modify-settings` (73 lines)

> View or modify Warp application settings using the bundled JSON schema for guidance.

Bundles `scripts/` for settings file manipulation. Pairs with `settings_schema.json` in the parent Resources directory.

### `change-keybinding` (84 lines)

> Customize Warp keyboard shortcuts (keybindings, keymappings) by editing the user's keybindings.yaml file. Use when the user asks to remap a key combination, rebind an action, change a shortcut, or remove a default keybinding.

Includes trigger phrases verbatim: "change ctrl+space to ctrl+s", "rebind the command palette to cmd+p", "remove the default for X". This is the "include realistic example phrasings" pattern recommended by `create-skill`.

### `pr-comments` (65 lines)

> Fetch and display GitHub PR review comments for the current branch.

The shortest description in the bundle. Bundles a script that wraps `gh api`.

## MCP skills (`bundled/mcp_skills/`)

Only one MCP server has bundled skills: **Figma**. 8 sub-skills covering the full Figma MCP surface:

| Sub-skill | Purpose |
|---|---|
| `figma-use` | **MANDATORY prerequisite** — must invoke before every `use_figma` tool call. NEVER call `use_figma` directly without loading this first. |
| `figma-create-new-file` | Create a new blank Figma/FigJam file. |
| `edit-figma-design` | Create or update Figma designs from a written description. |
| `figma-generate-design` | Translate an app page/view into Figma using design system components. |
| `figma-generate-library` | Build a professional-grade design system in Figma from a codebase. |
| `figma-implement-design` | Translate Figma designs into production code with 1:1 visual fidelity. |
| `figma-code-connect-components` | Map Figma design components to code components via Code Connect. |
| `figma-create-design-system-rules` | Generate custom design system rules for the user's codebase. |

The frontmatter has `metadata.mcp-server: figma` linking the skill to a specific MCP server. Some skills also carry `disable-model-invocation: true` (only `figma-create-new-file` does) — which suppresses automatic skill triggering for that one.

The `figma-use` pattern is interesting — a skill marked as **MANDATORY prerequisite** for direct tool calls. Effectively the skill loads context the model needs to use the tool correctly. If the model calls `use_figma` without loading the skill first, "common, hard-to-debug failures" follow. This is a soft enforcement: the skill description shouts about it, but nothing in the runtime actually blocks the call.

## Patterns across descriptions

Cataloguing what works:

| Pattern | Examples | Effect |
|---|---|---|
| **TRIGGER / SKIP** explicit clauses | `claude-api`, `feedback` | Sharp boundaries — agent knows exactly when NOT to invoke |
| **Negative-scope** ("SKIP when…") | `feedback`, `claude-api` | Defends against near-miss triggering |
| **Realistic example phrasings** | `change-keybinding`, `figma-code-connect-components` | Helps the agent match casual user phrasing |
| **MANDATORY prerequisite** | `figma-use` | Soft-enforces a setup skill before a tool call |
| **Single short sentence** | `pr-comments`, `modify-settings` | Used when the skill is unambiguous |
| **TOOL-OUTPUT triggering** | `claude-api` ("code imports `anthropic`") | The skill triggers on file content, not just user text |

Verbatim from `create-skill` SKILL.md on description style:

> The description is the primary triggering mechanism — include both what the skill does AND specific contexts for when to use it. All "when to use" info goes here, not in the body.
>
> Note: currently agents have a tendency to "undertrigger" skills — to not use them when they'd be useful. To combat this, please make the skill descriptions a little bit "pushy". So for instance, instead of "How to build a simple fast dashboard to display internal data.", you might write "How to build a simple fast dashboard to display internal data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"

So the bias in the catalog tilts toward over-trigger and is corrected by the optimization loop's stratified test set.

## What's missing

Notably absent:
- **No git skill.** Cursor has `git`-related subagents (best-of-n-runner, command-exec); Warp doesn't ship one. Git operations are delegated to `claude -p`.
- **No code-search / explore skill.** No equivalent of Antigravity's `codebase_search` or Cursor's `explore` subagent.
- **No browser / web automation skill.** Cursor has `browserUse`; Antigravity has the full Chrome DevTools MCP. Warp has neither.
- **No background-agent skill.** `oz-platform` documents the cloud-agent CLI, but the interactive Warp app doesn't ship skills for spawning local background subagents.

The pattern is clear: **Warp ships skills for Warp-specific surfaces** (tab configs, keybindings, settings, MCP setup, feedback, Oz cloud agents) and **Anthropic SDK** (claude-api). Anything else, the agent must improvise from its base tool surface. There's no expectation of code-editing skills because the user is, in principle, going to ask the agent to run shell commands or invoke `claude -p` for that.

Compare:
- **Antigravity** bundles ~no skills by default but provides a workspace `.agent/skills/` directory + Firebase-distributed skill packs.
- **Cursor** bundles 13+ built-in subagent types that act like skills, plus a plugin marketplace for custom subagents.
- **Warp** bundles 11 + 8 plaintext skills under Apache 2.0 and provides `create-skill` for users to write more.

Warp's choice puts the most weight on the user's ability to author skills. The bundle's `create-skill` SKILL.md is effectively a 456-line manual on how to do that, including an optimizing meta-loop.
