# Sentinel Cron Supervisor

A new pattern observed in Antigravity, distinct from the existing `verifier-loop` and `tiered-hierarchical-swarm` patterns. **The supervisor is asynchronous, not turn-driven.** It schedules itself to wake on a wall-clock interval and check on the orchestrator it spawned.

## Shape

```text
            ┌────────────────────────────────┐
            │  Sentinel                      │
            │  user_liaison + sentinel_      │
            │  reporter + dispatcher         │
            │                                │
            │  cron */8 * * * *  ──┐         │
            │  cron */10 * * * * ──┤         │
            └────────────────────┬─┴─────────┘
                                 │
                  spawns + monitors
                                 │
                                 ▼
            ┌────────────────────────────────┐
            │  Orchestrator                  │
            │  • writes progress.md per step │
            │  • includes "Last visited: ts" │
            │  • spawns workers/reviewers    │
            └────────────────────┬───────────┘
                                 │
                                 ▼
                       team of sub-agents

Every 8 minutes (Cron 1 — Progress Reporting):
  read orchestrator's progress.md + BRIEFING.md
  scan top 5 recently modified files (first 30 lines each)
  emit 3-5 bullet summary to user

Every 10 minutes (Cron 2 — Liveness Check):
  check progress.md mtime
  if stale > 10 min   → nudge orchestrator
  if still stale      → kill orchestrator, spawn fresh from
                        ORIGINAL_REQUEST.md
  if successor exists → switch monitoring target to successor
                        (successor inherits the "active" slot)
```

## Use when

- The orchestrator may stall, hang on a slow tool, or exhaust its context mid-run.
- You want the user to get progress updates *without* polling the orchestrator (the orchestrator should be working, not narrating).
- Multi-hour runs need durable supervision that survives the supervisor's own context windows.
- You want a *single* place to enforce "completion claims must be audited before reporting done."

## Prompt shape

```text
You are the supervisor for ONE orchestrator. You do NOT do technical work.

Run two cron jobs on yourself:
  Cron 1 (*/8 * * * *):
    Read <orch>/progress.md and <orch>/BRIEFING.md.
    Scan the top 5 most-recently-modified files in <orch>/ (first 30 lines).
    Emit 3-5 bullets to the user — concise, no structured presentation.

  Cron 2 (*/10 * * * *):
    Check mtime of <orch>/progress.md.
    If > 10 minutes since last update:
      - First strike: send a nudge message to the orchestrator.
      - Second strike (still stale): kill and respawn the orchestrator
        from ORIGINAL_REQUEST.md.
    If the orchestrator handed off to a successor (soft handoff):
      - Switch your monitoring target to the successor.
      - Do NOT respawn. A completed handoff is NOT death.

When the orchestrator reports completion:
  1. Spawn a victory_auditor (Auditor archetype).
  2. Forward its verdict to the orchestrator if VICTORY REJECTED.
  3. Do NOT report success to the user without VICTORY CONFIRMED.
```

## Guardrails

- **Sentinel must not write code.** Its only job is dispatch + monitoring. Mixing concerns lets it justify "fixing things itself" when the orchestrator hangs.
- **Append-only `ORIGINAL_REQUEST.md`.** First substantive user message is recorded verbatim; later non-trivial messages append under new timestamps. Skip continuations, acks, and approvals. The Sentinel must NEVER edit or summarize prior entries — this is the authoritative record that survives context truncation, crashes, and agent succession.
- **One Sentinel, one orchestrator.** Don't fan out at this level. The hierarchy widens below the orchestrator, not at the Sentinel.
- **Successor tracking is mandatory.** When the orchestrator hands off to a successor, the Sentinel updates BRIEFING.md with the new conversation ID and switches its cron target. Treating a soft handoff as death will respawn over a live successor and lose work.
- **Liveness != productivity.** `progress.md` updates only count if they include `Last visited: <timestamp>`. A timestamp-only update during a long build is allowed; a stale mtime is not.
- **Two strikes before kill.** Nudge first, kill on second stale check. Single-strike kill leads to thrashing when the orchestrator is mid-`run_command`.

## Failure modes

- **Sentinel grows ambitious.** A Sentinel that starts editing code or making technical decisions loses its independence from the team it supervises. Hard-wire the role definition to "no technical decisions — relay only."
- **Cron interval mismatch.** If Cron 2's window is shorter than the longest legitimate tool call, you'll kill working orchestrators. Antigravity uses `*/10` and requires updates `every 5 minutes even if only the timestamp changes` — give a 2x safety margin.
- **Successor confusion.** If your runtime doesn't track `_gen<N>` suffixes, the Sentinel can lose the active agent. Make conversation IDs explicit in BRIEFING.md.
- **Skipping VICTORY AUDIT to look fast.** The temptation to report "done" the moment the orchestrator finishes is what auditors exist to prevent. The Sentinel must block on the audit verdict.
- **Two crons drifting in phase.** Use independent timers; don't piggyback Cron 1 on Cron 2's wakeup.

## Differences from existing patterns

- **vs `verifier-loop`** — verifier-loop is turn-driven: parent emits work, then verifies. Sentinel is wall-clock driven: it wakes on a timer to check on a long-running child it doesn't own the turn for.
- **vs `tiered-hierarchical-swarm`** — tiered hierarchies are about *width* (many leaves need supervisors). Sentinel is about *duration* (one long task needs durable supervision).
- **vs `background-hybrid-swarm`** — background-hybrid coordinates an agent with a long-running shell command. Sentinel coordinates an agent with another agent that may itself be hung in a long-running shell command.

## Runtime primitives needed

To implement this pattern your runtime must provide:

- Wall-clock cron scheduling that wakes an agent (not just a shell command).
- A way to read another agent's working directory (`.agents/<orch>/progress.md`).
- A way to inspect another agent's mtime/heartbeat metadata.
- A way to kill + respawn an agent while preserving its `ORIGINAL_REQUEST.md`.
- A way to detect a soft handoff (successor `_gen<N>` directory appears) and switch monitoring target without respawning.

Antigravity's `cron`-like primitives plus `.agents/` filesystem give it all of these. Smaller runtimes can approximate with a separate process that pokes a `progress.md` watcher and re-invokes the agent SDK on stale heartbeats.

## Source

Verbatim from `language_server.str:90628` (the user_liaison + sentinel_reporter + dispatcher archetype's prompt). The `*/8 * * * *` and `*/10 * * * *` intervals are baked into the static prompt — they're not user-configurable.
