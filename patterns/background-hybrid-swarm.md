# Background Hybrid Swarm

## Shape

```text
                   ┌──────────────┐
                   │ Parent Agent │
                   └──────┬───────┘
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Test watcher │   │ Dev server   │   │ Subagent     │
│ bash task    │   │ bash task    │   │ code review  │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ▼
                  ┌──────────────┐
                  │ wait / poll  │
                  │ task IDs     │
                  └──────────────┘
```

## Use when

- Long-running commands should run while agents continue.
- Tests/builds/dev servers are useful feedback loops.
- Parent needs to poll or wait later.

## Prompt shape

```text
Start test watcher as background command.
Start dev server as background command.
Spawn reviewer subagent.
Continue implementation.
Poll task IDs before final response.
```

## Guardrails

- Capture output files for every background task.
- Kill long-running tasks when done.
- Avoid duplicate servers on the same port.

## Failure modes

- Background tasks keep running after the session.
- Parent forgets to read final output.
- Logs are truncated unless output files are persisted.
