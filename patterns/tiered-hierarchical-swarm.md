# Tiered Hierarchical Swarm

## Shape

```text
                      ┌──────────────┐
                      │ Root Parent  │
                      └──────┬───────┘
             ┌───────────────┴───────────────┐
             ▼                               ▼
      ┌──────────────┐                ┌──────────────┐
      │ Supervisor 1 │                │ Supervisor 2 │
      └──────┬───────┘                └──────┬───────┘
      ┌──────┴──────┐                 ┌──────┴──────┐
      ▼             ▼                 ▼             ▼
┌──────────┐   ┌──────────┐     ┌──────────┐   ┌──────────┐
│ Worker   │   │ Worker   │     │ Worker   │   │ Worker   │
│ 1A       │   │ 1B       │     │ 2A       │   │ 2B       │
└────┬─────┘   └────┬─────┘     └────┬─────┘   └────┬─────┘
     └──────┬───────┘                └──────┬───────┘
            ▼                               ▼
      ┌──────────────┐                ┌──────────────┐
      │ Supervisor   │                │ Supervisor   │
      │ local merge  │                │ local merge  │
      └──────┬───────┘                └──────┬───────┘
             └───────────────┬───────────────┘
                             ▼
                      ┌──────────────┐
                      │ Root merge   │
                      └──────────────┘
```

## Use when

- There are too many leaf tasks for one parent context.
- Each domain needs local coordination.
- Intermediate summaries reduce context pressure.

## Prompt shape

```text
Root spawns supervisors by domain.
Each supervisor must spawn real workers, not shell commands, for leaf tasks.
Each supervisor returns local synthesis and child IDs.
Root merges supervisor syntheses.
```

## Guardrails

- Set a depth limit.
- Force real subagents only when leaf tasks need context.
- Require supervisors to expose worker outputs, not just conclusions.

## Failure modes

- Supervisors collapse trivial leaves into shell commands.
- Context loss across levels hides evidence.
- Runaway nested spawning without a depth/width budget.
