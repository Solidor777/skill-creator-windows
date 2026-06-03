# skill-creator-windows

A **Windows-compatible** copy of the description-optimization scripts from
Anthropic's official `skill-creator` Claude Code plugin. The upstream scripts
assume a Unix environment and crash on Windows; this copy patches them so the
trigger-optimization loop runs.

> Derived from the `skill-creator` plugin's `scripts/` package. Only the
> portability fixes below were changed; the optimization logic is upstream's.

## What it does

Given a skill and a set of "should-trigger / should-not-trigger" queries, it
tests how reliably Claude invokes the skill from its `description`, then
iteratively proposes and re-scores improved descriptions (train/held-out split
to avoid overfitting). Use it to tune a skill's `description` for accurate
triggering.

## Why this fork exists — the Windows fixes

1. **`select()` on pipes → `WinError 10038`.** `run_eval.py` read the
   `claude -p` subprocess with `select.select()`, which on Windows only works on
   sockets, not pipes. Replaced with a blocking line reader plus a
   `threading.Timer` watchdog that kills the process on timeout.
2. **cp1252 `UnicodeEncodeError`.** The HTML report contains `✗`, which Windows'
   default text encoding can't encode. Run with `PYTHONUTF8=1` (see below) so
   `write_text` uses UTF-8.
3. **Command-vs-skill mismatch.** Upstream registered the candidate as a slash
   *command*, but current Claude Code only autonomously triggers *skills*, so
   every should-trigger query scored 0. The candidate is now written as a
   temporary **skill** so the trigger test is faithful to how skills actually
   fire.
4. **User-scope skills shadowed the candidate → still all-zero.** Writing the
   candidate into a throwaway *project* root does **not** hide user-scope skills:
   `claude -p` loads `~/.claude/skills` (or `$CLAUDE_CONFIG_DIR/skills`)
   regardless of cwd. If the skill under test is already installed there (the
   normal case on a dev machine), the real skill competes and wins — the model
   fires `Skill(skill="<real-name>")`, never the candidate's unique uuid name, so
   detection scored 0 for every query. **Fix:** each query now runs with its own
   isolated `CLAUDE_CONFIG_DIR` containing **only** the candidate skill (with
   `.credentials.json`/`settings.json` copied so auth survives), plus
   `stdin=DEVNULL` to skip a 3-second-per-query stdin wait. The candidate becomes
   the sole skill in scope, so the model can only fire it and the score reflects
   the **candidate** description. Verified: a skill that scored all-0 before now
   scores 22/22 with correctly discriminating positives/negatives.

## Requirements

- Python 3.10+
- The `claude` CLI on PATH (it shells out to `claude -p`)

## Usage

Run from the repo root (so the `scripts` package resolves):

```bash
PYTHONUTF8=1 python -m scripts.run_loop \
  --eval-set examples/example-trigger-evals.json \
  --skill-path /path/to/your/skill \
  --model claude-opus-4-7 \
  --max-iterations 5 \
  --runs-per-query 3 \
  --num-workers 10 \
  --verbose \
  --results-dir ./results \
  --report ./report.html
```

`--skill-path` points at a skill directory containing `SKILL.md`. The loop reads
its current `description`, scores it, and (if anything fails) proposes
improvements. The winning description is printed as `best_description` in the
JSON output and written into the HTML report.

### Eval-set format

A JSON array of queries, each flagged with whether the skill *should* trigger:

```json
[
  {"query": "who actually destroyed clae again?", "should_trigger": true},
  {"query": "summarize the grappling rules in D&D 5e", "should_trigger": false}
]
```

Make the should-not-trigger cases genuinely tricky near-misses (shared keywords,
adjacent domains) — those are what sharpen the description. See
`examples/example-trigger-evals.json`.

## Notes

- Other upstream scripts (`aggregate_benchmark.py`, `package_skill.py`, etc.) are
  included unchanged for completeness.
- Re-pull from the upstream plugin if it ships a native Windows fix; this fork is
  a stopgap.
