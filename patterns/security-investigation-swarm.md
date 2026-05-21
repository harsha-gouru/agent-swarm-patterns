# Security Investigation Swarm

Pattern family for vulnerability triage, reverse engineering, traffic analysis,
and evidence-backed security findings.

This pattern comes from transcript-mining real agent sessions: repeated loops of
scouting an unknown target, fanning out into independent evidence lanes,
verifying claims with commands/artifacts, then reducing everything into a
finding.

## Shape

```text
                 ┌────────────────┐
                 │ Orchestrator   │
                 │ goal + target  │
                 └───────┬────────┘
                         ▼
                 ┌────────────────┐
                 │ Artifact Scout │
                 │ map terrain    │
                 └───────┬────────┘
        ┌────────────────┼────────────────┬────────────────┐
        ▼                ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Code Auditor │  │ Binary/RE    │  │ Traffic/API  │  │ Runtime      │
│ source flows │  │ symbols      │  │ protocols    │  │ repro/capture │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       └─────────────────┴─────────────────┴─────────────────┘
                         ▼
                 ┌────────────────┐
                 │ Evidence Ledger │
                 └───────┬────────┘
                         ▼
                 ┌────────────────┐
                 │ Verifier       │
                 │ reject weak    │
                 │ claims         │
                 └───────┬────────┘
                         ▼
                 ┌────────────────┐
                 │ Finding Writer │
                 └────────────────┘
```

## Use when

- A vulnerability report needs fast first-pass triage.
- A repo, APK, IPA, binary, HAR, or cloud surface is unfamiliar.
- Multiple evidence sources can be inspected independently.
- Final output must be evidence-backed, not just a summary.

## Core loop

```text
scout target
→ split into independent evidence lanes
→ require every lane to emit claims + evidence
→ merge into an evidence ledger
→ verify the highest-impact claims
→ write finding / repro / next actions
```

## Agent roles

| Agent | Job | Output |
|---|---|---|
| Artifact Scout | Inventory target, identify artifact classes, choose next lanes | target map, entrypoints, recommended agents |
| Code Auditor | Trace source-level boundaries: auth, input parsing, secrets, filesystem, network | risky flows, file/line evidence |
| Binary/RE Agent | Inspect APK/IPA/Mach-O/ELF/firmware with strings, symbols, imports, metadata | runtime, protections, classes, opcodes, likely attack surface |
| Traffic/API Agent | Analyze HAR/CDP/logs/packets | endpoints, auth, telemetry, identifiers, state machine hints |
| Runtime/Repro Agent | Run target, reproduce behavior, capture logs/screenshots/traces | repro status, commands, captured artifacts |
| Verifier | Check claims against primary evidence | PASS/FAIL verdicts, unsupported claims, contradictions |
| Finding Writer | Turn verified evidence into a concise report | impact, repro, root cause, evidence, remediation/next work |

## Orchestration variants

### 1. Scout → Fanout → Verify

Best default.

```text
Scout maps target
→ specialists inspect independent surfaces
→ verifier checks claims
→ writer emits final finding
```

Use for unknown targets where one lane may be a dead end.

### 2. Parallel lane investigation

Use when several paths are plausible and cheap to start.

```text
Lane 1: static source/code review
Lane 2: binary/symbol/string analysis
Lane 3: network/API/traffic analysis
Lane 4: runtime capture/reproduction
Lane 5: docs/reference comparison
```

The parent should kill or de-prioritize lanes that stop producing evidence.

### 3. Repro → Root Cause → Impact → Regression

Use for vulnerability reports.

```text
Repro Agent
→ Root Cause Agent
→ Impact Agent
→ Regression/Test Agent
→ Finding Writer
```

Gate each stage. If the repro fails, do not let later agents invent impact.

### 4. Race / first valid path

Use when the fastest valid evidence path matters more than comparing all paths.

```text
strategy A: inspect source
strategy B: replay/capture traffic
strategy C: run dynamic repro
strategy D: inspect binary/imports
first verified result wins
```

Define success criteria before launch.

### 5. Map-reduce evidence mining

Use for large corpora: transcripts, logs, many files, many classes, many HARs.

```text
split corpus
→ mapper agents extract claims/tool sequences/failures
→ reducer clusters patterns
→ verifier checks representative samples
```

This is the right pattern for turning agent transcripts into reusable playbooks.

### 6. Red team / blue team / judge

Use for architecture/security review.

```text
Design Explainer: intended boundary
Red Team: how to break it
Blue Team: existing mitigations/fixes
Judge: evidence, severity, missing tests
```

Good for authz, privacy, sandbox, secrets, and trust-boundary reviews.

### 7. Evidence ledger swarm

Use when trust matters more than speed.

Every child writes claims in a machine-checkable shape:

```json
{
  "claim": "Endpoint includes a stable user identifier",
  "evidence": ["HAR request 42", "response field user.id"],
  "confidence": "high",
  "verified_by": null,
  "next_check": "replay request without auth cookie"
}
```

Parent only promotes claims after verification.

## Prompt shape

```text
You are <role>.
Target: <absolute path / artifact / capture>.
Stay inside your lane. Do not edit files unless explicitly assigned.

Return:
- findings: concrete claims only
- evidence: exact files, commands, request IDs, symbols, or log lines
- uncertainty: what is inferred or unverified
- next_checks: highest-leverage verification steps
```

## Guardrails

- Scout before fanout; otherwise specialists duplicate work.
- Split only independent lanes. Dependent steps should be a pipeline.
- Give write access to at most one lane unless using isolated worktrees.
- Require evidence paths/request IDs/commands from every child.
- Parent owns final verification and severity.
- Preserve failed paths; they are useful negative evidence.

## Failure modes

- Children produce summaries without evidence.
- Parent averages contradictory claims instead of resolving them.
- Race pattern accepts the fastest plausible answer, not the first verified one.
- Repro/impact agents run before the issue is actually reproduced.
- Too many agents are launched before the target is scoped.
