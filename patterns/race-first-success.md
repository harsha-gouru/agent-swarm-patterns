# Race / First-Success Swarm

## Shape

```text
                 ┌──────────────┐
                 │ Parent Agent │
                 └──────┬───────┘
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Strategy A  │  │ Strategy B  │  │ Strategy C  │
│ maybe slow  │  │ maybe wins  │  │ maybe fails │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       │                ▼                │
       │        ┌─────────────┐          │
       │        │ first good  │          │
       │        │ result      │          │
       │        └──────┬──────┘          │
       │               ▼                 │
       │        ┌─────────────┐          │
       └───────►│ cancel rest │◄─────────┘
                └─────────────┘
```

## Use when

- Latency matters.
- Any one valid answer is enough.
- Strategies have unpredictable runtimes.

## Prompt shape

```text
Launch independent strategies.
Wait for first successful result.
Validate the result.
Cancel or ignore remaining tasks.
```

## Guardrails

- Define success criteria before launching.
- Parent validates the first result before killing the rest.
- Keep side effects isolated; losing strategies may still be running.

## Failure modes

- First result is fast but wrong.
- Cancel does not clean up child side effects.
- Parent misses better but slower answers.
