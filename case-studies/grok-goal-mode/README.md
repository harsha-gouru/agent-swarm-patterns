# Case Study: Grok CLI — Goal Mode

> Reverse-engineered from Grok CLI **0.1.214** — the native Rust Mach-O at `~/.grok/downloads/grok-0.1.214-macos-aarch64` (~93 MB). Orchestrator and verifier prompts ship as plain UTF-8 in the binary's read-only data; every quoted block below was recovered statically and is reproducible (see [Reproducibility](#reproducibility)).

Grok CLI's general subagent mechanics (fresh / `fork_context` / `resume_from`, capability tiers, the live Tasks Pane) are already covered in [comparison-agent-control.md](../comparison-agent-control.md). This case study is about one thing Grok shipped in 0.1.214: **Goal mode** — a productized, verifier-gated swarm that runs an objective end-to-end without supervision.

## Why this is worth a case study

Goal mode is **new and undocumented**. Provenance, measured across three binaries:

| Marker | 0.1.210 | 0.1.211 | 0.1.214 |
|---|---|---|---|
| `goal orchestrator` prompt | 0 | 0 | ✅ |
| `update_goal` tool | 0 | 0 | ✅ (8 refs) |
| `GOAL ACHIEVED` verdict | 0 | 0 | ✅ |
| verifier-independence rule | 0 | 0 | ✅ |

It appears in the 0.1.213-alpha.7 changelog only as bug-fix lines ("Goal mode now auto-pauses on doom loops", "Removed `--budget` flag from `/goal`") — there is **no `goal` page** in `~/.grok/docs/user-guide/` and no `/goal` entry in the slash-commands doc. The entire design below was recovered from the binary.

What makes it interesting for this repo: Goal mode is the [verifier-loop](../../patterns/verifier-loop.md) pattern **productized and made mandatory**. It is not a pattern the model improvises — it is a fixed orchestrator/worker/verifier topology with the verifier-independence rule enforced in the prompt, a typed deliverable state machine, a persistent worker, and three named pause states. Most tools *suggest* a verifier; Grok *wires one in and forbids skipping it*.

## Files

| File | What's in it |
|---|---|
| [architecture.md](architecture.md) | The orchestrator / worker / verifier topology, the deliverable state machine, the `subagent_id` chain rule, the three-verdict system, the three pause states, and the verbatim recovered prompts |

## How it maps to this repo

| Goal-mode feature | Pattern in this repo |
|---|---|
| Orchestrator spawns worker, then a separate verifier, loops on FAIL | [verifier-loop](../../patterns/verifier-loop.md) — productized + mandatory |
| Deliverable list with `pending → in_progress → verified` states | [pipeline-handoff-swarm](../../patterns/pipeline-handoff-swarm.md) — typed stages |
| One persistent worker resumed across all deliverables | [background-hybrid-swarm](../../patterns/background-hybrid-swarm.md) — long-lived child |
| Orchestrator owns deliverables, never does work itself | [tiered-hierarchical-swarm](../../patterns/tiered-hierarchical-swarm.md) |
| `update_goal` powering a live dashboard | [runtime-primitives](../../patterns/runtime-primitives.md) — the status-view primitive |

## What's novel here (vs the other case studies)

| Axis | Grok Goal mode |
|---|---|
| **Verifier independence** | The orchestrator is *forbidden* to self-verify: *"NEVER self-verify by reading worker output — always use a verifier."* The verifier is a separate subagent that must **physically run the goal**, not read code. No other case study makes this a hard rule. |
| **Reality-grounded verdict** | *"'Theoretically achieved' … is NOT enough."* The verifier must produce concrete runtime evidence. Three verdicts only: GOAL ACHIEVED / GAP FOUND / TRULY BLOCKED. |
| **Persistent worker, chained ids** | One worker subagent is kept alive for the whole goal so it *accumulates context across deliverables*. Resumed by `resume_from` — and there's an explicit "`subagent_id` chain rule": every spawn returns a new id, the orchestrator must overwrite its saved id, a stale id silently restarts the worker with no memory. |
| **Typed state machine as a tool** | `update_goal(phase, deliverables[], completed)` — phases `planning → executing`, deliverable statuses `pending / in_progress / verified / needs_rework`. State is a tool call, not prose. |
| **Named failure states** | The runtime distinguishes `Paused (doom loop)`, `Paused (back-off)`, `Paused (verification blocked)` — auto-pause classification, not a generic stall. |

## How it differs from Claude Code Workflows

Both are "productized swarm": see [../claude-code/workflows-runtime.md](../claude-code/workflows-runtime.md).

| Axis | Claude Code Workflow | Grok Goal mode |
|---|---|---|
| Form | A JS script you author (`.claude/workflows/*.js`) | A built-in fixed topology; you only give the objective |
| Topology | Whatever the script codes | Fixed: orchestrator → worker → verifier |
| Determinism | Enforced (`Date.now`/`Math.random` removed) → resumable by memoised replay | Not script-based; resume is `subagent_id` chaining + `update_goal` state |
| Verifier | Optional, up to the script author | **Mandatory and independent** — cannot be skipped |
| Who writes the orchestration | The user (code) | xAI (the prompt is in the binary) |

Claude Code makes the swarm *programmable*; Grok makes one *specific* swarm — the verifier loop — a turnkey command. They are the two ends of the productization axis.

## Reproducibility

```sh
BIN=~/.grok/downloads/grok-0.1.214-macos-aarch64
file "$BIN"                                 # Mach-O 64-bit executable arm64 (Rust)

# Prompts live in rodata as plain UTF-8. Dump strings once, grep the cache:
strings -a "$BIN" > /tmp/grok214.str
grep -aF 'goal orchestrator'   /tmp/grok214.str   # orchestrator system prompt
grep -aF 'GOAL ACHIEVED'       /tmp/grok214.str   # the verdict block
grep -aF 'update_goal('        /tmp/grok214.str   # the state-machine tool
grep -aF 'never use fork_context' /tmp/grok214.str

# Strings splits the prompt on embedded NULs — for the full block, slice the file:
python3 - "$BIN" <<'EOF'
import sys; d=open(sys.argv[1],'rb').read()
i=d.find(b'You are the goal orchestrator')
print(''.join(chr(x) if 32<=x<127 or x==10 else '.' for x in d[i-200:i+1400]))
EOF
```

The leaked Rust source paths confirm the module layout: `crates/codegen/xai-grok-tools/src/implementations/grok_build/update_goal/mod.rs` (the `update_goal` tool) and `crates/codegen/xai-grok-pager/src/views/agent_status.rs` (the live dashboard). Version-diff workspace: `~/Developer/grok-re-214/` (binaries 0.1.210 / 0.1.211 / 0.1.214).
