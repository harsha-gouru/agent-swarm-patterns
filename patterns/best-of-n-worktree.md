# Best-of-N Worktree Swarm

## Shape

```text
                ┌──────────────┐
                │ Parent Agent │
                │ reads tests  │
                └──────┬───────┘
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Candidate 1 │  │ Candidate 2 │  │ Candidate 3 │
│ worktree A  │  │ worktree B  │  │ worktree C  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       └───────────────┼────────────────┘
                       ▼
              ┌────────────────┐
              │ Parent tests   │
              │ judge winner   │
              └───────┬────────┘
                      ▼
              ┌────────────────┐
              │ Apply winner   │
              └────────────────┘
```

## Use when

- Correctness matters more than latency.
- Multiple implementation strategies are plausible.
- The repo has tests or objective checks.

## Prompt shape

```text
Absolute repo: /path/to/repo
Allowed write set: /path/to/repo/src/foo.py only
Run: pytest path/to/test_foo.py
Use isolation=worktree for every candidate.
Candidate 1: simple direct implementation.
Candidate 2: stdlib/library implementation.
Candidate 3: defensive/manual implementation.
Return changed files, test output, and tradeoffs.
```

## Guardrails

- Worktree isolation is mandatory for file writes.
- Parent runs canonical tests in the main repo before applying a winner.
- Parent compares diffs, not just child prose.

## Failure modes

- Children edit the wrong cwd if prompts use relative paths.
- Candidates pass their invented tests but fail canonical tests.
- Parent applies a winner without re-running tests.
