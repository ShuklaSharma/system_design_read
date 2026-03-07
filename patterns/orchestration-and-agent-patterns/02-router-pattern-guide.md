# Router Pattern for AI Applications

> AI Engineering Series | March 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [How the Router Pattern Works](#how-the-router-pattern-works)
3. [Router Types](#router-types)
4. [Implementation Patterns](#implementation-patterns)
5. [Routing Strategies](#routing-strategies)
6. [AI-Specific Use Cases](#ai-specific-use-cases)
7. [Advanced Patterns](#advanced-patterns)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Monitoring & Observability](#monitoring--observability)
10. [Production Best Practices](#production-best-practices)
11. [Common Mistakes](#common-mistakes)
12. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is the Router Pattern?

The **Router Pattern** is an architectural pattern that sits in front of multiple models, agents, or tools and **intelligently dispatches each incoming request to the most appropriate backend** — based on intent, complexity, cost, latency requirements, or content type.

Rather than sending every request to one expensive model, the router acts as a traffic controller: routing simple queries to a fast cheap model, complex reasoning to a powerful large model, code questions to a specialised coding model, and so on.

**Analogy:** Like a hospital triage system — a nurse quickly assesses each patient and routes them to the right care level. Not everyone needs a surgeon; routing minor issues to a GP saves cost and time for everyone.

---

### The Problem Without a Router

```
Without Router:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

All requests → GPT-4 / Claude Opus (most expensive model)

User: "What is 2 + 2?"         → $0.06  ← overkill
User: "Write a haiku"          → $0.04  ← overkill
User: "Refactor this 500-line codebase" → $0.06 ← appropriate
User: "Translate: Hello"       → $0.03  ← overkill

Result:
- 70% of requests are simple tasks that waste expensive capacity
- Monthly cost: $15,000
- Average latency: 4.2 seconds (waiting for large model)
- No specialisation advantage
```

### The Solution With a Router

```
With Router:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User: "What is 2 + 2?"         → GPT-3.5 (fast, $0.001)  ✅
User: "Write a haiku"          → GPT-3.5 (fast, $0.001)  ✅
User: "Refactor this codebase" → GPT-4   (powerful, $0.06) ✅
User: "Translate: Hello"       → translate API ($0.00002) ✅
User: "Debug this segfault"    → CodeLlama (specialised)  ✅

Result:
- Each request handled by the right tool
- Monthly cost: $3,200 (79% reduction)
- Average latency: 1.1 seconds (fast model handles most)
- Specialised models give better quality per task type
```

---

### When to Use the Router Pattern

| Use It When | Avoid It When |
|---|---|
| You have 3+ model/tool options | Single model handles everything well |
| Wide variation in request complexity | All requests are uniformly complex |
| Cost optimisation is a priority | Latency budget is very tight (adds ~10-50ms) |
| Requests span different domains/tasks | Domain is narrow and homogeneous |
| You want specialised quality per task | Routing errors would be catastrophic |

---

## How the Router Pattern Works

### Core Flow

```
Incoming Request
      │
      ▼
┌─────────────────┐
│     ROUTER      │  ← classifies intent, complexity, cost tier
│                 │
│  1. Analyse     │
│  2. Score       │
│  3. Select      │
└────────┬────────┘
         │
    ┌────┴─────────────────────────────────┐
    │                                      │
    ▼                                      ▼
Simple Intent                        Complex Intent
Cost: Low                            Cost: High
    │                                      │
    ▼                                      ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ GPT-3.5  │  │ Translate│  │  GPT-4   │  │ CodeLLM  │
│  (fast)  │  │   API    │  │ (reason) │  │  (code)  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
    │              │              │              │
    └──────────────┴──────────────┴──────────────┘
                         │
                         ▼
                  Final Response
```

### The Three-Step Router Decision

```
Step 1 — CLASSIFY
  Determine what kind of request this is.
  Methods: LLM classifier, embedding similarity,
           keyword rules, ML model

Step 2 — SCORE
  Score the request on dimensions relevant to routing:
  - Complexity   (simple / moderate / complex)
  - Domain       (code / math / creative / factual / ...)
  - Sensitivity  (public / private / PII)
  - Urgency      (real-time / batch)
  - Cost tier    (free / pro / enterprise)

Step 3 — SELECT
  Map scores to the best backend:
  - Simple + factual    → small fast model
  - Complex + reasoning → large model
  - Code-specific       → code model
  - Translation         → dedicated API
  - Sensitive data      → on-premise model
```

---

## Router Types

### Type 1: Rule-Based Router

Simple keyword/regex matching. Fast, deterministic, zero-cost.

```
Rules:
  contains("translate", "翻译", "traduire")  → Translation API
  contains("def ", "class ", "import ")      → Code Model
  length < 50 tokens AND no code            → Small Model
  else                                       → Large Model
```

**Pros:** Ultra-fast (~0ms), predictable, debuggable  
**Cons:** Brittle, misses edge cases, high maintenance

---

### Type 2: LLM-Based Router

Use a small, cheap LLM as the classifier itself.

```
Router Prompt:
  "Classify this request into one of: [simple, complex, code,
  translation, image]. Reply with just the category."

Cost: ~$0.0001 per classification
Latency: ~200ms
Accuracy: ~92%
```

**Pros:** Flexible, handles nuance, easy to update  
**Cons:** Adds latency, small cost, can itself fail

---

### Type 3: Embedding-Based Router

Embed the request, compute cosine similarity against pre-embedded "archetype" prompts for each route.

```
Archetypes:
  code_archetype      = embed("Write a Python function that...")
  creative_archetype  = embed("Write a poem about...")
  factual_archetype   = embed("What is the capital of...")
  reasoning_archetype = embed("If A implies B and B implies C...")

Route = argmax(cosine_similarity(query_embedding, archetypes))
```

**Pros:** Fast after embedding (~5ms), no LLM call needed  
**Cons:** Needs careful archetype design, embedding cost

---

### Type 4: ML Classifier Router

A trained text classifier (BERT, logistic regression, etc.) running locally.

```
Training data: 10,000 labelled examples
Labels: [simple, complex, code, translation, creative, math]
Model size: ~50MB (runs in memory)
Latency: ~2ms
Accuracy: ~95%
```

**Pros:** Fastest, most accurate, no API call  
**Cons:** Requires labelled training data, periodic retraining

---

### Router Type Comparison

| Type | Latency | Cost | Accuracy | Maintenance |
|---|---|---|---|---|
| Rule-based | ~0ms | Free | ~75% | High |
| LLM-based | ~200ms | ~$0.0001 | ~92% | Low |
| Embedding-based | ~5ms | ~$0.00001 | ~88% | Medium |
| ML Classifier | ~2ms | Free (local) | ~95% | Medium |
| Hybrid (rules + LLM) | ~50ms avg | Minimal | ~96% | Low |

---

## Implementation Patterns

### Pattern 1: Rule-Based Router (Baseline)

```python
import re
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class Route(Enum):
    SMALL_MODEL   = "small_model"      # GPT-3.5, Claude Haiku
    LARGE_MODEL   = "large_model"      # GPT-4, Claude Opus
    CODE_MODEL    = "code_model"       # CodeLlama, GPT-4-code
    TRANSLATE_API = "translate_api"    # DeepL, Google Translate
    MATH_MODEL    = "math_model"       # Wolfram, Llama-math
    IMAGE_MODEL   = "image_model"      # DALL-E, Stable Diffusion

@dataclass
class RouterDecision:
    route:      Route
    confidence: float
    reason:     str
    metadata:   dict

class RuleBasedRouter:
    """
    Fast, deterministic router using keyword and heuristic rules.
    Use as the first layer in a hybrid router.
    """

    # Code indicators
    CODE_PATTERNS = [
        r'\bdef\s+\w+\s*\(',      # Python function definition
        r'\bclass\s+\w+',          # Class definition
        r'\bimport\s+\w+',         # Import statement
        r'```[\w]*\n',             # Code block
        r'\bfunction\s+\w+\s*\(', # JS function
        r'<[a-z]+>.*</[a-z]+>',   # HTML tags
        r'SELECT\s+.+\s+FROM',    # SQL
    ]

    # Translation indicators
    TRANSLATE_PATTERNS = [
        r'\btranslate\b', r'\btranslation\b',
        r'\bin (french|spanish|german|japanese|chinese|arabic)\b',
        r'[\u4e00-\u9fff]',  # CJK characters
        r'[\u0600-\u06ff]',  # Arabic characters
    ]

    # Complex reasoning indicators
    COMPLEX_PATTERNS = [
        r'\banalyse\b', r'\banalyze\b', r'\bcompare\b',
        r'\bexplain why\b', r'\bprove\b', r'\bargue\b',
        r'\bstrategize\b', r'\boptimize\b', r'\bdebug\b',
    ]

    # Math indicators
    MATH_PATTERNS = [
        r'\bsolve\b.*\bequation\b',
        r'\bcalculate\b',
        r'\bintegral\b', r'\bderivative\b',
        r'[\d]+\s*[\+\-\*\/\^]\s*[\d]+',
    ]

    # Image generation indicators
    IMAGE_PATTERNS = [
        r'\bgenerate\b.*\bimage\b', r'\bdraw\b', r'\billustrate\b',
        r'\bpaint\b.*\bpicture\b', r'\bvisual\b.*\bcreate\b',
    ]

    def route(self, query: str) -> RouterDecision:
        q = query.lower()

        # Check image generation first (very specific)
        if any(re.search(p, q) for p in self.IMAGE_PATTERNS):
            return RouterDecision(Route.IMAGE_MODEL, 0.95, "image generation detected", {})

        # Check translation
        if any(re.search(p, query, re.IGNORECASE) for p in self.TRANSLATE_PATTERNS):
            return RouterDecision(Route.TRANSLATE_API, 0.93, "translation request", {})

        # Check code
        if any(re.search(p, query) for p in self.CODE_PATTERNS):
            return RouterDecision(Route.CODE_MODEL, 0.90, "code pattern detected", {})

        # Check math
        if any(re.search(p, q) for p in self.MATH_PATTERNS):
            return RouterDecision(Route.MATH_MODEL, 0.88, "math task detected", {})

        # Check complexity
        if any(re.search(p, q) for p in self.COMPLEX_PATTERNS):
            return RouterDecision(Route.LARGE_MODEL, 0.80, "complex reasoning required", {})

        # Length heuristic — long = likely complex
        word_count = len(query.split())
        if word_count > 100:
            return RouterDecision(Route.LARGE_MODEL, 0.70, f"long query ({word_count} words)", {"word_count": word_count})

        # Default: simple model
        return RouterDecision(Route.SMALL_MODEL, 0.75, "no complex patterns found", {"word_count": word_count})
```

---

### Pattern 2: LLM-Based Router

```python
import anthropic
import json

class LLMRouter:
    """
    Uses a small, cheap model to classify intent and route.
    Most flexible — handles nuanced cases rule-based routers miss.
    """

    ROUTER_SYSTEM_PROMPT = """You are a request classifier. Analyse the user's request
and classify it into exactly ONE of these categories:

- simple_qa:     Short factual questions, greetings, simple lookups
- complex:       Multi-step reasoning, analysis, comparison, strategy
- code:          Writing, reviewing, debugging, explaining code
- creative:      Stories, poems, marketing copy, brainstorming
- translation:   Translating text between languages
- math:          Mathematical calculations, equations, proofs
- image:         Generating, editing, or describing images
- data_analysis: Working with tables, CSVs, statistics

Respond with valid JSON only:
{
  "category": "<category>",
  "confidence": <0.0-1.0>,
  "reasoning": "<one sentence>",
  "complexity": "<low|medium|high>",
  "estimated_tokens": <integer>
}"""

    def __init__(self, anthropic_api_key: str):
        self.client = anthropic.Anthropic(api_key=anthropic_api_key)

    def route(self, query: str) -> RouterDecision:
        try:
            response = self.client.messages.create(
                model="claude-haiku-4-5-20251001",   # cheapest model for routing
                max_tokens=150,
                system=self.ROUTER_SYSTEM_PROMPT,
                messages=[{"role": "user", "content": query}]
            )

            raw = response.content[0].text.strip()
            # Strip markdown fences if present
            raw = raw.replace("```json", "").replace("```", "").strip()
            data = json.loads(raw)

            category_to_route = {
                "simple_qa":     Route.SMALL_MODEL,
                "complex":       Route.LARGE_MODEL,
                "code":          Route.CODE_MODEL,
                "creative":      Route.SMALL_MODEL,
                "translation":   Route.TRANSLATE_API,
                "math":          Route.MATH_MODEL,
                "image":         Route.IMAGE_MODEL,
                "data_analysis": Route.LARGE_MODEL,
            }

            route = category_to_route.get(data["category"], Route.LARGE_MODEL)

            # Upgrade to large model if complexity is high
            if data.get("complexity") == "high" and route == Route.SMALL_MODEL:
                route = Route.LARGE_MODEL

            return RouterDecision(
                route      = route,
                confidence = data.get("confidence", 0.8),
                reason     = data.get("reasoning", ""),
                metadata   = {
                    "category":         data.get("category"),
                    "complexity":       data.get("complexity"),
                    "estimated_tokens": data.get("estimated_tokens"),
                }
            )

        except (json.JSONDecodeError, KeyError) as e:
            # Fallback to large model on parse error
            return RouterDecision(Route.LARGE_MODEL, 0.5, f"router parse error: {e}", {})
        except Exception as e:
            return RouterDecision(Route.LARGE_MODEL, 0.5, f"router error: {e}", {})
```

---

### Pattern 3: Embedding-Based Router

```python
import numpy as np
from sentence_transformers import SentenceTransformer

class EmbeddingRouter:
    """
    Routes by semantic similarity to pre-defined route archetypes.
    No LLM call needed at query time — very fast.
    """

    # Representative prompts for each route
    ARCHETYPES = {
        Route.SMALL_MODEL: [
            "What is the capital of France?",
            "Who wrote Romeo and Juliet?",
            "What time is it in Tokyo?",
            "Tell me a joke",
            "What does API stand for?",
        ],
        Route.LARGE_MODEL: [
            "Analyse the geopolitical implications of this policy change",
            "Compare and contrast these two business strategies",
            "Write a detailed technical specification for this system",
            "Explain the trade-offs between these architectural approaches",
            "Provide a comprehensive critique of this argument",
        ],
        Route.CODE_MODEL: [
            "Write a Python function to sort a list of dictionaries",
            "Debug this JavaScript error: TypeError undefined",
            "Refactor this class to use dependency injection",
            "Explain what this SQL query does",
            "Implement a binary search tree in Java",
        ],
        Route.TRANSLATE_API: [
            "Translate 'Hello, how are you?' into Spanish",
            "What does 'Bonjour' mean in English?",
            "Convert this English paragraph to Japanese",
            "How do you say 'thank you' in Mandarin?",
        ],
        Route.MATH_MODEL: [
            "Solve the equation 3x^2 + 2x - 5 = 0",
            "Calculate the integral of sin(x) from 0 to pi",
            "What is the derivative of ln(x^2 + 1)?",
            "Find the eigenvalues of this matrix",
        ],
    }

    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        self.model = SentenceTransformer(model_name)
        self.archetype_embeddings = self._build_archetype_embeddings()

    def _build_archetype_embeddings(self) -> dict:
        """Pre-compute and average archetype embeddings at startup."""
        embeddings = {}
        for route, examples in self.ARCHETYPES.items():
            vecs = self.model.encode(examples, normalize_embeddings=True)
            embeddings[route] = np.mean(vecs, axis=0)  # centroid
        return embeddings

    def _cosine(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b) + 1e-9))

    def route(self, query: str) -> RouterDecision:
        query_vec = self.model.encode(query, normalize_embeddings=True)

        scores = {
            route: self._cosine(query_vec, centroid)
            for route, centroid in self.archetype_embeddings.items()
        }

        best_route = max(scores, key=scores.get)
        best_score = scores[best_route]
        second_score = sorted(scores.values(), reverse=True)[1]

        # If top-2 scores are very close, escalate to large model
        if best_route == Route.SMALL_MODEL and (best_score - second_score) < 0.05:
            best_route = Route.LARGE_MODEL
            reason = f"low confidence ({best_score:.2f} vs {second_score:.2f}), escalating"
        else:
            reason = f"similarity {best_score:.3f} to {best_route.value} archetype"

        return RouterDecision(
            route      = best_route,
            confidence = best_score,
            reason     = reason,
            metadata   = {"all_scores": {r.value: round(s, 3) for r, s in scores.items()}}
        )
```

---

### Pattern 4: Hybrid Router (Recommended for Production)

Rules first (fast, free), LLM fallback for uncertain cases only:

```python
class HybridRouter:
    """
    Layered routing:
    1. Rule-based router (free, instant) → high confidence → use it
    2. Embedding router (fast, cheap)    → medium confidence → use it
    3. LLM router (accurate, slower)     → low confidence → use it
    4. Default fallback                  → large model

    ~80% of requests resolved by rules (0ms, free)
    ~15% resolved by embeddings (~5ms, ~$0.00001)
    ~5%  resolved by LLM (~200ms, ~$0.0001)
    """

    RULE_CONFIDENCE_THRESHOLD      = 0.88
    EMBEDDING_CONFIDENCE_THRESHOLD = 0.82

    def __init__(self, anthropic_api_key: str):
        self.rule_router      = RuleBasedRouter()
        self.embedding_router = EmbeddingRouter()
        self.llm_router       = LLMRouter(anthropic_api_key)

    def route(self, query: str) -> RouterDecision:
        # Layer 1: Rules (instant)
        decision = self.rule_router.route(query)
        if decision.confidence >= self.RULE_CONFIDENCE_THRESHOLD:
            decision.metadata["router_used"] = "rule"
            return decision

        # Layer 2: Embedding similarity (fast)
        decision = self.embedding_router.route(query)
        if decision.confidence >= self.EMBEDDING_CONFIDENCE_THRESHOLD:
            decision.metadata["router_used"] = "embedding"
            return decision

        # Layer 3: LLM classification (accurate)
        decision = self.llm_router.route(query)
        decision.metadata["router_used"] = "llm"
        return decision
```

---

### Pattern 5: Full Router Gateway

Wiring the router to actual model backends:

```python
import anthropic
import openai
import httpx
from typing import Dict, Any

class RouterGateway:
    """
    Complete gateway: routes request → dispatches to backend → returns response.
    """

    def __init__(self, config: dict):
        self.router   = HybridRouter(config["anthropic_key"])
        self.clients  = {
            "anthropic": anthropic.Anthropic(api_key=config["anthropic_key"]),
            "openai":    openai.OpenAI(api_key=config["openai_key"]),
        }
        self.metrics  = RouterMetrics()

    async def handle(self, query: str, user_context: dict = None) -> dict:
        import time
        start = time.time()

        # 1. Route
        decision = self.router.route(query)
        self.metrics.record_route(decision)

        # 2. Dispatch
        try:
            response = await self._dispatch(query, decision, user_context or {})
            latency  = (time.time() - start) * 1000

            self.metrics.record_success(decision.route, latency)

            return {
                "answer":        response["text"],
                "model_used":    response["model"],
                "route":         decision.route.value,
                "confidence":    decision.confidence,
                "router_reason": decision.reason,
                "latency_ms":    round(latency),
                "cost_usd":      response.get("cost", 0),
            }

        except Exception as e:
            self.metrics.record_error(decision.route)
            # Fallback: retry with large model
            if decision.route != Route.LARGE_MODEL:
                fallback = await self._call_large_model(query)
                return {**fallback, "route": "fallback_large_model"}
            raise

    async def _dispatch(self, query: str, decision: RouterDecision, ctx: dict) -> dict:
        route = decision.route

        if route == Route.SMALL_MODEL:
            return await self._call_small_model(query)
        elif route == Route.LARGE_MODEL:
            return await self._call_large_model(query)
        elif route == Route.CODE_MODEL:
            return await self._call_code_model(query)
        elif route == Route.TRANSLATE_API:
            return await self._call_translation(query)
        elif route == Route.MATH_MODEL:
            return await self._call_math_model(query)
        else:
            return await self._call_large_model(query)

    async def _call_small_model(self, query: str) -> dict:
        response = self.clients["anthropic"].messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=512,
            messages=[{"role": "user", "content": query}]
        )
        return {
            "text":  response.content[0].text,
            "model": "claude-haiku-4-5-20251001",
            "cost":  (response.usage.input_tokens * 0.00025 +
                      response.usage.output_tokens * 0.00125) / 1000,
        }

    async def _call_large_model(self, query: str) -> dict:
        response = self.clients["anthropic"].messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            messages=[{"role": "user", "content": query}]
        )
        return {
            "text":  response.content[0].text,
            "model": "claude-sonnet-4-6",
            "cost":  (response.usage.input_tokens * 0.003 +
                      response.usage.output_tokens * 0.015) / 1000,
        }

    async def _call_code_model(self, query: str) -> dict:
        # Could be a dedicated coding model or GPT-4 with code system prompt
        response = self.clients["openai"].chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "You are an expert software engineer."},
                {"role": "user",   "content": query}
            ],
            max_tokens=2048,
        )
        return {
            "text":  response.choices[0].message.content,
            "model": "gpt-4o",
            "cost":  (response.usage.prompt_tokens * 0.005 +
                      response.usage.completion_tokens * 0.015) / 1000,
        }

    async def _call_translation(self, query: str) -> dict:
        # Dedicated translation API (DeepL, Google Translate, etc.)
        # Placeholder — swap with your preferred translation service
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                "https://api.deepl.com/v2/translate",
                data={"text": query, "target_lang": "EN", "auth_key": "YOUR_DEEPL_KEY"}
            )
            result = resp.json()
        return {
            "text":  result["translations"][0]["text"],
            "model": "deepl",
            "cost":  len(query) * 0.00002,  # DeepL pricing
        }

    async def _call_math_model(self, query: str) -> dict:
        # Use LLM with math-specific system prompt, or Wolfram Alpha API
        response = self.clients["anthropic"].messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            system="You are a mathematics expert. Show your working step by step.",
            messages=[{"role": "user", "content": query}]
        )
        return {
            "text":  response.content[0].text,
            "model": "claude-sonnet-4-6 (math)",
            "cost":  (response.usage.input_tokens * 0.003 +
                      response.usage.output_tokens * 0.015) / 1000,
        }
```

---

## Routing Strategies

### Strategy 1: Cost-Based Routing

Route by token estimate to stay within budget:

```python
def cost_based_route(query: str, user_tier: str) -> Route:
    """Upgrade or downgrade route based on user tier and token budget."""

    estimated_tokens = len(query.split()) * 1.3   # rough estimate

    tier_budgets = {
        "free":       {"max_tokens": 500,   "max_cost": 0.001},
        "pro":        {"max_tokens": 2000,  "max_cost": 0.05},
        "enterprise": {"max_tokens": 10000, "max_cost": 1.00},
    }

    budget = tier_budgets.get(user_tier, tier_budgets["free"])

    if estimated_tokens > budget["max_tokens"]:
        return Route.SMALL_MODEL   # force cheaper model if over budget

    return None   # let other routing logic decide
```

---

### Strategy 2: Latency-Based Routing

Route differently based on latency SLA:

```python
def latency_aware_route(
    base_decision: RouterDecision,
    sla_ms: int,
    current_large_model_p95_ms: int
) -> RouterDecision:
    """Downgrade to faster model if SLA can't be met."""

    model_latencies = {
        Route.SMALL_MODEL:   200,    # p95 ms
        Route.LARGE_MODEL:   3500,   # p95 ms (live value injected)
        Route.CODE_MODEL:    2000,
        Route.TRANSLATE_API: 300,
    }

    expected_ms = model_latencies.get(base_decision.route, 2000)

    if expected_ms > sla_ms * 0.9:    # 90% headroom
        # Downgrade to small model
        return RouterDecision(
            route      = Route.SMALL_MODEL,
            confidence = base_decision.confidence,
            reason     = f"SLA {sla_ms}ms cannot be met by {base_decision.route.value} (p95={expected_ms}ms)",
            metadata   = {**base_decision.metadata, "sla_downgrade": True}
        )

    return base_decision
```

---

### Strategy 3: Semantic Complexity Scoring

Score complexity with multiple signals before routing:

```python
def complexity_score(query: str) -> float:
    """
    Returns a complexity score 0.0 (trivial) → 1.0 (very complex).
    Combines multiple signals.
    """
    score = 0.0
    q = query.lower()

    # Signal 1: Length
    words = len(query.split())
    score += min(words / 200, 0.3)          # max 0.3 contribution

    # Signal 2: Reasoning keywords
    reasoning_words = ["analyse", "compare", "evaluate", "critique",
                       "argue", "prove", "strategy", "trade-off", "why"]
    matches = sum(1 for w in reasoning_words if w in q)
    score += min(matches * 0.08, 0.25)      # max 0.25 contribution

    # Signal 3: Multi-part questions
    questions = query.count("?")
    score += min(questions * 0.07, 0.15)    # max 0.15 contribution

    # Signal 4: Technical depth indicators
    technical = ["algorithm", "architecture", "scalability", "complexity",
                 "optimise", "implement", "design pattern"]
    matches = sum(1 for w in technical if w in q)
    score += min(matches * 0.06, 0.20)      # max 0.20 contribution

    # Signal 5: Presence of numbers/data (quantitative reasoning)
    numbers = len(re.findall(r'\b\d+\.?\d*\b', query))
    score += min(numbers * 0.01, 0.10)      # max 0.10 contribution

    return min(score, 1.0)


def route_by_complexity(query: str) -> Route:
    score = complexity_score(query)
    if score < 0.3:
        return Route.SMALL_MODEL
    elif score < 0.6:
        return Route.SMALL_MODEL   # medium — still try small first
    else:
        return Route.LARGE_MODEL
```

---

## AI-Specific Use Cases

### Use Case 1: Multi-Tier Customer Support Bot

```python
class CustomerSupportRouter:
    """
    Routes support tickets to appropriate handling:
    - FAQ / greeting        → template response (free)
    - Simple question       → small LLM
    - Complex issue         → large LLM
    - Billing / account     → human agent hand-off
    - Technical debug       → engineering escalation
    """

    INTENT_MAP = {
        "faq":        "template",
        "simple":     Route.SMALL_MODEL,
        "complex":    Route.LARGE_MODEL,
        "billing":    "human_agent",
        "technical":  "engineering_queue",
        "escalation": "human_agent",
    }

    SENSITIVE_KEYWORDS = [
        "refund", "charge", "billing", "payment", "lawsuit",
        "sue", "angry", "cancel account", "worst", "fraud"
    ]

    def route(self, query: str, user_history: list = None) -> dict:
        q_lower = query.lower()

        # Hard rule: sensitive topics → human agent immediately
        if any(kw in q_lower for kw in self.SENSITIVE_KEYWORDS):
            return {
                "destination": "human_agent",
                "reason":      "sensitive topic detected",
                "priority":    "high"
            }

        # Check if repeat contact (frustrated customer)
        if user_history and len(user_history) >= 3:
            return {
                "destination": "human_agent",
                "reason":      "repeat contact — possible unresolved issue",
                "priority":    "medium"
            }

        # Route by classified intent
        intent = self._classify_intent(query)
        destination = self.INTENT_MAP.get(intent, Route.LARGE_MODEL)

        return {
            "destination": destination,
            "intent":      intent,
            "reason":      f"classified as {intent}",
            "priority":    "low"
        }

    def _classify_intent(self, query: str) -> str:
        # Use LLM router for intent classification
        # (abbreviated — use LLMRouter from Pattern 2)
        return "simple"   # placeholder
```

---

### Use Case 2: Document Processing Pipeline Router

```python
class DocumentRouter:
    """
    Routes documents to specialised processors based on content type.
    """

    def route_document(self, content: str, file_type: str, task: str) -> dict:

        # File type routing
        if file_type in [".py", ".js", ".ts", ".java", ".go", ".rs"]:
            return {"model": Route.CODE_MODEL, "system": "You are a senior software engineer."}

        if file_type in [".pdf", ".docx"] and "summarise" in task.lower():
            word_count = len(content.split())
            if word_count > 10000:
                # Very long document — needs large context model
                return {"model": Route.LARGE_MODEL, "chunk": True}
            return {"model": Route.SMALL_MODEL, "chunk": False}

        if file_type in [".csv", ".xlsx"]:
            return {"model": Route.LARGE_MODEL, "system": "You are a data analyst."}

        # Task-based routing
        task_lower = task.lower()
        if "translate" in task_lower:
            return {"model": Route.TRANSLATE_API}
        if "extract" in task_lower or "classify" in task_lower:
            return {"model": Route.SMALL_MODEL}

        return {"model": Route.LARGE_MODEL}
```

---

### Use Case 3: Cost-Optimised SaaS Router

```python
class SaaSRouter:
    """
    Routes to maximise quality within each user's subscription tier.
    Free tier → small model, Pro → large model, Enterprise → best model.
    """

    TIER_CONFIG = {
        "free": {
            "model":          Route.SMALL_MODEL,
            "max_tokens":     500,
            "requests_limit": 20,      # per day
        },
        "pro": {
            "model":          Route.LARGE_MODEL,
            "max_tokens":     4000,
            "requests_limit": 500,
        },
        "enterprise": {
            "model":          Route.LARGE_MODEL,
            "max_tokens":     32000,
            "requests_limit": None,    # unlimited
        },
    }

    def route(self, query: str, user: dict) -> RouterDecision:
        tier   = user.get("subscription_tier", "free")
        config = self.TIER_CONFIG.get(tier, self.TIER_CONFIG["free"])

        # Check daily usage limit
        if config["requests_limit"] and user.get("requests_today", 0) >= config["requests_limit"]:
            return RouterDecision(
                route      = Route.SMALL_MODEL,
                confidence = 1.0,
                reason     = "daily limit reached — downgraded to free tier model",
                metadata   = {"limit_reached": True, "tier": tier}
            )

        return RouterDecision(
            route      = config["model"],
            confidence = 1.0,
            reason     = f"{tier} tier routing",
            metadata   = {"tier": tier, "max_tokens": config["max_tokens"]}
        )
```

---

## Advanced Patterns

### Pattern: Cascading Router (Try Cheap → Escalate if Needed)

```python
class CascadingRouter:
    """
    Try the cheapest model first.
    If the response quality is insufficient, escalate to a better model.
    Maximises cost savings while maintaining quality floor.
    """

    def __init__(self, anthropic_client):
        self.client = anthropic_client

    def _quality_check(self, response: str, query: str) -> bool:
        """Quick heuristic quality check on the response."""
        if len(response) < 20:
            return False    # too short
        if "i don't know" in response.lower() or "i cannot" in response.lower():
            return False    # model gave up
        if response.count("?") > 3:
            return False    # response is mostly questions back
        return True

    def handle(self, query: str) -> dict:
        # Step 1: Try small model
        small_response = self.client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=512,
            messages=[{"role": "user", "content": query}]
        )
        text = small_response.content[0].text

        if self._quality_check(text, query):
            return {"answer": text, "model": "haiku", "escalated": False}

        # Step 2: Escalate to large model
        large_response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            messages=[{"role": "user", "content": query}]
        )
        return {
            "answer":    large_response.content[0].text,
            "model":     "sonnet",
            "escalated": True,
            "reason":    "small model response quality insufficient"
        }
```

---

### Pattern: Context-Aware Router

```python
class ContextAwareRouter:
    """
    Routes based on full conversation context, not just the latest message.
    Useful when a simple follow-up question is actually part of a complex thread.
    """

    def route(self, query: str, conversation_history: list) -> RouterDecision:
        # Check if conversation has been complex so far
        prior_topics = self._extract_topics(conversation_history)
        query_length = sum(len(m.get("content", "").split()) for m in conversation_history)

        is_complex_thread = (
            len(conversation_history) > 6 or
            query_length > 500 or
            any(t in prior_topics for t in ["code", "analysis", "architecture"])
        )

        # A simple follow-up in a complex thread still needs the large model
        if is_complex_thread:
            return RouterDecision(
                Route.LARGE_MODEL, 0.85,
                "complex conversation history", {"thread_length": len(conversation_history)}
            )

        # Otherwise route the query on its own merits
        return HybridRouter._instance.route(query)

    def _extract_topics(self, history: list) -> list:
        topics = []
        for msg in history:
            content = msg.get("content", "").lower()
            if any(kw in content for kw in ["def ", "class ", "import "]):
                topics.append("code")
            if any(kw in content for kw in ["analyse", "compare", "strategy"]):
                topics.append("analysis")
        return topics
```

---

### Pattern: A/B Testing Router

```python
import random

class ABTestingRouter:
    """
    Routes a percentage of traffic to an experimental model
    to compare quality and cost in production.
    """

    def __init__(self, base_router, experiment_config: dict):
        self.base_router = base_router
        self.experiments = experiment_config
        # experiment_config = {
        #   "exp_001": {"traffic_pct": 0.10, "model": Route.CODE_MODEL, "routes": [Route.LARGE_MODEL]}
        # }

    def route(self, query: str, user_id: str = "") -> RouterDecision:
        base_decision = self.base_router.route(query)

        for exp_id, config in self.experiments.items():
            if base_decision.route not in config["routes"]:
                continue

            # Deterministic bucket based on user_id for consistency
            bucket = int(hash(f"{user_id}:{exp_id}") % 100) / 100
            if bucket < config["traffic_pct"]:
                return RouterDecision(
                    route      = config["model"],
                    confidence = base_decision.confidence,
                    reason     = f"A/B experiment {exp_id}",
                    metadata   = {**base_decision.metadata, "experiment_id": exp_id}
                )

        return base_decision
```

---

## Integration with Other Patterns

### Router + Circuit Breaker

If a routed backend is down, the circuit breaker opens and the router falls back to an alternative:

```python
class ResilientRouter:
    def __init__(self, router, circuit_breakers: dict):
        self.router   = router
        self.breakers = circuit_breakers   # {Route: CircuitBreaker}

    def route_with_fallback(self, query: str) -> RouterDecision:
        decision = self.router.route(query)

        # Check if the chosen route's circuit breaker is open
        breaker = self.breakers.get(decision.route)
        if breaker and breaker.is_open():
            # Fall back to large model (assumed most reliable)
            return RouterDecision(
                route      = Route.LARGE_MODEL,
                confidence = 0.9,
                reason     = f"{decision.route.value} circuit open — falling back",
                metadata   = {**decision.metadata, "circuit_fallback": True}
            )

        return decision
```

---

### Router + Bulkhead

Isolate capacity per route to prevent one busy route from starving others:

```python
from asyncio import Semaphore

class BulkheadRouter:
    ROUTE_CONCURRENCY = {
        Route.SMALL_MODEL:   50,   # high concurrency — cheap
        Route.LARGE_MODEL:   10,   # limited — expensive
        Route.CODE_MODEL:    15,
        Route.TRANSLATE_API: 100,
    }

    def __init__(self):
        self.semaphores = {
            route: Semaphore(limit)
            for route, limit in self.ROUTE_CONCURRENCY.items()
        }

    async def handle(self, query: str, router) -> dict:
        decision = router.route(query)
        sem = self.semaphores.get(decision.route, Semaphore(10))

        async with sem:
            return await gateway.dispatch(query, decision)
```

---

### Router + Retry

On failure, retry with a more capable model instead of the same one:

```python
async def route_with_smart_retry(query: str, router, gateway) -> dict:
    decision = router.route(query)

    try:
        return await gateway.dispatch(query, decision)
    except Exception as first_error:
        # Don't retry the same model — escalate
        if decision.route != Route.LARGE_MODEL:
            fallback = RouterDecision(
                route=Route.LARGE_MODEL,
                confidence=1.0,
                reason=f"retry escalation after {type(first_error).__name__}",
                metadata={"original_route": decision.route.value}
            )
            return await gateway.dispatch(query, fallback)
        raise
```

---

## Monitoring & Observability

### Router Metrics

```python
from dataclasses import dataclass, field
from collections import defaultdict
import time

class RouterMetrics:
    def __init__(self):
        self.route_counts     = defaultdict(int)
        self.route_latencies  = defaultdict(list)
        self.route_costs      = defaultdict(float)
        self.route_errors     = defaultdict(int)
        self.confidence_scores = []

    def record_route(self, decision: RouterDecision):
        self.route_counts[decision.route.value] += 1
        self.confidence_scores.append(decision.confidence)

    def record_success(self, route: Route, latency_ms: float, cost_usd: float = 0):
        self.route_latencies[route.value].append(latency_ms)
        self.route_costs[route.value] += cost_usd

    def record_error(self, route: Route):
        self.route_errors[route.value] += 1

    def summary(self) -> dict:
        total = sum(self.route_counts.values()) or 1
        return {
            "total_requests": total,
            "route_distribution": {
                route: {"count": count, "pct": round(count / total * 100, 1)}
                for route, count in sorted(self.route_counts.items())
            },
            "avg_confidence":  round(sum(self.confidence_scores) / len(self.confidence_scores), 3)
                               if self.confidence_scores else 0,
            "cost_by_route":   {r: round(c, 4) for r, c in self.route_costs.items()},
            "total_cost_usd":  round(sum(self.route_costs.values()), 4),
            "p95_latency_ms":  {
                r: round(sorted(lats)[int(len(lats) * 0.95)], 1)
                for r, lats in self.route_latencies.items() if lats
            },
            "error_rates":     {
                r: round(self.route_errors[r] / max(self.route_counts[r], 1) * 100, 2)
                for r in self.route_errors
            },
        }
```

### Key Metrics to Track

| Metric | Why It Matters | Alert Threshold |
|---|---|---|
| **Route distribution** | Detect unexpected shifts in traffic patterns | > 20% change from baseline |
| **Router confidence** | Low confidence = poor routing decisions | Avg < 0.75 |
| **Routing latency** | Router itself adds overhead | p99 > 500ms |
| **Escalation rate** | How often cheap model escalated to expensive | > 30% |
| **Misroute rate** | Manual audit of sampled decisions | > 5% wrong routes |
| **Cost per route** | Track spend by model | Set per-route budget alerts |

---

## Production Best Practices

### 1. Always Have a Fallback

```python
# Never let a router failure kill the request
try:
    decision = router.route(query)
except Exception:
    # Router failed — default to large model (safe choice)
    decision = RouterDecision(Route.LARGE_MODEL, 0.5, "router error fallback", {})
```

---

### 2. Log Every Decision

```python
# Every routing decision should be logged with enough detail to debug misroutes
log_entry = {
    "timestamp":  time.time(),
    "query_hash": hashlib.md5(query.encode()).hexdigest(),
    "route":      decision.route.value,
    "confidence": decision.confidence,
    "reason":     decision.reason,
    "router_used": decision.metadata.get("router_used"),
    "user_id":    user_id,
}
```

---

### 3. Sample and Audit Routing Decisions

```python
# Periodically sample decisions for human review
import random

def maybe_audit(decision: RouterDecision, query: str):
    # Always audit low-confidence decisions
    if decision.confidence < 0.70:
        send_to_audit_queue(query, decision, reason="low_confidence")
        return

    # Random 1% sampling for quality monitoring
    if random.random() < 0.01:
        send_to_audit_queue(query, decision, reason="random_sample")
```

---

### 4. Version Your Routing Rules

```python
ROUTING_CONFIG_V2 = {
    "version":    "2.1.0",
    "updated":    "2026-03-01",
    "thresholds": {
        "complexity_large_model": 0.6,
        "rule_confidence_cutoff": 0.88,
    },
    "routes": {
        "code":        "code_model",
        "translation": "translate_api",
        "complex":     "large_model",
        "default":     "small_model",
    }
}
# Treat routing config like code: version control, changelog, staged rollout
```

---

### 5. Test Routing Accuracy Regularly

```python
ROUTING_TEST_SUITE = [
    ("What is 2 + 2?",                                  Route.SMALL_MODEL),
    ("Translate 'hello' to French",                     Route.TRANSLATE_API),
    ("Write a Python quicksort",                        Route.CODE_MODEL),
    ("Analyse the trade-offs of microservices vs mono", Route.LARGE_MODEL),
    ("def fibonacci(n):",                               Route.CODE_MODEL),
    ("What is the capital of Germany?",                 Route.SMALL_MODEL),
]

def run_routing_tests(router) -> float:
    correct = 0
    for query, expected_route in ROUTING_TEST_SUITE:
        decision = router.route(query)
        if decision.route == expected_route:
            correct += 1
        else:
            print(f"  FAIL: '{query[:50]}' → {decision.route.value} (expected {expected_route.value})")
    accuracy = correct / len(ROUTING_TEST_SUITE)
    print(f"\nRouting accuracy: {accuracy:.0%} ({correct}/{len(ROUTING_TEST_SUITE)})")
    return accuracy
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| No fallback on router failure | Router error kills the whole request | Wrap in try/except, default to large model |
| Routing only on first message | Misroutes follow-up questions in complex threads | Consider full conversation history |
| Over-complex rule sets | Hard to maintain, brittle to edge cases | Start simple, add rules only when tests fail |
| No confidence threshold | Routes even when classifier is uncertain | If confidence < 0.70, default to large model |
| Routing sensitive content to external APIs | Privacy / compliance violation | Always check data sensitivity before routing |
| No routing test suite | Silent regression when rules change | Maintain curated test set, run in CI |
| Same retry model on failure | Retrying a broken route | Escalate to different (larger) model on retry |
| Routing without logging | Can't debug misroutes in production | Log every decision with query hash, route, confidence |

---

## Interview Cheat Sheet

**Q: What is the Router Pattern in AI systems?**
A: A dispatcher that sits in front of multiple models or tools and routes each request to the most appropriate backend based on intent, complexity, cost, or latency requirements. Saves cost by using cheap models for simple tasks and powerful models only when needed.

**Q: What are the main router types?**
A: Rule-based (keyword/regex, ~0ms, free, ~75% accurate), LLM-based (cheap model classifies intent, ~200ms, ~92% accurate), embedding-based (cosine similarity to archetypes, ~5ms, ~88% accurate), ML classifier (trained locally, ~2ms, ~95% accurate). Best production approach is hybrid: rules first, embedding fallback, LLM for uncertain cases.

**Q: What dimensions do you route on?**
A: Intent/domain (code vs translation vs reasoning), complexity (simple vs complex), cost tier (user subscription level), latency SLA (real-time vs batch), data sensitivity (PII must stay on-premise), and content type (text, code, math, image).

**Q: How do you handle router failures?**
A: Wrap every router call in try/except with a safe default (usually the large model). Never let a routing decision kill the request. Log all failures for debugging.

**Q: How do you measure routing accuracy?**
A: Maintain a labelled test suite of (query, expected_route) pairs. Run it in CI whenever routing rules change. Sample production decisions weekly for human audit — target < 5% misroute rate. Track confidence score distribution (avg < 0.75 signals a problem).

**Q: How does the Router Pattern integrate with Circuit Breaker?**
A: The router selects the ideal backend. The circuit breaker monitors each backend's health. If the circuit for the chosen route is open, the router falls back to the next best option (usually the large model as the safest fallback).

**Q: What is a Cascading Router?**
A: Try the cheapest model first. If the response quality check fails (too short, "I don't know", mostly questions back), escalate to the next model tier. Maximises cost savings while maintaining a quality floor.

**Q: How do you handle sensitive data routing?**
A: Check for PII, financial data, or proprietary content before routing. Sensitive requests must go to on-premise models or providers with appropriate data processing agreements — never to general cloud APIs.

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
*See also: [Orchestrator subagent patterns Guide](./03-orchestrator-subagent-pattern-guide.md)*
