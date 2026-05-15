# Runtime Primitives for a Swarm CLI

## Minimal object model

```text
Task {
  id
  kind: bash | agent | server | job
  parent_id
  cwd
  status: queued | running | completed | failed | cancelled
  started_at
  ended_at
  output_file
  metadata
}

AgentSession {
  id
  parent_task_id?
  cwd
  model
  system_prompt
  chat_history.jsonl
  events.jsonl
  updates.jsonl
  summary.json
}
```

## Required operations

```text
spawn(task) -> task_id
wait(task_ids, mode=all|any|first_success) -> results
poll(task_id) -> status + partial output
kill(task_id) -> cancellation result
resume(agent_session_id) -> session handle
apply(worktree_id) -> patch into parent repo
```

## Event log contract

```text
turn_started
first_token
tool_started
tool_completed
task_started
task_completed
permission_requested
permission_resolved
turn_ended
```

## Safety contract

```text
read-only      -> read/search only
read-write     -> read/search/edit, no shell
execute        -> read/search/edit/shell
worktree       -> writes isolated until parent applies
parent verify  -> canonical tests/checks in root context
```

## Design rule

Do not build separate systems for commands and agents. Make both tasks. Then orchestration becomes one scheduler plus one wait API.
