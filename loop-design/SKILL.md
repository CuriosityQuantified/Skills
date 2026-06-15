---
name: loop-design
description: "Use when designing a high-quality agent loop. Interviews the user to define the goal, restrictions, deterministic exit conditions, and guardrails before building. Prevents loopmaxxing. Outputs a structured loop spec."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [loop-engineering, agent-loops, architecture, planning, guardrails]
    related_skills: [plan, hermes-agent, subagent-driven-development]
---

# Loop Design

Design a high-quality agent loop with the user, not for them. This skill walks through a structured interview to extract the five critical dimensions of any loop — then outputs a loop spec you can build from.

## When to Use

- You're about to build an agent loop and need to avoid loopmaxxing
- The user says "I want an agent that runs autonomously to X"
- You need to convert a vague ambition ("do research overnight") into a bounded, verifiable loop
- You're designing a cronjob or scheduled agent task and need to define its exit conditions

**Don't use for:**
- One-shot prompts or single-turn agent calls (no loop needed)
- Simple automations with no iterative feedback (e.g., "send me weather every morning" — just use cron)
- Debugging an existing loop that's already broken (use systematic-debugging skill instead)

## The Interview Process

Walk through these five phases **in order**, one question at a time. Do not move to the next phase until the user has answered the current question. Each answer feeds the loop spec.

### Phase 1: Goal Definition

> **Purpose:** Turn vague intention into a falsifiable statement.

Ask the user these questions **one at a time** (wait for each answer):

1. **"What single, specific task should the loop accomplish? Describe the output in concrete terms — what file, state, or condition signals the loop is done."**

   *Good:* "Find all Python files with TODO comments, read their contents, and compile a list prioritized by file age."
   *Bad:* "Refactor the codebase to be better."

2. **"What does success look like? If the loop runs perfectly, what artifact, metric, or condition exists when it stops?"**

   *Examples:* A file written, all tests passing at 100%, a score >= 4.0, a PR opened, a specific exit code.

3. **"How will a human (or another agent) independently verify the loop succeeded without re-running it?"**

   *Examples:* Check the file, read the PR, re-run the tests, inspect the score log.

### Phase 2: Restrictions & Constraints

> **Purpose:** Bound the loop's search space and resource usage before it starts.

Ask these **one at a time**:

1. **"What is off-limits? Which files, directories, APIs, databases, or actions should the agent NEVER touch?"**

   *Examples:* `/etc/passwd`, production DB, the `secrets/` directory, DELETE endpoints, user accounts.

2. **"What is the maximum budget for one loop run?"**

   | Budget Type | How to Bound |
   |-------------|-------------|
   | Token budget | Max N tokens total (e.g., 500K) |
   | Cost ceiling | Max $X per run (e.g., $2.00) |
   | Time budget | Max N minutes wall-clock |
   | API calls | Max N external API calls |
   | Retries | Max N retries per step |

3. **"Are there any speed or latency constraints? How quickly does the loop need to complete?"**

   *Example:* "Must finish in under 30 seconds because it runs between user requests."

4. **"What context or data sources should the loop always have access to?"**

   *Examples:* A specific AGENTS.md file, MCP server, database dump, Slack channel.

### Phase 3: Deterministic Requirements

> **Purpose:** Define the hard checkpoints and exit conditions — the things that make this a *closed* loop, not open-ended drift.

Ask these **one at a time**:

1. **"What is the deterministic exit condition? What specific, checkable state means 'done'?"**

   | Good (deterministic) | Bad (fuzzy) |
   |---------------------|-------------|
   | `pytest --tb=short exit code 0` | "tests should pass" |
   | `git log --oneline | wc -l >= 5` commits | "make progress" |
   | Score from an external verifier >= 4/5 | "make it better" |
   | File exists at `/path/to/output.json` | "finish the work" |

2. **"Will there be a separate verifier checker? If so, what does it check?"**

   *Key rule:* The agent that does the work should NOT be the one that decides if it's done. Specify what the verifier checks and what score/condition satisfies "pass."

3. **"What is the maximum iteration count?"**

   *Rule of thumb:* For most loops, 3-5 iterations is enough. 10+ is rare. If you think you need 50+, the goal is too fuzzy.

4. **"What happens when the loop fails gracefully? What state should it leave behind?"**

   *Examples:* A log file with the last error, a checkpoint file, an @mention in a Slack channel, a GitHub Issue with the failure details.

### Phase 4: Guidelines & Guardrails

> **Purpose:** Define the operational policies the loop follows while running.

Ask these **one at a time**:

1. **"Human-in-the-loop or fully autonomous?"**

   - **Human-in-the-loop:** The loop pauses at defined checkpoints for human approval/rejection. Use when output is high-stakes or subjective.
   - **Fully autonomous:** Loop runs from trigger to completion with no human intervention. Use only when verification is fully deterministic.

2. **"How do we retry? What's the max retry count, and what's the backoff strategy?"**

   *Recommendation:* Max 2-3 retries. After that, fail gracefully to human. Use exponential backoff for API calls.

3. **"What trace-logging or observability do we need?"**

   *Minimum:* Log each turn with timestamp, action taken, result summary, and whether the metric improved. Push to a file that survives the session.

4. **"Are there any specific skills, tools, or MCP servers the loop should load?"**

   *Examples:* `hermes-agent` skill, your code review skill, a Linear MCP connector, a Slack webhook.

5. **"Should the loop have a 'circuit breaker' — a condition that aborts immediately regardless of state?"**

   *Examples:* A specific error message appears, token usage exceeds threshold, a critical API returns 500 twice in a row, the agent output matches a regex for "I'm stuck."

### Phase 5: Output a Loop Spec

After all questions are answered, synthesize into a structured loop spec. Format:

```markdown
## Loop Spec: <Name>

### Goal
    
**One-liner:** <sentence>
**Output artifact:** <what exists when done>
**Verification:** <how a human checks success>

### Boundaries

**Budget:** <tokens | cost | time max>
**Off-limits:** <paths, APIs, actions>
**Required context:** <skills, files, MCP servers>

### Exit Conditions

**Deterministic passes when:** <exact condition>
**Deterministic fails when:** <exact condition>
**Max iterations:** N
**Max retries per step:** N
**Graceful failure produces:** <log, checkpoint, notification>

### Architecture

**Loop type:** <fully autonomous | human-in-the-loop>
**Maker agent:** <what it does>
**Checker agent:** <what it checks>
**Verification mechanism:** <how success is independently verified>

### Guardrails

**Retry strategy:** <count + backoff>
**Circuit breaker:** <condition that aborts>
**Observability:** <log file, format, key events>
**Human handoff:** <when and how control returns to human>

### Schedule (if recurring)

**Cadence:** <cron or interval>
**Trigger:** <event, cron, manual>
**Delivery:** <where the result goes>
```

Save this spec to `.hermes/plans/` or the location the user specifies.

## Loop Architecture Patterns

These are common loop shapes. Reference them during the interview if the user needs inspiration.

### Pattern A: The Verification Loop

```
Goal → Agent does work → External verifier scores output → Score above threshold? → Yes: done. No: Agent retries (up to N times).
```

**Best for:** Code review, content generation, SEO, any task with an automated scoring system.

**Example:** "I want an agent that writes blog posts until they score 85+ on readability."

### Pattern B: The Self-Improving Experiment Loop

```
Agent reads state → Hypothesizes change → Applies change → Runs evaluation → Keeps or reverts (git) → Repeats until budget exhausted.
```

**Best for:** ML hyperparameter search, code optimization, benchmarking.

**Example:** Karpathy's AutoResearch — 5-min experiments, BPB metric, git revert on failure.

### Pattern C: The Triage-and-Notify Loop

```
Check a source → Find items meeting criteria → Compile report → Send to human → Clear processed items → Wait → Repeat.
```

**Best for:** CI triage, social listening, monitoring, inbox management.

**Example:** "Check GitHub Actions every hour, summarize failed workflows, DM me the report."

### Pattern D: The Pipeline Loop

```
Step 1 agent runs → Writes output → Step 2 agent reads output → Runs → Writes output → ... → Final step delivers.
```

**Best for:** Multi-stage processing where each stage's output is the next stage's input.

**Example:** Research pipeline: scrape → extract entities → classify → summarize → save to Notion.

### Pattern E: The Human-in-the-Loop Gate

```
Agent drafts → Human reviews → Approve? → Yes: ship. No: human edits notes → Agent revises → Human reviews again (up to N cycles).
```

**Best for:** High-stakes content, code that ships to production, any output with subjective quality.

**Example:** "Agent drafts email campaigns, I approve each before send."

## Common Pitfalls

1. **Skipping Phase 2 (restrictions).** The most expensive loopmaxxing failures come from unbounded scope. If you haven't explicitly defined what's off-limits and what the budget is, the loop will find those limits expensively.

2. **Fuzzy exit conditions.** "Make it better" is not an exit condition. If you can't write `if condition: stop()` in pseudocode, the condition isn't ready.

3. **Self-verification.** The same agent that does the work should not grade it. A separate checker or external tool catches mistakes the maker agent cannot see. This is the single most important rule in loop design.

4. **No graceful failure path.** Every loop should produce a useful artifact even when it fails — a log, a partial result, a clear error message. A loop that goes silent on failure is a loop that will be trusted long after it breaks.

5. **Over-engineering before running.** Start with a minimal loop with human oversight. Run it a few times. Then add automation. The interview gives you the spec; the implementation should be iterative.

6. **Setting max iterations too high.** 3-5 iterations is usually enough. Each additional iteration compounds cost. If your loop consistently needs 10+ iterations, the goal is too broad or the context is too thin.

7. **No cost ceiling for long-running loops.** A fast loop that burns $0.10 per iteration can hit $100 if left running all night. Always set a token budget or cost ceiling.

## Verification Checklist

Before declaring the loop spec complete, verify:

- [ ] **Goal is falsifiable** — you can point to a specific state that proves "done"
- [ ] **Goal passes the "GPS test"** — a human could verify success without re-running the loop
- [ ] **Off-limits are documented** — paths, APIs, actions the agent should never touch
- [ ] **Budget is bounded** — tokens, cost ceiling, or time limit
- [ ] **Exit condition is deterministic** — binary pass/fail check, not subjective
- [ ] **Separate verifier defined** — maker and checker are different agents or systems
- [ ] **Max iterations set** — hard cap, typically 3-5
- [ ] **Retry limit set** — max 2-3 retries per step before human handoff
- [ ] **Graceful failure produces an artifact** — log, notification, partial result
- [ ] **Trace-logging specified** — what events are logged and where
- [ ] **Circuit breaker defined** — abort condition for emergency stop
- **Human handoff documented** — when and how control returns to a person

## One-Shot Recipes

### Quick Interview (no time for full process)

When the user is impatient, ask the **four essential questions**:

1. **"What is the single concrete output the loop produces?"**
2. **"How does the loop know it's done — what exact condition?"**
3. **"What's the maximum it should spend (tokens, time, dollars)?"**
4. **"Who or what checks the work — a separate agent, a test suite, or you?"**

Then build the loop spec from those four answers.

### Converting a cron idea to a loop

User says: "I want a daily agent that checks X and reports Y."

Interview: "A daily check of X is a pure cron — no iterative feedback needed. To turn it into a loop, we need a condition where the agent iterates on Y until it's good enough. Do you want iteration, or just a single check-and-report?"
