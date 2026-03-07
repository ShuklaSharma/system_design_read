# Chain-of-Thought (CoT) Pattern

> Prompt Engineering Series — Pattern 01 of 03 | AI Engineering | March 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why CoT Works](#why-cot-works)
3. [CoT Variants](#cot-variants)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Combining with Other Patterns](#combining-with-other-patterns)
8. [Production Best Practices](#production-best-practices)
9. [Common Mistakes](#common-mistakes)
10. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is Chain-of-Thought Prompting?

**Chain-of-Thought (CoT)** is a prompting technique that instructs a language model to show its reasoning process step by step before delivering a final answer. Rather than jumping straight to a conclusion, the model externalises its intermediate thinking — making it transparent, auditable, and significantly more accurate.

**Origin:** Introduced by Wei et al. (Google Brain, 2022) — they discovered that simply adding *"Let's think step by step"* to a prompt dramatically improved performance on reasoning benchmarks.

```
Without CoT:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Q: Roger has 5 tennis balls. He buys 2 more cans of 3 balls each.
   How many tennis balls does he have now?

A: 11   ✅  (got lucky — model compressed reasoning)


With CoT:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Q: Roger has 5 tennis balls. He buys 2 more cans of 3 balls each.
   How many tennis balls does he have now? Let's think step by step.

A: Roger starts with 5 tennis balls.
   He buys 2 cans × 3 balls = 6 new balls.
   Total = 5 + 6 = 11 tennis balls.   ✅  (reliable, auditable)
```

### The Problem Without CoT

```
Scenario: Complex multi-step reasoning

Without CoT:
─────────────────────────────────────────
User asks a complex reasoning question
Model predicts answer in a single step
Model skips key intermediate logic
Answer is wrong or inconsistently correct
No way to audit where it went wrong

Result:
- Unreliable answers on hard problems
- No visibility into model reasoning
- Cannot identify or fix the failure point
- Low user trust in complex domains
```

### The Solution With CoT

```
With CoT:
─────────────────────────────────────────
User asks a complex reasoning question
Model externalises reasoning step by step
Each step is visible and verifiable
Model self-corrects errors mid-chain
Final answer is grounded in shown logic

Result:
- Reliable answers on complex problems
- Full reasoning audit trail
- Easy to identify and fix failure points
- High user trust in complex domains
```

---

## Why CoT Works

### The Core Mechanism

LLMs are next-token predictors. Without CoT, the model tries to compress all reasoning into the probability distribution of a single output token — like asking someone to solve a maths problem in their head without writing anything down.

CoT forces the model to write its working, which:

1. **Increases the computation budget** — each reasoning step is a new set of attention operations
2. **Reduces the cognitive load per token** — each step only needs to advance reasoning by one hop
3. **Enables self-correction** — the model can "read" its own prior steps and catch errors
4. **Grounds later steps in earlier ones** — builds a reliable chain rather than a single leap

### Performance Gains (Research Benchmarks)

| Task Type | Without CoT | With CoT | Improvement |
|---|---|---|---|
| GSM8K (grade school math) | 17.9% | 56.4% | +38.5pp |
| MATH (competition math) | 4.2% | 18.0% | +13.8pp |
| StrategyQA (reasoning) | 64.8% | 73.0% | +8.2pp |
| Date Understanding | 49.3% | 78.5% | +29.2pp |

> **Key Insight:** CoT gains are larger for harder problems. On simple tasks it adds little value — on complex tasks it is transformative.

---

## CoT Variants

### 1. Zero-Shot CoT

The simplest form — no examples needed. Just append a reasoning trigger.

```text
# Trigger phrases (ranked by effectiveness)
"Let's think step by step."                    ← most effective
"Let's work through this carefully."
"Think step by step before answering."
"Walk me through your reasoning."
"Let's break this down."
```

**Best for:** Quick wins on tasks where you don't have example reasoning chains ready.

---

### 2. Few-Shot CoT

Provide complete worked examples including the reasoning chain. More reliable than zero-shot for production.

```text
Q: The cafeteria had 23 apples. If they used 20 to make lunch and
   bought 6 more, how many apples do they have?
A: They started with 23 apples.
   Used 20: 23 - 20 = 3 apples left.
   Bought 6 more: 3 + 6 = 9 apples.
   Answer: 9

Q: Shawn has 5 toys. For Christmas he got 2 toys each from his mom and dad.
   How many toys does he have now?
A: Shawn started with 5 toys.
   Got 2 from mom and 2 from dad = 4 new toys.
   Total: 5 + 4 = 9 toys.
   Answer: 9

Q: There were 9 computers in the server room. 5 more were installed each day
   for 3 days. How many computers are there now?
A:
```

---

### 3. Self-Consistency CoT

Sample multiple independent reasoning paths and take the majority vote. Significantly improves accuracy over single-pass CoT.

```python
import anthropic
from collections import Counter

client = anthropic.Anthropic()

def self_consistent_cot(question: str, num_samples: int = 5) -> str:
    """
    Generate multiple reasoning chains and vote on the final answer.
    Reduces variance and improves accuracy on hard reasoning tasks.
    """
    answers = []

    for i in range(num_samples):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=512,
            messages=[{
                "role": "user",
                "content": f"{question}\n\nLet's think step by step. "
                           f"At the end, state your final answer clearly as 'Answer: X'."
            }],
            # Slight temperature variation encourages diverse reasoning paths
            # (use temperature parameter if needed for your model version)
        )

        text = response.content[0].text

        # Extract final answer
        if "Answer:" in text:
            answer = text.split("Answer:")[-1].strip().split("\n")[0]
            answers.append(answer)

    # Majority vote
    if not answers:
        return "Could not determine answer"

    vote_counts = Counter(answers)
    majority_answer, count = vote_counts.most_common(1)[0]

    return {
        "answer": majority_answer,
        "confidence": count / len(answers),
        "all_answers": dict(vote_counts)
    }

# Usage
result = self_consistent_cot(
    "If a store sells 3 items at $12 each and gives a 10% discount on the total, "
    "what is the final price?",
    num_samples=5
)
print(f"Answer: {result['answer']} (confidence: {result['confidence']:.0%})")
```

**Performance boost:** Self-consistency improves CoT accuracy by 10–20pp on hard benchmarks.

---

### 4. Tree-of-Thought (ToT)

Explore multiple reasoning branches simultaneously. Use when the problem has many possible solution paths.

```python
def tree_of_thought(problem: str, branches: int = 3) -> str:
    """
    Generate multiple reasoning branches and evaluate which leads
    to the best solution.
    """

    # Step 1: Generate multiple reasoning starting points
    branch_prompt = f"""
Problem: {problem}

Generate {branches} different approaches to solve this problem.
For each approach, write:
APPROACH [N]: [brief name]
REASONING: [first 2-3 steps of this approach]
PROMISING: [rate 1-10 and explain why]
"""

    branches_response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": branch_prompt}]
    )

    # Step 2: Select the most promising branch and complete it
    completion_prompt = f"""
Problem: {problem}

Here are several reasoning approaches that were considered:
{branches_response.content[0].text}

Select the most promising approach and complete the full reasoning chain
to arrive at a final answer.
"""

    final_response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": completion_prompt}]
    )

    return final_response.content[0].text
```

---

### 5. Program-of-Thought (PoT)

For mathematical tasks, have the model write code instead of prose reasoning — then execute it.

```python
def program_of_thought(math_problem: str) -> dict:
    """
    Generate Python code to solve the problem, then execute it.
    More accurate than prose reasoning for numerical tasks.
    """

    code_prompt = f"""
Solve this problem by writing Python code.
Only output the Python code — no explanation.

Problem: {math_problem}

Requirements:
- Store the final answer in a variable called `answer`
- Print the answer at the end
- Use clear variable names
"""

    # Generate code
    code_response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=512,
        messages=[{"role": "user", "content": code_prompt}]
    )

    code = code_response.content[0].text.strip()
    # Strip markdown fences if present
    code = code.replace("```python", "").replace("```", "").strip()

    # Execute safely
    try:
        namespace = {}
        exec(code, namespace)
        answer = namespace.get("answer", "Not found")
        return {"code": code, "answer": answer, "success": True}
    except Exception as e:
        return {"code": code, "error": str(e), "success": False}


# Example
result = program_of_thought(
    "A train travels at 80km/h for 2.5 hours, then at 60km/h for 1.5 hours. "
    "What is the total distance covered?"
)
print(f"Answer: {result['answer']}")
```

---

### Variant Comparison

| Variant | Accuracy | Cost | Speed | Best For |
|---|---|---|---|---|
| Zero-Shot CoT | ⭐⭐⭐ | $ | Fast | Quick wins, prototyping |
| Few-Shot CoT | ⭐⭐⭐⭐ | $$ | Medium | Production reasoning tasks |
| Self-Consistency | ⭐⭐⭐⭐⭐ | $$$ | Slow | High-stakes decisions |
| Tree-of-Thought | ⭐⭐⭐⭐⭐ | $$$$ | Slowest | Creative problem solving |
| Program-of-Thought | ⭐⭐⭐⭐⭐ | $$ | Medium | Mathematical computation |

---

## Implementation Patterns

### Pattern 1: Structured Reasoning Schema

Enforce a consistent reasoning format for production reliability:

```python
STRUCTURED_COT_SYSTEM = """
You are an expert analyst. For every problem you solve, follow this exact format:

PROBLEM RESTATEMENT:
[Restate the problem in your own words]

KNOWN INFORMATION:
[List all given facts and values]

REASONING STEPS:
Step 1: [First reasoning step]
Step 2: [Next step, building on previous]
Step 3: [Continue as needed]
...

VERIFICATION:
[Check if your answer is reasonable. Does it make sense?]

FINAL ANSWER:
[State the answer clearly]
"""

def structured_cot(question: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system=STRUCTURED_COT_SYSTEM,
        messages=[{"role": "user", "content": question}]
    )
    return response.content[0].text
```

---

### Pattern 2: Domain-Specific CoT

Tailor the reasoning schema to your domain:

```python
# Medical reasoning CoT
MEDICAL_COT_SYSTEM = """
When analysing medical cases, reason as follows:

SYMPTOMS ANALYSIS:
[List and analyse each symptom]

DIFFERENTIAL DIAGNOSIS:
[List possible diagnoses, most to least likely]

SUPPORTING EVIDENCE:
[Evidence for and against each diagnosis]

RED FLAGS:
[Any symptoms requiring urgent attention]

RECOMMENDED ACTION:
[Suggested next steps — always recommend consulting a doctor]

Note: Always remind users to seek professional medical advice.
"""

# Legal reasoning CoT
LEGAL_COT_SYSTEM = """
When analysing legal questions:

LEGAL ISSUE:
[Identify the core legal question]

APPLICABLE LAW:
[Relevant statutes, regulations, or precedents]

ANALYSIS:
[Apply the law to the specific facts]

COUNTER-ARGUMENTS:
[Strongest arguments against your position]

CONCLUSION:
[Reasoned conclusion with confidence level]

Note: Always remind users to consult a qualified attorney.
"""
```

---

### Pattern 3: CoT with Output Parsing

Parse structured CoT output for downstream use:

```python
import re

def cot_with_parsing(question: str) -> dict:
    """Generate CoT response and extract structured components."""

    response = structured_cot(question)

    # Parse sections
    sections = {}
    patterns = {
        "restatement":  r"PROBLEM RESTATEMENT:\n(.*?)(?=\n[A-Z]|\Z)",
        "known":        r"KNOWN INFORMATION:\n(.*?)(?=\n[A-Z]|\Z)",
        "steps":        r"REASONING STEPS:\n(.*?)(?=\n[A-Z]|\Z)",
        "verification": r"VERIFICATION:\n(.*?)(?=\n[A-Z]|\Z)",
        "answer":       r"FINAL ANSWER:\n(.*?)(?=\n[A-Z]|\Z)",
    }

    for key, pattern in patterns.items():
        match = re.search(pattern, response, re.DOTALL)
        sections[key] = match.group(1).strip() if match else ""

    return {
        "full_response": response,
        "parsed": sections,
        "answer": sections.get("answer", "")
    }

# Usage
result = cot_with_parsing(
    "A company's revenue grew from $2M to $3.5M. What was the percentage growth?"
)
print(f"Answer: {result['parsed']['answer']}")
print(f"Verification: {result['parsed']['verification']}")
```

---

## AI-Specific Use Cases

### Use Case 1: Code Debugging

```text
SYSTEM: You are an expert software engineer. When debugging code:
  1. Trace the execution line by line
  2. Identify the exact line where the bug occurs
  3. Explain why it is a bug
  4. Provide the corrected code
  5. Explain what the fix does

USER:
def calculate_average(numbers):
    total = 0
    for num in numbers:
        total += num
    return total / len(numbers)

result = calculate_average([])
print(result)

This code throws an error. What's wrong and how do I fix it?

ASSISTANT:
Step 1: Trace execution
  - calculate_average([]) is called with an empty list
  - total = 0 (initialised correctly)
  - for loop: numbers is empty, so loop body never executes
  - return total / len(numbers) → return 0 / 0

Step 2: Identify the bug
  - Line: return total / len(numbers)
  - len([]) = 0 → division by zero → ZeroDivisionError

Step 3: Why it's a bug
  - No guard against empty input
  - Python raises ZeroDivisionError when dividing by 0

Step 4 & 5: Fix
def calculate_average(numbers):
    if not numbers:          # Guard: return None for empty list
        return None
    total = sum(numbers)
    return total / len(numbers)

The fix adds an early return for empty lists, preventing division by zero.
```

---

### Use Case 2: RAG Answer Quality

Use CoT to improve answer quality in Retrieval-Augmented Generation:

```python
RAG_COT_SYSTEM = """
You are a helpful assistant. When answering from retrieved documents:

DOCUMENT ANALYSIS:
[Summarise what each retrieved document says about the question]

RELEVANCE ASSESSMENT:
[Rate how relevant each document is — High / Medium / Low]

SYNTHESIS:
[Combine the most relevant information]

GAPS:
[What the documents don't cover that would help answer the question]

ANSWER:
[Final answer grounded only in the retrieved documents]

CONFIDENCE:
[High / Medium / Low — based on document quality and coverage]
"""

def rag_with_cot(question: str, retrieved_docs: list) -> str:
    context = "\n\n".join([
        f"Document {i+1}:\n{doc}" for i, doc in enumerate(retrieved_docs)
    ])

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1500,
        system=RAG_COT_SYSTEM,
        messages=[{
            "role": "user",
            "content": f"Retrieved Documents:\n{context}\n\nQuestion: {question}"
        }]
    )
    return response.content[0].text
```

---

## Advanced Patterns

### Pattern: Metacognitive CoT

Ask the model to reason about the quality of its own reasoning:

```text
SYSTEM:
After completing your reasoning, add a METACOGNITION section:

METACOGNITION:
- Confidence: [0-100%] in your final answer
- Weakest step: [Which step in your reasoning is least certain?]
- Assumptions made: [What did you assume that might be wrong?]
- Alternative interpretations: [Could the question mean something else?]
```

### Pattern: Stepwise Verification

Have the model verify each step before proceeding:

```text
After each reasoning step, ask:
- Is this step logically valid?
- Does it follow from the previous step?
- Are there any errors in this step?

Only proceed to the next step if the current one passes verification.
```

---

## Combining with Other Patterns

### CoT + Few-Shot

Provide complete worked examples that include reasoning chains:

```text
# Few-Shot CoT: examples include full reasoning

Q: Janet's ducks lay 16 eggs per day. She eats 3 for breakfast and bakes
   muffins using 4. She sells the rest for $2 each. How much daily income?
A: Janet's ducks lay 16 eggs per day.
   She uses 3 (breakfast) + 4 (muffins) = 7 eggs.
   Remaining: 16 - 7 = 9 eggs to sell.
   Income: 9 × $2 = $18 per day.
   Answer: $18

Q: [new question]
A:
```

### CoT + ReAct

Use CoT within each reasoning step of a ReAct agent:

```text
Thought: Let me think step by step about what tool to use.
  Step 1: The question asks for today's stock price — this is live data.
  Step 2: My training data has a cutoff — I cannot know current prices.
  Step 3: I should use the get_stock_price tool.
  Therefore: I will call get_stock_price(AAPL).
Action: get_stock_price(AAPL)
```

---

## Production Best Practices

### Do's

- Always use structured output formats (PROBLEM / STEPS / ANSWER) for consistency
- For high-stakes decisions, use Self-Consistency (5+ samples) to reduce variance
- Cache CoT prompts — the system prompt doesn't change, only user input does
- Log the full reasoning chain for debugging and quality monitoring
- Use Program-of-Thought for any purely mathematical computation

### Don'ts

- Don't use CoT for simple factual lookups — wasteful token spend
- Don't mix CoT schemas between requests — consistency is key
- Don't truncate the reasoning output — you lose the verification step
- Don't assume CoT eliminates hallucinations — it reduces but doesn't eliminate them

### Token Cost Awareness

```
Simple factual Q&A:     ~50  output tokens
Zero-Shot CoT:          ~150-300 output tokens  (3-6x increase)
Structured CoT:         ~300-500 output tokens  (6-10x increase)
Self-Consistency (5x):  ~750-1500 output tokens (15-30x increase)

Rule of thumb:
- Only use CoT when the accuracy improvement justifies the cost
- Self-Consistency only for decisions with significant downstream impact
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| Using CoT for simple tasks | Wastes tokens, adds latency | Only use for 3+ step reasoning |
| No structured schema | Inconsistent output format | Define a fixed reasoning template |
| Ignoring the reasoning | Only using the final answer | Read steps to catch errors early |
| Single sample on hard problems | High variance answers | Use Self-Consistency |
| Mixing CoT styles | Model gets confused | Standardise schema per use case |
| Too short reasoning budget | Model rushes to answer | Set `max_tokens` generously |

---

## Interview Cheat Sheet

**Q: What is Chain-of-Thought prompting?**
A: A technique where you instruct the LLM to show its intermediate reasoning steps before giving a final answer. Introduced by Wei et al. (2022). The phrase "Let's think step by step" is the minimal trigger.

**Q: Why does CoT improve accuracy?**
A: LLMs are next-token predictors. CoT increases the effective computation budget — each reasoning step is a new set of attention operations. It also enables self-correction and prevents the model from taking "shortcuts".

**Q: When would you NOT use CoT?**
A: Simple factual lookups, classification with clear patterns, latency-sensitive applications, or when token cost is a primary constraint.

**Q: What is Self-Consistency CoT?**
A: Generate multiple independent reasoning chains (typically 5–20), then take the majority vote on the final answer. Improves accuracy by 10–20pp on hard benchmarks at the cost of more API calls.

**Q: How does CoT differ from fine-tuning?**
A: CoT is a prompting technique — no model weights change. Fine-tuning bakes reasoning patterns into the model. CoT is faster to iterate, fine-tuning is more cost-efficient at scale.

**Q: How do you use CoT in production?**
A: Define a structured reasoning schema (Problem → Known → Steps → Verify → Answer), version the prompt, log reasoning chains, and use Self-Consistency for high-stakes decisions.

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
*Next: [Few-Shot Prompting Guide](02-few-shot-pattern-guide.md) →*
