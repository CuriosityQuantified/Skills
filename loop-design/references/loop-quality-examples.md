# Loop Quality Examples — 1-to-5 Grading Scale

A reference catalog of agent loops graded on a 1–5 scale. Use these to calibrate during the loop-design interview — show the user examples so they can say "I want something like the 4, not the 2."

---

## Grading Rubric

| Score | Label | Characteristics |
|-------|-------|----------------|
| **5** | **Exemplary** | Deterministic exit, separate verifier, bounded budgets, graceful failure, full observability, circuit breaker. Ready for autonomous deployment. |
| **4** | **Solid** | Clear exit condition, budgeted, basic logging. May lack separate verifier or circuit breaker. Needs light human oversight. |
| **3** | **Adequate** | Defined goal and exit but fuzzy on bounds or logging. Human must watch it. Will work but wastes tokens. |
| **2** | **Weak** | Vague goal, no real exit condition, no budget. Drifts. Needs redesign before use. |
| **1** | **Broken** | No exit condition, self-verifying, unbounded. Guaranteed to loopmaxx or produce garbage. Do not deploy. |

---

## Score 1 — Broken

### 1A. The Feedback Scraper That Loops Forever

```
Goal: "Scrape all user feedback from the last year."
Exit: None — it just keeps going.
Verification: The agent itself decides when it's "done."
Budget: None.
```

**Why it fails:** The agent has no definition of completeness. It scrapes, finds more pages, scrapes those, spirals. Without a deterministic exit (e.g., "no new records found across 3 consecutive pages"), it will run until token limit, API credit exhaustion, or a human kills the process. Self-verifying — the agent that scrapes also decides when to stop.

**What it produces:** Infinite token burn, duplicate records, and a confused human getting paged at 3 AM.

### 1B. The "Improve the Codebase" Loop

```
Goal: "Make the code better."
Exit: "When it's good enough."
Verification: Agent runs its own linting and declares victory.
Budget: Unlimited tokens.
```

**Why it fails:** "Good enough" is not an exit condition. The agent will refactor, re-refactor, undo, redo, add unnecessary abstractions, and eventually hit a context window. It declares success because *it* evaluates its own work. No human can verify "good enough" without re-running the entire thing.

---

## Score 2 — Weak

### 2A. The Blog Generator With a Page Count

```
Goal: "Write 10 blog posts about SaaS metrics."
Exit: "When we have 10 blog posts."
Verification: Agent counts files in the output directory.
Budget: None set.
Off-limits: None defined.
```

**Why it's a 2:** Counting files is a deterministic exit — that's the one good thing here. But there's no quality gate, no budget, no off-limits. The agent produces 10 posts that are all shallow, repetitive, or wrong. It exits "successfully" with garbage. A human has to read and reject everything. The loop technically stops, but the result is useless.

**What fixes it to a 3:** Add a quality gate (readability score >= 60, fact-check pass), budget prompt tokens, and a retry limit per post.

### 2B. The Notion Sync That Never Finishes

```
Goal: "Sync all my Notion databases to local markdown."
Exit: "When all databases are synced."
Verification: Agent checks its own todo list.
Budget: 1 hour wall-clock.
Off-limits: None.
```

**Why it's a 2:** "All databases" sounds deterministic but isn't — the agent discovers databases it didn't know about, syncs those, discovers more, repeats. No database enumeration step before starting. Self-verifying on a mutable todo list. The 1-hour budget is the only guardrail, which means it just crashes at 60 minutes with partial data and no checkpoint.

**What fixes it to a 4:** Enumerate all database IDs upfront, then sync is a bounded set. Write a checkpoint file after each database. Exit when `completed == total`. Log which ones succeeded and which failed.

---

## Score 3 — Adequate

### 3A. The PR Review Commenter

```
Goal: "Read open PRs, review code, leave comments on PRs that have been open >24h."
Exit: "All qualifying PRs reviewed or skipped."
Verification: A reviewer re-checks 2 random PRs manually.
Budget: 300K tokens, max 15 API calls.
Off-limits: PRs with WIP in the title, PRs already reviewed.
Retries: 1 retry on API failure.
Logging: Writes a summary to `review-log.md`.
```

**Why it's a 3:** The goal is concrete and the exit is deterministic. Budgets are set. Off-limits are defined. Logging exists. But — the agent is self-verifying (it decides "reviewed or skipped" without a separate checker). The manual spot-check is an afterthought, not a system. Quality of reviews varies because there's no review-quality gate. Graceful failure isn't defined — if it crashes mid-way, the log is incomplete with no way to resume.

**What fixes it to a 4:** Add a review-quality score (comment specificity check), a checkpoint file for resumability, and a notification on failure.

### 3B. The Daily Security Scan

```
Goal: "Scan our AWS environment for public S3 buckets once per day."
Exit: "All regions scanned. Report written to `security-report.json`."
Verification: Human reads the report.
Budget: 100 API calls, 10 minutes.
Off-limits: No write operations, no IAM changes, no production resources.
Retries: 2 retries with 5s backoff per region.
Logging: Report file + stdout timestamps.
```

**Why it's a 3:** Well-bounded scope. Clear off-limits (read-only). Retry strategy. Budget is reasonable for scanning 20+ regions. The human reads the report for verification. But — no graceful failure path (what if region us-east-1 times out? Does it skip or crash?), no circuit breaker (if all regions return 403, keep going or abort?), no separate verifier (the human reading the report isn't an automated check). The report format is undefined, so quality varies run to run.

---

## Score 4 — Solid

### 4A. The Research Synthesizer

```
Goal: "Research topic X across 5 sources, compile a structured brief with citations."
Exit: "Brief saved to `output/<topic>.md` with all 5 sources cited and a confidence score >= 3/5."
Verification: External checker agent scores the brief on (coverage, citation accuracy, structure). Pass if all 3 dimensions >= 3/5.
Budget: 500K tokens, 50 API calls, $0.50 cost ceiling.
Off-limits: No social media, no paywalled content, no user data.
Max iterations: 3 (draft → revise → final pass).
Max retries per step: 2.
Circuit breaker: If 3 consecutive API calls return 429 or 500, abort.
Graceful failure: Saves partial brief with error log and checkpoint state.
Logging: Full turn log with timestamps, scores per iteration, cost-per-turn to `research-log.json`.
Human handoff: If fails after max retries, notify user with partial brief and failure reason.
```

**Why it's a 4:** Nearly complete. Deterministic exit (score threshold), separate verifier checker agent, budgets on all dimensions, circuit breaker, graceful failure with artifact, full logging, human handoff. The only thing keeping it from a 5 is that the verifier is another agent rather than a deterministic program — agent-as-verifier introduces its own failure modes (hallucinated scores, inconsistent grading).

### 4B. The A/B Test Analyzer

```
Goal: "Analyze last 7 days of A/B test data, identify statistically significant winners (p < 0.05), write recommendations."
Exit: "Report saved to `ab-reports/<date>.md`. Must contain: test names, sample sizes, p-values, effect sizes, and a recommendation for each (ship / iterate / kill)."
Verification: A script checks the report has all required sections and p-values are computed correctly. Human reads the recommendations.
Budget: 200K tokens, 30 API calls, 15 minutes.
Off-limits: No raw user PII, no experiment config changes, no production DB writes.
Max iterations: 2 (draft → refine).
Retries: 1 per computation step.
Circuit breaker: If data retrieval returns empty set, abort — no analysis on no data.
Graceful failure: Saves partial analysis with error detail.
Logging: Computation steps, p-values per test, errors logged to `ab-<date>.log`.
```

**Why it's a 4:** Strong verification (script validates report structure + computed values). Clear off-limits. Circuit breaker for empty data prevents pointless runs. Graceful failure path. Bumped to a 5 by adding: (a) a cost ceiling in dollars, (b) a human-in-the-loop gate before any "ship" recommendation is acted on, and (c) a deterministic checker script (not just a format check — re-run the computation independently and assert p-values within tolerance).

---

## Score 5 — Exemplary

### 5A. The AutoResearch Experiment Loop

(Loosely inspired by Karpathy's approach)

```
Goal: "Given a research question, design and run a 5-minute experiment, report the result."
Exit: "Experiment log saved to `experiments/<run-id>.md` with: hypothesis, method, raw results, BPB metric, and a go/no-go recommendation."
Verification: A deterministic script runs the evaluation pipeline independently and compares against the agent's reported metric. Must match within 1% tolerance.
Budget: 300K tokens, 20 API calls, $0.30 cost ceiling, 5 minutes wall-clock.
Off-limits: No internet writes, no files outside `experiments/`, no training jobs, no DELETE operations.
Max iterations: 1 (one experiment per trigger — no self-repeating).
Max retries per step: 2 with exponential backoff (2s, 4s).
Circuit breaker: If any tool call returns an exception, log and abort immediately. Do not retry past the first retry pair.
Graceful failure: Saves partial experiment log with the error, the state at failure, and a "REQUIRES HUMAN" flag.
Logging: Full turn log to `experiments/<run-id>.trace.json`. Includes: each tool call, its result, token cost, wall-clock time.
Human handoff: On any circuit breaker or retry exhaustion, DM the user with a link to the partial log. Include the research question and the failure reason.
Schedule: Triggered by cron (every 4 hours), but each tick runs exactly one experiment — no internal looping.
```

**Why it's a 5:** Every dimension is covered:
- **Exit:** One-shot experiment per tick — bounded by design, not by chance
- **Verification:** Independent deterministic script re-evaluates the metric
- **Budget:** Tokens, API calls, cost, and wall-clock — all bounded
- **Off-limits:** Explicit and scoped
- **Circuit breaker:** Catches tool failures instantly, no compounding
- **Graceful failure:** Always produces an artifact, even on crash
- **Observability:** Full trace log per run
- **Human handoff:** On any failure mode
- **No self-verification:** The agent runs the experiment; a script validates the result

### 5B. The CI Failure Triage Pipeline

```
Goal: "Every hour, check GitHub Actions for failed workflow runs on main. For each failure, extract the failing test/step, search the repo for recent commits touching that code, and write a triage report."
Exit: "Triage report saved to `ci-reports/<date>.md`. One entry per failure. Each entry contains: failed job, failing test/step, error excerpt (first 10 lines), list of recent commits touching the failing file (last 5), and a confidence assessment: HIGH (clear culprit) / MEDIUM (possible) / LOW (unclear)."
Verification: A separate checker agent reads the report and validates each entry:
  - Error excerpt matches the raw CI log (string match)
  - Listed commits actually touch the implicated file (git log check)
  - No hallucinated test names (must match known test suite output)
  If any entry fails validation, the report is rejected and the pipeline re-runs (up to 1 retry).
Budget: 400K tokens, 40 API calls, $0.60 cost ceiling.
Off-limits: No workflow re-runs, no deployment changes, no Secrets/Config access, no GitHub admin APIs.
Max iterations: 1 pass per cron tick (no re-triaging old failures).
Max retries per step: 2 with exponential backoff.
Circuit breaker: If GitHub API returns 403 (rate limit), abort and notify. If all workflows pass (no failures to triage), exit silently with a brief log entry — do not generate an empty report.
Graceful failure: Saves partial report with what was completed and what failed. Includes raw error data for manual triage.
Logging: Full trace to `ci-reports/<date>.trace.json`. Logs: API calls made, rate limit remaining, failures found, per-entry validation pass/fail.
Human handoff: If checker rejects the report after retry, or if circuit breaker fires, post a summary to Slack #ci-alerts with the partial report link.
Schedule: Cron `0 * * * *` (every hour on the hour).
```

**Why it's a 5:**
- **Dual verification:** Checker agent validates each entry against ground truth (git log, CI log), not against the maker agent's claims
- **Silent success on no-op:** If nothing is broken, the loop exits without noise — no false urgency
- **Graceful partial output:** Even a failed run produces a salvageable artifact
- **Slack handoff on failure:** Human gets alerted with context, not just "it broke"
- **Self-contained budget:** Every dimension capped so it can't runaway even if something goes wrong
- **Zero write risk:** Off-limits explicitly blocks destructive actions

---

## Quick Reference Card

```
Score 5 — Deterministic exit + separate verifier + full budgets + circuit breaker
           + graceful failure + observability + human handoff
           → Safe to deploy autonomously.

Score 4 — Clear exit + budgets + logging. May lack separate verifier or breaker.
           → Needs light oversight, close to autonomous.

Score 3 — Defined goal/exit but fuzzy on bounds. Human must watch.
           → Works but wastes tokens. Tighten budgets and logging.

Score 2 — Vague goal or no real exit. Will drift.
           → Don't use as-is. Redesign the exit condition first.

Score 1 — No exit, self-verifying, unbounded.
           → Will loopmaxx. Do not deploy in any form.
```

---

*Use this reference during the Phase 1 (Goal Definition) and Phase 3 (Deterministic Requirements) interviews. Show the user examples at different scores and ask: "Where on this scale does your loop need to land?"*
