# Reviewer Swarm

## Shape

```text
                  ┌────────────────┐
                  │ Parent Agent   │
                  │ has diff/patch │
                  └───────┬────────┘
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Security     │   │ Performance  │   │ Test         │
│ reviewer     │   │ reviewer     │   │ reviewer     │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       └──────────────────┼──────────────────┘
                          ▼
                 ┌────────────────┐
                 │ Parent decides │
                 │ fix / accept   │
                 └────────────────┘
```

## Use when

- One patch has multiple risk dimensions.
- You want independent review lenses.
- Parent can triage findings into actionable fixes.

## Prompt shape

```text
Security reviewer: look only for security/privacy regressions.
Performance reviewer: look only for CPU/memory/latency regressions.
Test reviewer: look only for missing/brittle tests.
Each reviewer returns severity, file, line, rationale, suggested fix.
```

## Guardrails

- Reviewers should not edit files.
- Parent applies fixes after consolidating duplicate findings.
- Require tight file/line anchors.

## Failure modes

- Reviewers produce broad advice instead of actionable comments.
- Parent treats all findings as equal severity.
- No final test run after fixes.
