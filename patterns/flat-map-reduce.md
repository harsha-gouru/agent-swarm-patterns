# Flat Map-Reduce Swarm

## Shape

```text
                ┌──────────────┐
                │ Parent Agent │
                └──────┬───────┘
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
 ┌──────────┐    ┌──────────┐    ┌──────────┐
 │ Child A  │    │ Child B  │    │ Child C  │
 │ item 1   │    │ item 2   │    │ item 3   │
 └────┬─────┘    └────┬─────┘    └────┬─────┘
      └───────────────┼───────────────┘
                      ▼
              ┌──────────────┐
              │ Parent merge │
              └──────────────┘
```

## Use when

- The items are independent: files, URLs, repos, logs, tickets, papers.
- Each child can produce a bounded summary.
- Parent can merge summaries into one answer.

## Prompt shape

```text
Spawn one read-only child per item.
Each child inspects exactly one absolute path/item.
Each child returns JSON with: item_id, findings, confidence, evidence_paths.
Parent waits for all children, then merges and de-duplicates.
```

## Guardrails

- Give each child one item only.
- Make output schema strict.
- Parent verifies representative evidence before trusting the merged result.

## Failure modes

- Children drift into neighboring files if paths are relative.
- Parent over-trusts child summaries.
- Summaries lose important evidence unless required fields include paths/snippets.
