---
name: evaluate-agent
description: >
  Evaluate multi-step, tool-using AI agents by grading trajectories, individual
  steps, and outcomes separately. Use when evaluating agents that take actions
  over multiple turns, call tools, modify state, or interact with users in
  conversations. Do NOT use for single-turn prompt-response evaluation (use
  error-analysis and write-judge-prompt instead).
---

# Evaluate Agent

Evaluate AI agents that operate over multiple turns: calling tools, modifying state, and adapting based on intermediate results.

## Overview

1. Complete error analysis on agent traces first
2. Define what to evaluate: outcome, trajectory, or both
3. Build graders for each evaluation layer
4. Handle non-determinism with repeated trials
5. Separate capability evals from regression evals

## Prerequisites

- Error analysis on agent traces is complete (use error-analysis skill)
- Access to the agent's execution environment (or a sandboxed replica)
- Ability to run the agent programmatically and capture full traces

## Core Instructions

### Evaluate Outcome and Trajectory Separately

Agent evaluation has two layers:

**Outcome evaluation:** Did the agent achieve the goal? Check the final state — not the agent's claim about what it did. A flight-booking agent might say "Your flight has been booked" but the actual check is whether a reservation exists in the database.

**Trajectory evaluation:** Was the agent's path reasonable? An agent that arrives at the correct answer via 50 unnecessary tool calls is different from one that takes 3 efficient steps. Trajectory evaluation catches:
- Unnecessary or redundant tool calls
- Incorrect tool selection (used search when it should have used a calculator)
- Inefficient ordering (fetched data it already had)
- Missed opportunities (had the information but didn't act on it)

Evaluate both. A correct outcome via a bad trajectory is fragile — it may fail on harder cases.

### Grader Types

Combine three types of graders per task:

**Code-based graders (prefer these):**
- State verification: check database records, file system, API responses after the agent runs
- Tool call verification: check that required tools were called with correct parameters
- Output format checks: schema validation, required fields present
- Transcript metrics: turn count, token usage, tool call count, latency

**Model-based graders (when code cannot check):**
- Rubric-based scoring against specific criteria (one criterion per grader)
- Tool call justification: was there a good reason for each tool call?
- Communication quality: tone, clarity, helpfulness in user-facing messages
- Reasoning quality: did intermediate steps show sound logic?

**Human graders (for calibration):**
- Periodic review of transcripts to calibrate model-based graders
- Gold-standard labeling for ambiguous or novel scenarios
- Final authority when code and model graders disagree

### Grade What Was Produced, Not the Path

Avoid requiring agents to follow a specific sequence of tool calls. Agents find valid approaches that designers didn't anticipate. Check that the agent used required tools and achieved the correct state, not that it followed the exact expected sequence.

Exception: when the order matters for correctness (e.g., must verify identity before processing a refund), check ordering constraints explicitly.

### Partial Credit

Build in partial credit for tasks with multiple components. A support agent that correctly identifies the problem and verifies the customer but fails to process a refund is better than one that fails immediately.

```
graders:
  - type: state_check
    check: identity_verified
    weight: 0.3
  - type: state_check
    check: refund_processed
    weight: 0.5
  - type: llm_rubric
    criterion: communication_quality
    weight: 0.2
```

Binary scoring (all-or-nothing) works for simple tasks. Use weighted partial credit for complex multi-step tasks.

### Step-Level Evaluation

For complex agents, evaluate individual steps in addition to the overall trace:

**Tool call evaluation:** For each tool call, check:
- Was the tool appropriate for this step?
- Were the parameters correct?
- Was the result used appropriately in the next step?

**Decision point evaluation:** At each point where the agent chose between actions, was the choice reasonable? Build graders for critical decision points identified during error analysis.

**Retrieval decision evaluation (for agentic RAG):** Did the agent decide to retrieve at the right time? Grade should-retrieve vs. should-not-retrieve accuracy separately from retrieval quality.

### Handling Non-Determinism

Agents produce different trajectories on repeated runs. Run multiple trials per task.

**pass@k:** Probability of at least one success in k attempts. Use when one correct solution is sufficient (e.g., coding tasks where any working solution is acceptable).

```
pass@k = 1 - C(n-c, k) / C(n, k)
```
Where n = total trials, c = successful trials, k = attempts.

**pass^k:** Probability that all k trials succeed. Use when consistency matters (e.g., customer-facing agents where users expect reliable behavior every time).

```
pass^k = (per-trial success rate)^k
```

Run at least 3-5 trials per task. Report both pass@1 (single-attempt success) and pass^3 (consistency across 3 attempts). A task with 90% pass@1 but 73% pass^3 has reliability issues.

### Simulated User Interactions

Multi-turn conversational agents need a simulated user to test against.

**Building a user simulator:**
- Define a persona with a goal, constraints, and personality traits
- Provide private context the user knows but the agent doesn't (e.g., order details, preferences)
- Include a termination condition (max turns, goal achieved, or user gives up)

```
You are a customer named Alex who wants to return a laptop purchased
3 weeks ago. The order number is #12345. You are frustrated because
this is your second attempt — the first agent couldn't help. You want
a full refund, not store credit. If the agent asks you to verify your
identity, provide your email: alex@example.com. End the conversation
when you receive a refund confirmation or after 10 turns.
```

**Grading with simulated users:** Grade the outcome (was the refund processed?) and the interaction quality (was the user's frustration acknowledged?). The simulator's satisfaction is not a metric — grade against objective criteria.

### Environment Isolation

Each trial must start from a clean environment. Shared state between runs (leftover files, cached data, database records from previous trials) causes correlated failures and unreliable results.

- Reset databases, file systems, and external service state before each trial
- Use containers or snapshots for reproducible environments
- Verify that git history, caches, or logs from previous runs are not accessible

### Building the Evaluation Suite

**Task design:**
- Each task needs defined inputs, success criteria, and a reference solution that passes all graders
- If a task has 0% pass rate across many trials, suspect a broken task before suspecting an incapable agent
- Test both positive cases (agent should act) and negative cases (agent should not act). One-sided evals produce one-sided optimization.

**Capability vs. regression evals:**

| Type | Purpose | Expected Pass Rate | When to Run |
|------|---------|-------------------|-------------|
| Capability | What can the agent do? | Low, climbing over time | During development |
| Regression | Does the agent still work? | ~100% | On every change, in CI |

Capability evals that reach high pass rates graduate to the regression suite. Regression evals that start failing signal a broken change.

**Scaling the suite:**
- Start with 20-50 tasks drawn from real failures
- Add tasks from bug reports and user feedback
- Balance tasks across agent capabilities (tool use, reasoning, error handling, refusal)
- Track latency, token usage, and cost per task alongside correctness

## Anti-Patterns

- **Evaluating only the final output.** Check the environment state, not the agent's text claim.
- **Requiring a specific tool call sequence.** Agents find valid paths designers didn't anticipate. Grade outcomes and required tools, not exact sequences.
- **Single trial per task.** Agent behavior varies between runs. Use multiple trials with pass@k and pass^k.
- **Shared state between trials.** Leftover data from previous runs causes correlated failures.
- **One-sided test suites.** Test both when the agent should act and when it should refuse or do nothing.
- **Holistic model-based grading.** Use one grader per criterion. A single LLM grading "overall quality" produces unactionable results.
- **Ignoring trajectory efficiency.** A correct outcome via 50 redundant tool calls is fragile and expensive.
