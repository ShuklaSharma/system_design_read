# Reflection / Self-Critique Pattern for AI Applications

> AI Engineering Series | May 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Reflection Works](#why-reflection-works)
3. [How the Pattern Works](#how-the-pattern-works)
4. [Variants of Reflection](#variants-of-reflection)
5. [Building a Reflection Loop — Step by Step](#building-a-reflection-loop)
6. [The Critic — Designing the Judge](#the-critic--designing-the-judge)
7. [Termination & Convergence](#termination--convergence)
8. [AI-Specific Use Cases](#ai-specific-use-cases)
9. [Advanced Patterns](#advanced-patterns)
10. [Integration with Other Patterns](#integration-with-other-patterns)
11. [Monitoring & Observability](#monitoring--observability)
12. [Production Best Practices](#production-best-practices)
13. [Common Mistakes](#common-mistakes)
14. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is Reflection?

The **Reflection Pattern** wraps a generation step in a critique-and-revise loop:

```
generate → critique → (if not good enough) revise → critique → ... → final
```

A first model produces a candidate answer. A critic — the same model, a different model, or a deterministic checker — evaluates that answer against criteria, returns structured feedback, and the generator revises. Loop until the critic accepts or a budget is exhausted.

**Origin:** Self-Refine (Madaan et al., 2023), Reflexion (Shinn et al., 2023). The empirical finding: even when the same model both generates and critiques, the second pass meaningfully improves output on tasks where errors are easier to *spot* than to *avoid*.

**Analogy:** A writer drafts, then re-reads with an editor's eye. Re-reading catches typos and weak arguments that "first-draft mode" missed even though the same brain wrote both.

### Why It Belongs as a Pattern

LLMs make systematic mistakes — hallucinated facts, missed constraints in long prompts, off-by-one errors in code, unsafe outputs. A second pass that *only* looks for those mistakes catches a large fraction of them. The cost is one extra LLM call; the benefit is materially higher quality.

---

## Why Reflection Works

### Generation vs Verification Asymmetry

For many tasks, **verifying** an answer is easier than **producing** one:

```
Task: "Is 1,238,409 prime?"
─────────────────────────────────────────────────────
Generation: hard (try every divisor up to √N)
Verification: also hard, but easier patterns to check (parity, last digit, mod 3, etc.)

Task: "Write code that meets these 12 requirements"
─────────────────────────────────────────────────────
Generation: must hold all 12 in mind while writing
Verification: walk through the code with the requirement list, one at a time
```

Reflection exploits this asymmetry. Even if the critic is the same model, the prompt is now *focused on a single sub-question* (correctness of THIS code against THESE requirements) instead of doing everything at once.

### Empirical Lift

| Task family | Reflection lift over single-shot |
|---|---|
| Code generation (HumanEval+) | +6 to +12 pp pass rate |
| Multi-constraint instruction following | +10 to +25 pp constraint-satisfaction |
| Math word problems | +4 to +9 pp |
| Long-form factuality | -30 to -50% hallucination rate |
| Creative writing | mixed; mostly cosmetic improvements |

Reflection helps when there are **objective things to check**. It helps less when "good" is subjective.

---

## How the Pattern Works

### The Core Loop

```
                       ┌────────────────┐
   user task ─────────►│   GENERATOR    │── candidate ──┐
                       └────────────────┘                │
                                                         ▼
                                                 ┌────────────────┐
                                                 │     CRITIC     │
                                                 │ checks against │
                                                 │   criteria     │
                                                 └───┬─────────┬──┘
                                                     │         │
                                              "accept"     "issues: [...]"
                                                     │         │
                                                     ▼         ▼
                                                 ┌──────┐  ┌──────────────┐
                                                 │ done │  │ feed back to │
                                                 └──────┘  │  generator   │
                                                           └──────┬───────┘
                                                                  │
                                                          loop ◄──┘
```

### Four Roles to Separate in Your Mind

1. **Task** — what the user actually asked for.
2. **Criteria** — measurable standards the answer must meet (correctness, format, safety, completeness).
3. **Generator** — produces / revises the candidate.
4. **Critic** — checks the candidate against the criteria; emits structured feedback or "accept."

The pattern collapses if criteria are vague. Spend most of your design time making criteria concrete.

---

## Variants of Reflection

### 1. Self-Refine (Same Model, Both Roles)

```
generator: claude-sonnet-4-6
critic:    claude-sonnet-4-6   ← same model, different prompt
```

Cheapest variant. Works surprisingly well because the critic prompt narrows attention.

### 2. Stronger-Critic / Weaker-Generator

```
generator: claude-haiku-4-5    ← cheap, fast
critic:    claude-sonnet-4-6   ← smarter, catches more
```

Good cost/quality trade-off when the generator's job is repetitive (templated outputs) and the critic's is rare (only fires on suspicious outputs).

### 3. Tool-Verified Critic

```
generator: LLM
critic:    deterministic — runs tests, executes code, validates schema
```

The strongest variant for code, structured data, or anything testable. The critic is not an LLM; it's a runtime check. LLMs can't argue with a failed test.

### 4. Reflexion (Memory Across Attempts)

```
attempt 1 fails → critic writes a "lesson" → stored in scratchpad
attempt 2: generator sees the lesson + retries
```

Lessons accumulate across attempts. Useful for agentic tasks where the same error shape recurs.

### 5. Multi-Agent Debate

```
generator A produces answer → generator B critiques → A revises → ...
```

Two adversarial LLMs argue. Higher cost; can yield better calibration on contested factual claims.

---

## Building a Reflection Loop

### Minimal Implementation (Anthropic SDK)

```python
import anthropic
import json

client = anthropic.Anthropic()
MODEL = "claude-sonnet-4-6"
MAX_ITERATIONS = 3

def generate(task: str, prior_attempt: str = None, feedback: str = None) -> str:
    user = task if prior_attempt is None else f"""\
Task:
{task}

Your previous attempt:
{prior_attempt}

Critic feedback:
{feedback}

Produce a revised answer that addresses every point of feedback."""

    msg = client.messages.create(
        model=MODEL,
        max_tokens=2048,
        system="You are an expert. Produce careful, correct work.",
        messages=[{"role": "user", "content": user}],
    )
    return msg.content[0].text

CRITIC_SYSTEM = """You are a strict reviewer. Evaluate the candidate answer against \
the task. Return ONLY JSON with this schema:

{
  "accept": boolean,
  "issues": [
    {"severity": "blocker|major|minor", "description": "..."}
  ],
  "suggestions": "concrete revision instructions, or empty string"
}

Set "accept": true only if there are no blocker or major issues."""

def critique(task: str, candidate: str) -> dict:
    msg = client.messages.create(
        model=MODEL,
        max_tokens=1024,
        system=CRITIC_SYSTEM,
        messages=[
            {"role": "user", "content": f"TASK:\n{task}\n\nCANDIDATE:\n{candidate}"}
        ],
    )
    return json.loads(msg.content[0].text)

def reflect(task: str) -> dict:
    candidate = generate(task)
    history = []
    for i in range(MAX_ITERATIONS):
        verdict = critique(task, candidate)
        history.append({"iteration": i, "candidate": candidate, "verdict": verdict})
        if verdict["accept"]:
            return {"answer": candidate, "iterations": i + 1, "history": history}
        candidate = generate(
            task,
            prior_attempt=candidate,
            feedback=verdict["suggestions"],
        )
    return {"answer": candidate, "iterations": MAX_ITERATIONS,
            "history": history, "stopped": "budget"}
```

### What Makes This Work

- The critic returns **structured JSON**, not prose. The loop has a deterministic accept/reject signal.
- The revised generation prompt **carries forward** both the prior attempt and the feedback. Without this, the model re-tries from scratch and may repeat the same mistake.
- A **hard iteration cap** prevents runaway loops.

---

## The Critic — Designing the Judge

The critic is where most reflection systems fail. A vague critic accepts everything; a maximalist critic rejects everything. Two principles:

### 1. Make Criteria Explicit and Atomic

Bad critic prompt:

```
"Check if this is good."
```

Good critic prompt:

```
Evaluate the candidate against these specific criteria:
1. SCHEMA: Output is valid JSON matching the schema in <schema>.
2. COMPLETENESS: Every field listed in <required_fields> is present and non-null.
3. GROUNDING: Every factual claim is supported by <sources>. List unsupported claims.
4. SAFETY: Output contains no PII (emails, phones, full names).
5. STYLE: Tone is formal, sentences are < 25 words.

For each criterion, output PASS or FAIL with a one-line reason.
```

The critic now has five separable, near-binary judgements instead of one fuzzy one.

### 2. Force Structured Output

JSON / typed verdicts let your loop branch programmatically. Free-form prose makes the loop unreliable.

### 3. Calibrate the Critic Independently

Before deploying, run the critic on a labelled dataset of (good, bad) pairs. Measure:

- **Precision**: when critic says "fail," is it really a failure? (avoid over-rejecting)
- **Recall**: when an answer truly fails, does the critic catch it? (avoid rubber-stamping)

Tune the prompt — or swap in a stronger model — until both are above 85%.

### 4. Use a Deterministic Critic When You Can

If the criterion is "the code passes these unit tests," **don't use an LLM**. Run the tests. The same logic applies to:

- JSON-schema validation
- SQL parsing / dry-run
- Regex/format checks
- Numeric range checks
- Length / token count
- Static analysis tools (linters, type checkers)

LLMs hallucinate verdicts. Deterministic checks don't.

---

## Termination & Convergence

### Hard Budget

Always cap iterations (typical: 3–5). Past 5, returns diminish sharply and cost compounds.

### Detecting Stagnation

If iteration N's candidate hash equals iteration N-1's, the model has converged on its best (possibly-wrong) answer. Break the loop with a "stagnation" status.

```python
import hashlib

def fingerprint(s: str) -> str:
    return hashlib.sha256(s.strip().encode()).hexdigest()[:16]

# inside the loop
if fingerprint(candidate) == fingerprint(history[-1]["candidate"]):
    return {"answer": candidate, "stopped": "stagnation", ...}
```

### Quality Floors

Sometimes the loop "improves" the answer but never quite passes. Decide in advance: do you ship the best-so-far, or fail the request?

```python
if not verdict["accept"] and i == MAX_ITERATIONS - 1:
    if has_only_minor_issues(verdict):
        return {"answer": candidate, "stopped": "budget", "quality": "acceptable"}
    else:
        return {"error": "could_not_meet_criteria", "history": history}
```

### Cost Budget

Track tokens per attempt. If a single task has consumed $X, abort even before the iteration cap.

---

## AI-Specific Use Cases

### 1. Code Generation with Test-Verified Critic

The strongest fit. The critic is the test runner.

```
generate code → run tests → fail? → feed failures back → regenerate
```

We routinely see pass-rate jumps from 70% → 90%+ on standard code-gen benchmarks with this loop.

### 2. Structured-Output Extraction

Generate JSON → validate against schema → if invalid, feed validator errors back. Two iterations almost always achieves 100% schema compliance, vs ~95% single-shot.

### 3. Long-Form Factuality

Generator writes the answer with citations. Critic checks each claim against the source documents. Issues become "claim X is not supported by source Y."

### 4. Compliance / Safety Review

Generator drafts a response. Critic checks against a policy doc (e.g., "must not give medical diagnosis," "must not name competitors"). Material reduction in policy violations.

### 5. SQL / Query Generation

Generator writes the SQL. Critic runs `EXPLAIN` (deterministic) to check syntax + table existence, then optionally runs on a test slice to check semantics.

### 6. Translation Quality

Generator translates. Back-translation critic translates back to source language. Critic compares semantic distance to original.

---

## Advanced Patterns

### Reflexion with Persistent Memory

Across **multiple tasks**, persist successful "lessons" the critic produced, and inject the most-relevant ones into future attempts. The system gets better over time without retraining.

```python
def select_relevant_lessons(task, lesson_store, top_k=3):
    # vector search lesson_store by task embedding
    return lesson_store.knn(embed(task), top_k)
```

### Critic-Driven Sampling

Instead of one generator pass, sample N candidates. Critic ranks them. Pick the top-1 (no revise loop).

```
generator: temperature=0.7, n=5  → 5 candidates
critic:    rank → return best
```

Trades latency (parallel calls) for not needing a revise step. Good for code where running tests on 5 candidates and picking the one that passes is cheap.

### Hierarchical Reflection

For long outputs, reflect *per section* before reflecting on the whole.

```
draft outline → critique outline
                    ↓
draft each section → critique section
                    ↓
assemble → critique whole
```

Catches errors locally before they get buried in 10K tokens.

### Constitutional Self-Critique

The critic checks against a written "constitution" (Anthropic's terminology) — a list of principles the response must satisfy. The constitution is editable policy, decoupled from the agent code.

---

## Integration with Other Patterns

### + Plan-and-Execute

Apply reflection at two levels:

```
plan → critique plan → revise plan
                ↓
execute steps → critique each step output
                ↓
synthesise → critique synthesis
```

Reflection on the plan catches strategy errors before any tool is called — far cheaper than catching them mid-execution.

### + RAG

Reflection helps RAG hallucinate less. Critic checks each generated claim against the retrieved chunks; unsupported claims trigger a regenerate or a "I don't know" response.

### + Tool Use / ReAct

Inside an agent loop, run reflection on each completed sub-task before committing to the next. Costly but invaluable for irreversible actions (sending email, writing files).

### + Prompt Caching

The critic prompt is highly stable — perfect cache target. Mark the criteria as a 1h TTL cached block; only the candidate-under-review changes between calls.

### + Semantic Cache

Cache the *(task, accepted_final_answer)* pair, not intermediate iterations. Subsequent semantically similar tasks skip the entire reflection loop.

### + Cost Guardrails

Reflection can 3–5× per-task cost. Always wire a token-budget circuit breaker that aborts the loop when a per-request budget is exceeded.

---

## Monitoring & Observability

### Required Metrics

| Metric | What to Watch |
|---|---|
| `iterations_per_task` (p50/p95/max) | Right-sized iteration budget |
| `accept_rate_at_iteration_N` | Where the loop converges |
| `tokens_per_task` | Cost reality check |
| `critic_disagreement_rate` | Critic flagging vs human-labelled good outputs |
| `stagnation_rate` | Tasks bailing out unchanged |
| `quality_lift_vs_single_shot` | Is reflection actually helping? |

### A/B Test Reflection

Before defaulting to reflection, run shadow traffic:

```
50% of requests: single-shot
50% of requests: reflection
→ measure quality (human eval or automated metric) and cost
→ decision: ship if (Δquality / Δcost) > threshold
```

Many teams discover reflection helps on Tier-1 traffic only. Apply selectively.

### Structured Trace

Log the full chain so you can audit why a final answer was accepted:

```json
{
  "task": "...",
  "iterations": [
    {"i": 0, "candidate": "...", "verdict": {"accept": false, "issues": [...]}},
    {"i": 1, "candidate": "...", "verdict": {"accept": true}}
  ],
  "total_tokens": 4321,
  "total_cost_usd": 0.034,
  "duration_ms": 6800
}
```

---

## Production Best Practices

### 1. Cap Iterations Aggressively

Default 3, raise to 5 only if the data justifies it.

### 2. Prefer Deterministic Critics

If a check can be a regex, schema validator, or test, make it one. Use the LLM critic for things only an LLM can judge.

### 3. Cache the Critic Prompt

Stable criteria are cheap to cache. Saves ~30% on critic costs.

### 4. Don't Reflect on Everything

Trivial tasks ("classify this short message") don't need a second pass — the cost doubles, the lift is near zero. Apply reflection where the failure cost is high or the task is multi-constraint.

### 5. Make the Critic Stronger Than the Generator (Sometimes)

Stronger critic + cheap generator can outperform a single Sonnet/Opus call at lower cost.

### 6. Persist Critic Verdicts for Eval

The critic's pass/fail history is an automatic dataset for offline evaluation.

### 7. Beware Self-Aggrandising Loops

A model critiquing its own work can develop a positive-bias drift over revisions ("looks great now!"). Calibrate the critic against a held-out human-judged set quarterly.

### 8. Time-Box User-Facing Use

Reflection doubles or triples latency. For interactive UX, either show progressive output or limit to background tasks.

---

## Common Mistakes

1. **Vague critic** — accepts everything, loop terminates instantly with bad output.
2. **No iteration cap** — runaway cost on pathological tasks.
3. **Free-form critic output** — loop can't reliably parse "should I revise?"
4. **Forgetting to feed feedback forward** — model retries from scratch and repeats the same error.
5. **Reflecting where verification is harder than generation** — pure cost, no benefit.
6. **Single critic on every task** — calibrate critics per-task-type.
7. **No stagnation detection** — loop spins burning tokens producing the same answer.
8. **Letting the critic edit the candidate directly** — turns into a single-pass system, you've lost the separation that makes reflection work.

---

## Interview Cheat Sheet

**Q: What is the Reflection pattern?**
A: A generate → critique → revise loop. A first model produces a candidate; a critic (LLM, model, or deterministic check) evaluates it against criteria; if not accepted, feedback is fed back to the generator. Repeat until accept or budget exhausted.

**Q: When does reflection actually help?**
A: When verification is easier than generation — code with tests, structured output with schema, factual claims with citations, multi-constraint instructions. Less helpful for subjective creative tasks.

**Q: Same model for generator and critic, or different?**
A: Same model often suffices because the critic prompt narrows attention. Different models (cheap generator + strong critic) can win on cost. Deterministic critics (test runner, schema validator) win whenever applicable.

**Q: How do you prevent infinite loops?**
A: Hard iteration cap (3–5), stagnation detection (same fingerprint across iterations), and per-task token budget.

**Q: Reflection vs Plan-and-Execute?**
A: Orthogonal. Plan-and-Execute splits *strategy from execution*; Reflection adds a *quality loop* around any single generation step. They compose — reflect on the plan, then reflect on the execution.

**Q: What's Reflexion?**
A: A reflection variant that persists "lessons" from failures across attempts, accumulating in an episodic memory the agent reads on the next try. Particularly effective for tool-using agents that hit similar errors repeatedly.

**Q: Cost trade-off?**
A: 2–5× per-task cost depending on iteration count. Justified when failure cost is high or quality lift is measurable. Always A/B test before defaulting to reflection.

---

**Document Version:** 1.0
**Last Updated:** May 2026
