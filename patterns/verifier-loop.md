# Verifier Loop

## Shape

```text
┌──────────────┐
│ Parent       │
│ implements   │
└──────┬───────┘
       ▼
┌──────────────┐
│ Verifier     │
│ checks work  │
└──────┬───────┘
       ▼
┌──────────────┐
│ Parent fixes │
│ or ships     │
└──────────────┘
```

## Use when

- Work is risky or irreversible.
- Parent may have tunnel vision.
- You need an independent second opinion.

## Prompt shape

```text
Verifier receives exact requirements, diff, and test command.
Verifier must return VERDICT: PASS or VERDICT: FAIL.
If FAIL, include actionable file/line issues.
Parent fixes or explains why not.
```

## Guardrails

- Verifier should be read-only by default.
- Parent still runs objective checks.
- Do not let verifier rewrite the implementation unless explicitly staged.

## Failure modes

- Verifier rubber-stamps without evidence.
- Parent ignores legitimate failures.
- Verifier lacks full task context.
