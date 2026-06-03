# Trigger eval sets — format & workflow

A **trigger eval set** measures whether a skill's `description` causes Claude to
fire the skill (via the `Skill` tool) on the queries where it *should*, and to
stay quiet on the near-misses where it *shouldn't*. It's the input to both the
baseline scorer (`run_eval.py`) and the description-improvement loop
(`run_loop.py`).

## File format

A flat JSON array of objects, each with exactly two keys:

```json
[
  {"query": "the user prompt, written the way a real user would type it", "should_trigger": true},
  {"query": "a near-miss that shares keywords but needs a different tool",  "should_trigger": false}
]
```

- `query` — a realistic user message. Make it **substantive** (multi-step or
  specialized). Trivial one-step asks like "read file X" won't trigger *any*
  skill regardless of description quality, so they're useless as test cases.
- `should_trigger` — `true` if this query *should* load the skill, `false` if not.

A good set is **20 queries, ~half and half** (8–10 each).

## Designing the queries

**Positives (`true`)** — vary the surface form of the same intent: formal and
casual phrasings, cases where the user never names the skill or file type but
clearly needs it, a few uncommon use cases, and cases where this skill competes
with another but should win.

**Negatives (`false`)** — the valuable ones are **near-misses**: queries that
share keywords or concepts with the skill but actually need something else
(adjacent domains, ambiguous phrasing a naive keyword match would trip on, a
context where another tool fits better). Avoid obviously-irrelevant negatives
("write a fibonacci function" against a PDF skill) — they test nothing.

See `examples/trigger-eval.example.json` for a worked set (a PDF-extraction
skill) showing both halves, and the skill-creator `SKILL.md` for the fuller
query-design philosophy.

## Running it

Set UTF-8 first (Windows): `$env:PYTHONUTF8 = "1"` (PowerShell) or prefix
`PYTHONUTF8=1` (bash).

**Baseline — score one description as-is (no rewriting):**

```bash
python -m scripts.run_eval \
  --eval-set examples/trigger-eval.example.json \
  --skill-path <path-to-skill-dir> \
  --runs-per-query 3 --num-workers 10 --verbose
```

Use this to check a description you wrote by hand, or to confirm an edit didn't
regress. It reads the skill's current description (override with `--description`),
runs each query `--runs-per-query` times, and prints per-query
`trigger_rate` + `pass` plus a summary `passed/total`. Nothing is modified.

**Improvement loop — propose and test better descriptions:**

```bash
python -m scripts.run_loop \
  --eval-set examples/trigger-eval.example.json \
  --skill-path <path-to-skill-dir> \
  --model <model-id> --max-iterations 5 --runs-per-query 3 --num-workers 10 --verbose
```

This splits the set 60/40 train/test, scores the current description, asks Claude
to propose improvements based on what failed, and iterates (up to
`--max-iterations`), selecting `best_description` by **test** score to avoid
overfitting. Use the model ID powering your current session so the test matches
what users experience.

## Reading results

- `trigger_rate` — fraction of runs that fired the skill (e.g. `2/3` = 0.67).
- `pass` — whether the rate landed on the correct side of `--trigger-threshold`
  (default 0.5) given `should_trigger`.
- Broad, abstract skills are inherently noisier run-to-run on their borderline
  positives; the most reliable signal is that **near-miss negatives stay at 0**
  (no over-triggering) and clear positives stay high. A description change that
  keeps negatives clean while lifting a weak positive is a real win.

## Note on isolation

`run_eval.py` runs each query in an isolated `CLAUDE_CONFIG_DIR` containing only
the candidate skill, so an already-installed copy of the same skill on your
machine can't shadow and beat the candidate (which otherwise makes every query
score 0). Auth carries over via copied credentials, or set `ANTHROPIC_API_KEY`.
