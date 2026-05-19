# Warp Agent Architecture

## Shape

```text
                       ┌────────────────────────────┐
                       │  Warp.app (terminal UI)    │
                       │  Resources/MacOS/stable    │ ← 373 MB Rust binary
                       │  Agent + chat + blocks     │   (system prompt server-side)
                       └──────────┬─────────────────┘
                                  │
              consults at trigger time
                                  ▼
              ┌───────────────────────────────────────┐
              │  Resources/bundled/skills/            │ ← Apache 2.0 plaintext
              │  11 skills + 1 mcp_skills/figma       │
              └────┬──────────────┬───────────────┬───┘
                   │              │               │
                   ▼              ▼               ▼
            SKILL.md         scripts/         references/
            (instructions)   (executable)     (load on demand)

  Skills can spawn subagents and shell out to other tools:

         ┌───────────────────┐
         │  create-skill     │
         │  • grader subagent│ ──► reads transcripts, evaluates expectations
         │  • comparator     │ ──► blind A/B judgment, no skill identity
         │  • analyzer       │ ──► post-hoc: WHY did the winner win?
         └─────────┬─────────┘
                   │
                   ▼ (subprocess)
         ┌───────────────────┐
         │   claude -p       │ ← Claude Code headless
         │   (eval execution │
         │    + LLM-judging) │
         └───────────────────┘

  Cloud agents live elsewhere:

         ┌─────────────────────────────────────────────┐
         │  Resources/bin/oz                            │  bash launcher
         │  "Orchestration platform for cloud agents"   │
         │                                              │
         │  oz agent / environment / mcp / run / model  │
         │  • Launch and inspect cloud agents           │
         │  • Schedule cloud agents to run in the future│
         │  • Manage environments cloud agents run in   │
         │  • Upload secrets to Oz's secure storage     │
         └─────────────────────────────────────────────┘
```

## The Skills format (Anthropic standard, Apache 2.0)

Verbatim from `create-skill/SKILL.md`:

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description required)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/    - Executable code for deterministic/repetitive tasks
    ├── references/ - Docs loaded into context as needed
    └── assets/     - Files used in output (templates, icons, fonts)
```

### Three-level progressive disclosure (verbatim)

> Skills use a three-level loading system:
> 1. **Metadata** (name + description) — Always in context (~100 words)
> 2. **SKILL.md body** — In context whenever skill triggers (<500 lines ideal)
> 3. **Bundled resources** — As needed (unlimited, scripts can execute without loading)

### YAML frontmatter (the minimum)

```yaml
---
name: skill-name
description: When this skill triggers + what it does (~100 words)
license: Complete terms in LICENSE.txt
---
```

The `compatibility` field is optional and rarely used. `license` is recommended when the skill carries Apache-style attribution.

### Triggering theory (verbatim from create-skill)

> Skills appear in the agent's `available_skills` list with their name + description, and the agent decides whether to consult a skill based on that description. The important thing to know is that the agent only consults skills for tasks it can't easily handle on its own — simple, one-step queries like "read this PDF" may not trigger a skill even if the description matches perfectly, because the agent can handle them directly with basic tools. Complex, multi-step, or specialized queries reliably trigger skills when the description matches.

### Domain organization pattern (verbatim)

> When a skill supports multiple domains/frameworks, organize by variant:
> ```
> cloud-deploy/
> ├── SKILL.md (workflow + selection)
> └── references/
>     ├── aws.md
>     ├── gcp.md
>     └── azure.md
> ```
> The agent reads only the relevant reference file.

This is the **Read-on-demand variant resolver pattern** — used in Warp's `claude-api` skill (which has per-language references: `python/`, `typescript/`, `go/`, `java/`, etc., each its own subdirectory).

## What ships in the bundle (Warp v0.2026.05.18)

```text
Resources/bundled/
├── metadata/version.json                          { "warp_version": "v0.2026.05.18.05.32.stable_02" }
├── skills/                                        11 skills (Apache 2.0)
│   ├── add-mcp-server/         (103 lines)        Add MCP server to Warp settings
│   ├── change-keybinding/      ( 84 lines)        Modify Warp keybindings
│   ├── claude-api/             (324 lines)        Anthropic API helper, 9-language code refs
│   ├── create-skill/           (456 lines)        ★ Skill creator + optimizer (the meta-skill)
│   ├── create-tab-config/      ( 29 lines)        New tab config from prompt
│   ├── feedback/               (240 lines)        File feedback issues, resolve Warp version/platform
│   ├── modify-settings/        ( 73 lines)        Update Warp settings.json
│   ├── oz-platform/            (273 lines)        How to use Oz cloud-agent platform
│   ├── pr-comments/            ( 65 lines)        Read + respond to PR review comments
│   ├── tab-configs/            (263 lines)        Manage Warp tab configurations
│   └── update-tab-config/      ( 25 lines)        Modify existing tab config
└── mcp_skills/
    └── figma/                                     8 sub-skills for Figma MCP integration
        ├── edit-figma-design/
        ├── figma-code-connect-components/
        ├── figma-create-design-system-rules/
        ├── figma-create-new-file/
        ├── figma-generate-design/
        ├── figma-generate-library/
        ├── figma-implement-design/
        └── figma-use/
```

Total in bundle: **1,935 lines of SKILL.md** across 11 skills (excluding `mcp_skills/figma/*` sub-skill SKILL.md files and `scripts/` Python).

## The `create-skill` meta-skill

This is the central skill. Its job is to **create new skills + iteratively improve existing ones**.

Three workflows:

1. **Capture intent** — interview the user, write SKILL.md draft
2. **Run + evaluate test cases** — spawn with-skill AND baseline runs in the same turn, grade with assertions, launch HTML viewer for human review
3. **Improve loop** — read feedback, rewrite, re-run

Plus a **description optimization loop** (see `skill-optimization-loop.md` in this folder) that uses an LLM to rewrite the skill's `description:` field, scored against a trigger eval set.

The skill bundles:
- `scripts/run_loop.py` — the optimization loop (328 lines)
- `scripts/run_eval.py` — single eval run (310 lines), shells out to `claude -p`
- `scripts/improve_description.py` — calls `claude -p` to rewrite the description (246 lines)
- `scripts/aggregate_benchmark.py` — produces `benchmark.json` + `benchmark.md`
- `scripts/generate_report.py` — HTML output
- `scripts/package_skill.py` — bundle skill into `.skill` archive
- `scripts/quick_validate.py` — pre-flight check
- `agents/grader.md` — grader subagent prompt
- `agents/comparator.md` — blind A/B comparator subagent prompt
- `agents/analyzer.md` — post-hoc winner-analysis subagent prompt
- `references/schemas.md` — JSON schemas for evals, grading, benchmarks
- `assets/eval_review.html` — interactive HTML viewer template

## Subprocess delegation to `claude -p`

Verbatim from `improve_description.py`:

```python
def _call_claude(prompt: str, model: str | None, timeout: int = 300) -> str:
    """Run `claude -p` with the prompt on stdin and return the text response.

    Prompt goes over stdin (not argv) because it embeds the full SKILL.md
    body and can easily exceed comfortable argv length.
    """
    cmd = ["claude", "-p", "--output-format", "text"]
    if model:
        cmd.extend(["--model", model])

    # Remove CLAUDECODE env var to allow nesting claude -p inside an
    # interactive session. The guard is for interactive terminal conflicts;
    # programmatic subprocess usage is safe.
    env = {k: v for k, v in os.environ.items() if k != "CLAUDECODE"}

    result = subprocess.run(
        cmd, input=prompt, capture_output=True, text=True,
        env=env, timeout=timeout,
    )
```

So when Warp's agent improves a skill description, it:
1. Constructs a prompt that includes the full SKILL.md body + eval results
2. Spawns `claude -p` as a subprocess
3. Pipes the prompt over stdin
4. Reads the rewritten description back from stdout

The `--model` flag is forwarded so the optimization uses the SAME model that's powering the calling session — "so the triggering test matches what the user actually experiences" (verbatim from the SKILL.md).

This pattern recurs in `run_eval.py` — the actual skill execution during eval runs is also delegated to `claude -p`.

**Why this matters**: Warp is composing on top of Claude Code rather than reimplementing the agent loop. The Rust binary handles the terminal UI, blocks, and shell integration; Claude Code handles the actual agentic reasoning.

## The three subagent roles (verbatim role definitions)

### Grader

> Evaluate expectations against an execution transcript and outputs.
>
> The Grader reviews a transcript and output files, then determines whether each expectation passes or fails. Provide clear evidence for each judgment.
>
> You have two jobs: grade the outputs, and critique the evals themselves. A passing grade on a weak assertion is worse than useless — it creates false confidence. When you notice an assertion that's trivially satisfied, or an important outcome that no assertion checks, say so.

### Comparator (blind)

> Compare two outputs WITHOUT knowing which skill produced them.
>
> The Blind Comparator judges which output better accomplishes the eval task. You receive two outputs labeled A and B, but you do NOT know which skill produced which. This prevents bias toward a particular skill or approach.
>
> Your judgment is based purely on output quality and task completion.

### Analyzer

> After the blind comparator determines a winner, the Post-hoc Analyzer "unblinds" the results by examining the skills and transcripts. The goal is to extract actionable insights: what made the winner better, and how can the loser be improved?

Inputs to Analyzer: `winner`, `winner_skill_path`, `winner_transcript_path`, `loser_skill_path`, `loser_transcript_path`, `comparison_result_path`, `output_path`.

This three-role pattern is essentially a **structured PR review**: Grader = lint/unit tests, Comparator = human review without seeing the author, Analyzer = post-merge retro.

## Oz: the cloud-agent CLI

`Resources/bin/oz` is a bash launcher (the actual binary is downloaded/elsewhere). `oz --help` output:

```
The orchestration platform for cloud agents

The Oz CLI is a tool for running, managing, and orchestrating coding agents at scale.
Use the CLI to:
* Launch and inspect cloud agents
* Schedule cloud agents to run in the future
* Manage the environments that cloud agents run in
* Upload secrets to Oz's secure storage

Commands:
  agent          Interact with Oz
  environment    Manage cloud environments [aliases: e]
  mcp            Manage MCP servers
  run            Manage runs
  model          Manage available models
  login          Log in to Warp
  logout         Log out of Warp
```

Plus subcommands referenced in the bundled `oz-platform` skill: schedule, secret, federate, harness-support, artifact, provider, integration.

**Oz is the equivalent of Antigravity's Cascade RPCs and Cursor's `Task(subagent_type="...")`** — except as a separate command-line tool rather than an in-IDE agent surface. Implication: Warp users orchestrate background fleets via `oz`, while their interactive terminal session uses local skills.

## What's NOT in the bundle

The system prompt that runs *inside* the Warp app (the persona, base tool definitions, agent loop) is not in plaintext. Grepping the 373 MB Rust binary for prompt openings (`"You are..."`, etc.) returns nothing — the prompt is server-side. The 40 MB string table contains tool names (`task`, `grep`) but no agent persona literals.

Compare:
- **Antigravity** ships its system prompts as static string literals in `language_server` C++ binary (RE-able with `strings`)
- **Cursor** ships its system prompts as JSX/template literals in `cursor-agent-exec/main.js` (RE-able after beautification)
- **Warp** keeps its system prompts server-side. Only the skills (which are user-extensible content) are open.

That's a deliberate design choice: Warp treats the agent persona as proprietary, but the skill format as a public standard.

## Tool surface (inferred)

From scripts that skills call: `claude` (Claude Code), `python`, `python -m scripts.X`, browser open. From the agent's own surface (visible in skill instructions): file read/write, shell exec, web fetch, "Task" subagent, "MCP" tools. Warp's own tool catalog isn't enumerated in the bundle — skills assume the tools by reference.
