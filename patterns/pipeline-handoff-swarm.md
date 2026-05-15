# Pipeline Handoff Swarm

## Shape

```text
┌──────────────┐
│ Parent Agent │
└──────┬───────┘
       ▼
┌──────────────┐
│ Researcher   │
│ investigates │
└──────┬───────┘
       ▼ resume_from / summary
┌──────────────┐
│ Implementer  │
│ writes patch │
└──────┬───────┘
       ▼ resume_from / diff
┌──────────────┐
│ Reviewer     │
│ audits patch │
└──────┬───────┘
       ▼
┌──────────────┐
│ Parent merge │
└──────────────┘
```

## Use when

- Work has natural stages.
- Later agents need artifacts from earlier agents.
- You want clean role separation.

## Prompt shape

```text
Researcher produces findings.md or structured JSON.
Implementer receives researcher output and exact write scope.
Reviewer receives diff and tests.
Parent owns final apply/merge.
```

## Guardrails

- Persist artifacts between stages.
- Do not rely only on hidden context; pass explicit summaries/files.
- Run tests after implementation and after review fixes.

## Failure modes

- Bad research contaminates later stages.
- Handoff summary omits important constraints.
- Reviewer lacks original requirements.
