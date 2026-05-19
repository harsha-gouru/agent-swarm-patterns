# Skill Optimization Loop

A new pattern observed in Warp's bundled `create-skill` skill. **An LLM iteratively rewrites a skill's own description (and optionally body), scored against a held-out test split of trigger queries.** Repeat until either everything passes, the held-out score stops improving, or a max-iteration budget is exhausted.

This is distinct from the `verifier-loop` pattern in this repo — that's a single second-pass verification. This is a *training loop* with split-prevention-of-overfitting.

## Shape

```text
                ┌───────────────────────────────────┐
                │ Generate ~20 trigger eval queries │
                │ • 8-10 should_trigger             │
                │ • 8-10 should_not_trigger         │
                │   (near-misses, not obvious)      │
                └───────────────┬───────────────────┘
                                ▼
                ┌───────────────────────────────────┐
                │ User reviews via HTML viewer      │
                │ assets/eval_review.html           │
                │ • edit queries                    │
                │ • toggle should_trigger           │
                │ • add/remove entries              │
                │ → exports eval_set.json           │
                └───────────────┬───────────────────┘
                                ▼
                ┌───────────────────────────────────┐
                │ Stratified 60/40 train/test split │
                │ random.seed(42)                   │
                │ • split should_trigger separately │
                │   from should_not_trigger         │
                └───────────────┬───────────────────┘
                                ▼
            ┌────────── iteration N (max 5) ─────────────────┐
            │                                                 │
            │   ┌──────────────────────────────────────────┐  │
            │   │ For each eval query:                     │  │
            │   │   run 3 times via `claude -p`            │  │
            │   │   compute trigger_rate                   │  │
            │   └──────────────┬───────────────────────────┘  │
            │                  ▼                              │
            │   ┌──────────────────────────────────────────┐  │
            │   │ Aggregate scores on TRAIN + TEST sets    │  │
            │   └──────────────┬───────────────────────────┘  │
            │                  ▼                              │
            │   ┌──────────────────────────────────────────┐  │
            │   │ Call `claude -p` to propose new          │  │
            │   │ description (sees current SKILL.md +     │  │
            │   │ failing eval results)                    │  │
            │   └──────────────┬───────────────────────────┘  │
            │                  ▼                              │
            │   ┌──────────────────────────────────────────┐  │
            │   │ Re-evaluate new description (TRAIN+TEST) │  │
            │   └──────────────┬───────────────────────────┘  │
            │                  ▼                              │
            │            Better than best so far?             │
            └─────────────────────────────────────────────────┘
                              │
                              ▼
                ┌───────────────────────────────────┐
                │ Return best_description           │
                │ • selected by TEST score          │
                │   (not train) to avoid overfit    │
                │ • report opens in HTML browser    │
                └───────────────────────────────────┘
```

## Use when

- The skill triggers too rarely (under-triggering) or too often (over-triggering on near-misses).
- You have ≥ 20 realistic trigger queries the user can vouch for.
- You can afford ~3 runs × ~20 queries × ~5 iterations × ~2 description variants = ~600 LLM calls. The script runs it in the background and reports per-iteration scores.
- You want a defensible "this description is provably better than the old one" claim.

## Prompt shape

```sh
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

The `--model` should match the model running the calling session, "so the triggering test matches what the user actually experiences" (verbatim from `create-skill/SKILL.md`).

## The eval JSON

```json
[
  {"query": "ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think", "should_trigger": true},
  {"query": "another realistic, specific prompt with file paths/personal context/typos", "should_trigger": false}
]
```

Verbatim guidance from `create-skill/SKILL.md` on writing good evals:

> The queries must be realistic and something a user would actually type. Not abstract requests, but requests that are concrete and specific and have a good amount of detail. For instance, file paths, personal context about the user's job or situation, column names and values, company names, URLs. A little bit of backstory. Some might be in lowercase or contain abbreviations or typos or casual speech.
>
> For the **should-not-trigger** queries, the most valuable ones are the near-misses — queries that share keywords or concepts with the skill but actually need something different. Think adjacent domains, ambiguous phrasing where a naive keyword match would trigger but shouldn't, and cases where the query touches on something the skill does but in a context where another tool is more appropriate.
>
> The key thing to avoid: don't make should-not-trigger queries obviously irrelevant. "Write a fibonacci function" as a negative test for a PDF skill is too easy — it doesn't test anything. The negative cases should be genuinely tricky.

## Guardrails

- **Stratified split.** `random.seed(42)`; split `should_trigger` items separately from `should_not_trigger`. Ensures both classes appear in train and test in proportion.
- **Held-out test selection.** `best_description = max(candidates, key=test_score)`, not by `train_score`. Prevents overfitting on the specific queries the LLM was rewriting against.
- **3 runs per query.** Trigger decisions are non-deterministic; one run is noise. Three gives a usable trigger rate (0/3, 1/3, 2/3, 3/3).
- **Same model end-to-end.** The model that judges trigger-eligibility must match the model that runs the agent. A description optimized for Sonnet might mis-trigger on Haiku.
- **HTML viewer for the eval set BEFORE the loop runs.** The user signs off on the queries first. Bad queries → bad descriptions; the loop can't fix the eval set.
- **Run length cap.** `--max-iterations 5` is the default. The loop stops earlier if a perfect train+test score is hit.
- **Background execution.** The loop is long-running (minutes to hours). `create-skill/SKILL.md` tells the agent to tail the output periodically to update the user instead of blocking.

## Failure modes

- **Eval set is unrealistic.** If the queries are obvious keyword matches, every description optimizes to a trivial regex. The "near-miss" rule for should-not-trigger queries is the explicit defense.
- **Reward hacking the trigger judge.** The same LLM judges whether a query should trigger; clever description prompts can manipulate it. Defended by held-out test set + multiple runs per query, but not eliminated.
- **Overtriggering vs undertriggering trade-off.** Verbatim from create-skill: *"agents have a tendency to 'undertrigger' skills — to not use them when they'd be useful. To combat this, please make the skill descriptions a little bit 'pushy'."* So the loop is biased toward pushiness. This can swing into the opposite failure (overtriggering on related-but-different tasks).
- **Train/test contamination at the model level.** If the same LLM has memorized the eval queries (e.g., from a prior session), the test set isn't truly held out. Defended only by using fresh queries.
- **Cost.** ~600 LLM calls per optimization run. The `--model` defaults to the calling session's model, which is usually the strongest one. Cheap models can be specified but reduce signal quality.

## Differences from existing patterns

- **vs `verifier-loop`** — verifier-loop is one second pass. This is a training loop with multiple iterations and held-out scoring.
- **vs `best-of-n-worktree`** — best-of-N picks the winner from parallel attempts on the *same* problem. This picks the winner across iterations that mutate the artifact.
- **vs `reviewer-swarm`** — reviewer-swarm gives one patch many specialized reviewers in parallel. This gives one skill many candidate descriptions in sequence.

## Runtime primitives needed

- A subprocess primitive (`claude -p` via `subprocess.run`) so the optimization can issue LLM calls outside the main agent loop without nesting context.
- A way to run with a model identifier specified externally (matches the calling session).
- A JSON eval format and a runner that returns per-query trigger booleans.
- A way to open an HTML viewer (or, in headless environments, write a static HTML file: `--static <output_path>`).
- File I/O for `feedback.json` (user comments) and `eval_set.json` (the eval queries after user edits).

## A blind comparison variant

`create-skill` also ships an **optional blind comparison system** for direct A-vs-B skill comparison (used when the user asks "is the new version actually better?"). Flow (from `agents/comparator.md` + `agents/analyzer.md`):

1. Spawn the eval task with skill A and skill B independently.
2. Give the two outputs to a **Blind Comparator** subagent labeled only A and B (no identity).
3. Comparator generates a rubric for the task and judges which output better satisfies it.
4. **Post-hoc Analyzer** subagent "unblinds" the result — reads both skills + both transcripts + the comparator's reasoning to identify *why* the winner won and how to improve the loser.

This is a structurally cleaner version of the same idea as the description-optimization loop, but for content comparison rather than triggering comparison.

## Source

`/Applications/Warp.app/Contents/Resources/bundled/skills/create-skill/`:

- `SKILL.md` — the meta-skill (456 lines)
- `scripts/run_loop.py` — the optimization loop (328 lines)
- `scripts/run_eval.py` — per-query evaluator (310 lines)
- `scripts/improve_description.py` — calls `claude -p` to propose a new description (246 lines)
- `scripts/aggregate_benchmark.py` — produces `benchmark.json` (401 lines)
- `scripts/generate_report.py` — HTML output (326 lines)
- `agents/grader.md`, `agents/comparator.md`, `agents/analyzer.md`
- `references/schemas.md`
- `assets/eval_review.html`

Apache 2.0 licensed. The pattern can be lifted directly into other skill-based agent systems without rewriting; you'd just substitute `claude -p` for whatever your eval-execution channel is.
