# Research Scout Swarm

## Shape

```text
                 ┌──────────────┐
                 │ Parent Agent │
                 │ question     │
                 └──────┬───────┘
      ┌─────────────────┼─────────────────┐
      ▼                 ▼                 ▼
┌────────────┐    ┌────────────┐    ┌────────────┐
│ Docs scout │    │ Code scout │    │ Web scout  │
│ reads docs │    │ reads repo │    │ checks web │
└─────┬──────┘    └─────┬──────┘    └─────┬──────┘
      └─────────────────┼─────────────────┘
                        ▼
                ┌──────────────┐
                │ Synthesis    │
                │ by parent    │
                └──────────────┘
```

## Use when

- A question needs different evidence lanes.
- Each lane benefits from its own context window.
- You want contradictions surfaced, not averaged away.

## Prompt shape

```text
Docs scout: read official docs only.
Code scout: inspect local source only.
Issues scout: inspect issues/PRs only.
Each scout returns: claims, evidence, uncertainty, contradictions.
Parent merges and marks confirmed vs inferred.
```

## Guardrails

- Assign source boundaries explicitly.
- Require citations/paths from every scout.
- Parent labels unverified claims.

## Failure modes

- Scouts duplicate work.
- Web scout outruns evidence quality.
- Parent smooths over contradictions instead of preserving them.
