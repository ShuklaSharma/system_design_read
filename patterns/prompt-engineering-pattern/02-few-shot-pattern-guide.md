# Few-Shot Prompting Pattern

> Prompt Engineering Series — Pattern 02 of 03 | AI Engineering | March 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Few-Shot Works](#why-few-shot-works)
3. [Shot Types Compared](#shot-types-compared)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Combining with Other Patterns](#combining-with-other-patterns)
8. [Production Best Practices](#production-best-practices)
9. [Common Mistakes](#common-mistakes)
10. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is Few-Shot Prompting?

**Few-Shot prompting** is the technique of including one or more worked input-output examples (called "shots") directly within the prompt. The model learns the desired pattern, format, tone, and logic from these examples and applies it to new inputs — with zero parameter updates and no fine-tuning.

It exploits the model's **in-context learning** ability: the capacity to adapt behaviour purely from examples seen in the prompt window.

```
Without Few-Shot:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Prompt: Classify this review sentiment: "Shipping was fast but the item broke immediately."

Output: Mixed   ← vague, inconsistent across calls


With Few-Shot:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Review: "Absolutely loved it!"  → Sentiment: POSITIVE
Review: "Waste of money."       → Sentiment: NEGATIVE
Review: "Arrived on time."      → Sentiment: NEUTRAL

Review: "Shipping was fast but the item broke immediately."
→ Sentiment: NEGATIVE  ← consistent, label-locked
```

### The Problem Without Few-Shot

```
Scenario: Enforcing a strict JSON output format

Without Few-Shot:
─────────────────────────────────────────────
Request 1  → {"name": "Alice", "age": 30}
Request 2  → Here is the data: name=Bob, age=25
Request 3  → {"name":"Carol","age":28,"extra":"field"}
Request 4  → I'll format this as JSON: { "name" : "Dave" }

Result:
- Inconsistent output format
- Downstream parser breaks
- Extra/missing fields
- Unreliable in production
```

### The Solution With Few-Shot

```
With 2-3 examples showing the exact format:
─────────────────────────────────────────────
Request 1  → {"name": "Alice", "age": 30}
Request 2  → {"name": "Bob", "age": 25}
Request 3  → {"name": "Carol", "age": 28}
Request 4  → {"name": "Dave", "age": 31}

Result:
- Perfectly consistent format
- Parser never breaks
- Correct fields every time
- Production-ready output
```

---

## Why Few-Shot Works

### In-Context Learning

LLMs learn from examples provided in the context window. Each example shifts the model's probability distribution toward outputs that match the demonstrated pattern. The model is essentially doing "implicit fine-tuning" within a single forward pass.

Key factors that make it work:

1. **Format anchoring** — Examples lock the output to a specific structure
2. **Label space definition** — Examples show the model what valid outputs look like
3. **Tone and style transfer** — The model mimics the register shown in examples
4. **Edge case handling** — Examples demonstrate how to treat unusual inputs
5. **Domain calibration** — Examples ground the model in your specific terminology

### Why It Outperforms Instruction-Only Prompts

```
Instruction only:
  "Always respond in JSON with fields: name, age, city"
  → Model interprets "JSON" loosely, may add extra fields

Instruction + 3 examples:
  "Always respond in JSON..." + 3 exact format examples
  → Model matches the exact structure demonstrated

Why: Instructions describe, examples demonstrate.
     Models learn better from demonstration than description.
```

---

## Shot Types Compared

| Type | Examples | Cost | Consistency | Best For |
|---|---|---|---|---|
| **Zero-Shot** | 0 | $ | Low | General tasks, exploration |
| **One-Shot** | 1 | $ | Medium | Simple formatting guidance |
| **Few-Shot** | 2–8 | $$ | High | Production classification/formatting |
| **Many-Shot** | 8–50+ | $$$ | Very High | Complex tasks, rare label types |
| **Fine-Tuning** | 100s–1000s | $$$$ | Highest | High-volume, stable tasks |

**Sweet spot:** 3–5 high-quality examples covers 80% of production use cases.

---

## Implementation Patterns

### Pattern 1: Basic Classification

```text
Classify customer support tickets into categories.

Ticket: "My payment was charged twice."
Category: BILLING

Ticket: "The app crashes when I try to upload a file."
Category: BUG

Ticket: "How do I export my data to CSV?"
Category: FEATURE_REQUEST

Ticket: "I haven't received my order from 2 weeks ago."
Category: SHIPPING

Ticket: "I need to reset my two-factor authentication."
Category:
```

---

### Pattern 2: Structured JSON Extraction

```text
Extract structured data from customer messages into JSON.
Use null for missing fields.

Message: "Hi, I'm Sarah Chen (sarah@email.com). I'd like to cancel order #4521."
Output: {"name":"Sarah Chen","email":"sarah@email.com","order_id":"4521","intent":"cancel"}

Message: "Order #8834 was delivered damaged. Need a refund ASAP."
Output: {"name":null,"email":null,"order_id":"8834","intent":"refund","issue":"damaged"}

Message: "When will my order arrive? I placed it on Monday."
Output: {"name":null,"email":null,"order_id":null,"intent":"status_enquiry","issue":null}

Message: "I'm James (james@co.com) and I want to upgrade my plan to Pro."
Output:
```

---

### Pattern 3: Text Transformation

```python
def few_shot_transform(text: str, style: str = "formal") -> str:
    """Transform text tone using few-shot examples."""

    examples = {
        "formal": [
            ("hey can u check this out asap",
             "Could you please review this at your earliest convenience?"),
            ("thx for the help, really appreciate it!!",
             "Thank you for your assistance. I greatly appreciate your support."),
            ("the meeting got pushed back, fyi",
             "Please be advised that the meeting has been rescheduled."),
        ],
        "casual": [
            ("Please be advised that your request has been received.",
             "Just letting you know we got your request!"),
            ("We regret to inform you of a delay in processing.",
             "Hey, sorry — there's a small delay on our end."),
            ("Kindly confirm receipt of this communication.",
             "Can you let me know you got this?"),
        ]
    }

    shots = examples.get(style, examples["formal"])
    prompt = f"Rewrite each message in a {style} tone.\n\n"

    for original, transformed in shots:
        prompt += f"Original: {original}\n{style.capitalize()}: {transformed}\n\n"

    prompt += f"Original: {text}\n{style.capitalize()}:"

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=256,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text.strip()
```

---

### Pattern 4: Named Entity Recognition

```text
Extract all named entities from the text. Format: TYPE: value

Text: "Apple CEO Tim Cook announced the new iPhone at their Cupertino headquarters."
Entities:
  ORG: Apple
  PERSON: Tim Cook
  PRODUCT: iPhone
  LOCATION: Cupertino

Text: "Dr. Sarah Johnson at Johns Hopkins published a study on COVID-19 vaccines."
Entities:
  PERSON: Dr. Sarah Johnson
  ORG: Johns Hopkins
  DISEASE: COVID-19
  PRODUCT: vaccines

Text: "Elon Musk's Tesla reported record deliveries in Shanghai last quarter."
Entities:
```

---

### Pattern 5: Dynamic Example Selection

For large example banks, select the most relevant examples per query using semantic similarity:

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity
import anthropic

client = anthropic.Anthropic()

class DynamicFewShot:
    def __init__(self, example_bank: list, task_description: str):
        """
        example_bank: list of {"input": str, "output": str}
        """
        self.examples = example_bank
        self.task = task_description
        self.embeddings = self._embed_examples()

    def _embed(self, texts: list) -> np.ndarray:
        """Get embeddings using Claude-compatible embedding model."""
        # Use your preferred embedding model here
        # e.g., OpenAI text-embedding-3-small, Cohere, etc.
        raise NotImplementedError("Plug in your embedding model")

    def _embed_examples(self) -> np.ndarray:
        inputs = [ex["input"] for ex in self.examples]
        return self._embed(inputs)

    def select(self, query: str, k: int = 3) -> list:
        """Select k most similar examples to the query."""
        query_emb = self._embed([query])
        scores = cosine_similarity(query_emb, self.embeddings)[0]
        top_k_idx = scores.argsort()[-k:][::-1]
        return [self.examples[i] for i in top_k_idx]

    def build_prompt(self, query: str, k: int = 3) -> str:
        shots = self.select(query, k)

        prompt = f"{self.task}\n\n"
        for shot in shots:
            prompt += f"Input: {shot['input']}\nOutput: {shot['output']}\n\n"
        prompt += f"Input: {query}\nOutput:"

        return prompt

    def run(self, query: str, k: int = 3) -> str:
        prompt = self.build_prompt(query, k)

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=512,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text.strip()


# Usage
classifier = DynamicFewShot(
    example_bank=[
        {"input": "Payment failed twice", "output": "BILLING"},
        {"input": "App keeps crashing on login", "output": "BUG"},
        {"input": "Can I export to Excel?", "output": "FEATURE_REQUEST"},
        # ... hundreds more examples
    ],
    task_description="Classify support tickets into categories."
)

result = classifier.run("I was charged the wrong amount on my invoice")
print(result)  # BILLING
```

---

## AI-Specific Use Cases

### Use Case 1: LLM Output Formatting (JSON Schema Enforcement)

```python
def enforce_schema(raw_text: str, target_schema: dict) -> dict:
    """Use few-shot to reliably extract structured data matching a schema."""

    schema_str = str(target_schema)

    prompt = f"""Extract information into this exact JSON schema: {schema_str}
Use null for any field not mentioned. No extra fields.

Text: "Alice Smith called about her account. She's been a customer since 2019 and wants to upgrade."
JSON: {{"name":"Alice Smith","contact_reason":"upgrade","customer_since":2019,"account_id":null}}

Text: "Ref #TK-4521: Device overheating reported by user in NY. Priority: HIGH"
JSON: {{"ticket_id":"TK-4521","issue":"overheating","location":"NY","priority":"HIGH","name":null}}

Text: "{raw_text}"
JSON:"""

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=256,
        messages=[{"role": "user", "content": prompt}]
    )

    import json
    return json.loads(response.content[0].text.strip())
```

---

### Use Case 2: Consistent Evaluation Rubric

```text
Rate the quality of AI-generated answers on a scale of 1-5.

Question: What causes rain?
Answer: "Water evaporates, forms clouds, falls as rain."
Score: 3
Reason: Correct but missing condensation step and lacks detail.

Question: What causes rain?
Answer: "Solar energy evaporates water into vapour, which rises and cools, condensing around particles to form clouds. When droplets combine and grow heavy enough, they fall as precipitation."
Score: 5
Reason: Complete, accurate, covers all key steps in the water cycle.

Question: What causes rain?
Answer: "Clouds get heavy and water falls down."
Score: 2
Reason: Oversimplified, misses evaporation and condensation entirely.

Question: What is photosynthesis?
Answer: "Plants use sunlight to make food from CO2 and water, releasing oxygen."
Score:
```

---

### Use Case 3: Synthetic Data Generation

```python
def generate_synthetic_examples(
    seed_examples: list,
    num_to_generate: int,
    domain: str
) -> list:
    """Generate synthetic training data using few-shot."""

    seeds_text = "\n\n".join([
        f"Input: {ex['input']}\nLabel: {ex['label']}"
        for ex in seed_examples
    ])

    prompt = f"""Generate {num_to_generate} new examples in the same style as these {domain} examples.
Each must be realistic, diverse, and follow the exact format.

{seeds_text}

Generate {num_to_generate} new examples:"""

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )

    # Parse generated examples
    raw = response.content[0].text
    examples = []
    for block in raw.strip().split("\n\n"):
        lines = block.strip().split("\n")
        if len(lines) >= 2:
            input_line = lines[0].replace("Input: ", "").strip()
            label_line = lines[1].replace("Label: ", "").strip()
            examples.append({"input": input_line, "label": label_line})

    return examples
```

---

## Advanced Patterns

### Pattern: Calibrated Few-Shot (With Confidence)

Include confidence scores in your examples to get calibrated model outputs:

```text
Classify the intent and provide a confidence score.

Text: "Cancel my subscription immediately"
Intent: CANCELLATION | Confidence: HIGH

Text: "I think I might want to downgrade maybe"
Intent: DOWNGRADE | Confidence: LOW

Text: "How much does the Pro plan cost?"
Intent: PRICING_ENQUIRY | Confidence: HIGH

Text: "I've been having some issues lately and not sure what to do"
Intent: GENERAL_COMPLAINT | Confidence: MEDIUM

Text: "Looking to possibly explore other options for my team"
Intent:
```

---

### Pattern: Contrastive Examples

Include negative examples (what NOT to do) to sharpen model behaviour:

```text
Write professional email subject lines.

Input: Meeting about Q3 results
GOOD: "Q3 Results Review — Action Items for Your Team"
BAD: "meeting" ← too vague
BAD: "URGENT!!! Q3 NUMBERS ARE IN — READ NOW" ← clickbait

Input: Asking for a deadline extension
GOOD: "Request for Extension: Project Alpha Deadline (Nov 15)"
BAD: "Need more time" ← no context
BAD: "Hey can we push the deadline???" ← too informal

Input: Following up on a job application
Subject:
```

---

### Pattern: Chain-of-Thought Few-Shot

Combine with CoT for complex classification:

```text
Classify whether a loan application should be APPROVED, REJECTED, or REVIEW.
Show your reasoning before the classification.

Application: Income $85K, Credit score 720, Loan $200K, Debt-to-income 28%
Reasoning: Income is solid. Credit score above 700 threshold. DTI at 28% is
           within the 36% limit. Loan-to-income ratio is 2.35x — acceptable.
Decision: APPROVED

Application: Income $32K, Credit score 580, Loan $150K, Debt-to-income 52%
Reasoning: Credit score below 600 minimum. DTI at 52% far exceeds 36% limit.
           Loan is 4.7x income — very high risk. Multiple rejection criteria met.
Decision: REJECTED

Application: Income $62K, Credit score 650, Loan $180K, Debt-to-income 38%
Reasoning: Credit score borderline — above 600 but below ideal 700. DTI slightly
           above 36% limit at 38%. Loan is 2.9x income. Mixed signals.
Decision: REVIEW

Application: Income $110K, Credit score 680, Loan $320K, Debt-to-income 41%
Reasoning:
```

---

## Combining with Other Patterns

### Few-Shot + CoT

Provide reasoning chains in your examples — the model learns both the format and the thinking:

```text
Each example shows full reasoning, not just the answer.
Use this pattern for tasks where the process matters.
```

### Few-Shot + ReAct

Show examples of the complete Thought/Action/Observation trace:

```text
Demonstrate 1-2 complete ReAct traces as few-shot examples
before presenting the new task to the agent.
This dramatically improves tool selection accuracy.
```

### Few-Shot for Prompt Hardening

Use few-shot to make prompts resilient to edge cases:

```text
Include examples that specifically cover:
- Ambiguous inputs
- Out-of-scope requests
- Adversarial inputs
- Edge case formats

This teaches the model how to handle unusual situations.
```

---

## Production Best Practices

### Example Selection Principles

1. **Diversity** — Cover the full distribution of real inputs, not just the easy cases
2. **Balance** — For N-class classification, aim for equal representation per class
3. **Recency position** — Place the most representative example last (highest influence)
4. **Realistic** — Use real or realistic examples, not toy data
5. **Quality over quantity** — 3 perfect examples beat 10 noisy ones

### Scaling Strategy

```
# Small scale (< 1K queries/day)
Static 3-5 examples in system prompt
→ Simple, fast, low latency

# Medium scale (1K–100K queries/day)
Static prompt with carefully curated examples
→ Review examples monthly, update when accuracy dips

# Large scale (> 100K queries/day)
Dynamic example selection via semantic search
→ Embedding-based retrieval from example bank
→ Cache embeddings, refresh bank weekly

# Very large scale / high accuracy requirement
Consider fine-tuning on your example bank
→ Few-shot for flexibility, fine-tuning for volume efficiency
```

### Token Cost Management

```
Example cost breakdown:
  1 example (input + output): ~50-150 tokens
  3 examples: ~150-450 tokens
  5 examples: ~250-750 tokens

At $3/M tokens (Claude Sonnet):
  3 examples at 100K calls/day = $13.50/day additional cost
  5 examples at 100K calls/day = $22.50/day additional cost

Optimisation: Use shorter, denser examples.
              Remove examples that don't add new information.
```

### Versioning Examples

```python
EXAMPLE_BANK = {
    "version": "2.1.0",
    "updated": "2026-03-01",
    "task": "support_ticket_classification",
    "examples": [
        {
            "id": "ex_001",
            "input": "My payment failed twice",
            "output": "BILLING",
            "added": "2025-11-01",
            "performance_lift": "+3.2%"
        },
        # ...
    ]
}
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| Inconsistent formatting between examples | Model gets confused about the real format | Standardise every example to identical structure |
| All examples from the happy path | Model fails on edge cases | Include examples of tricky, ambiguous inputs |
| Too many examples in the prompt | High token cost, diminishing returns beyond 8 | Use 3–5 examples; add more only with evidence |
| Examples that contradict each other | Model averages them out inconsistently | Audit examples for internal consistency |
| Static examples for dynamic tasks | Examples become stale over time | Review and refresh examples quarterly |
| Examples longer than necessary | Wastes context window space | Make examples as concise as the real task allows |

---

## Interview Cheat Sheet

**Q: What is Few-Shot prompting?**
A: Providing worked input-output examples directly in the prompt so the model learns the desired pattern via in-context learning — no fine-tuning or weight updates required.

**Q: How many examples are needed?**
A: 3–5 high-quality examples cover most production use cases. Beyond 8, returns diminish rapidly. One-shot (single example) is sufficient for simple format guidance.

**Q: When does few-shot outperform fine-tuning?**
A: When the task changes frequently, data volume is low (<1000 examples), or you need fast iteration. Fine-tuning wins at high volume, stable tasks, and strict latency budgets.

**Q: What is Dynamic Few-Shot?**
A: Instead of fixed examples in the prompt, you maintain a large example bank and select the k most semantically similar examples to each incoming query using embedding-based cosine similarity.

**Q: How do you handle class imbalance in few-shot classification?**
A: Deliberately balance examples across all classes in your prompt, even if the real distribution is imbalanced. Otherwise the model biases toward over-represented classes.

**Q: How do you version few-shot examples?**
A: Treat examples like code — version them with semantic versioning, track which examples were added when, measure the performance impact of each example, and roll back if accuracy drops.

**Q: What's the difference between few-shot and RAG?**
A: Few-shot embeds examples directly in the prompt to teach format/behaviour. RAG retrieves relevant *knowledge* documents to ground factual answers. They solve different problems and are often used together.

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
*← [Chain-of-Thought Guide](01-chain-of-thought-pattern-guide.md) | Next: [ReAct Guide](./react-pattern-guide.md) →*
