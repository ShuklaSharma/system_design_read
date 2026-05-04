# Semantic Cache Pattern for AI Applications

> AI Engineering Series | May 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Semantic Cache Matters](#why-semantic-cache-matters)
3. [How It Works](#how-it-works)
4. [Core Components](#core-components)
5. [Building a Semantic Cache — Step by Step](#building-a-semantic-cache)
6. [Similarity Thresholds & Embedding Models](#similarity-thresholds--embedding-models)
7. [Cache Key Design](#cache-key-design)
8. [Negative Caching & Stale-Hit Prevention](#negative-caching--stale-hit-prevention)
9. [AI-Specific Use Cases](#ai-specific-use-cases)
10. [Advanced Patterns](#advanced-patterns)
11. [Monitoring & Observability](#monitoring--observability)
12. [Integration with Other Patterns](#integration-with-other-patterns)
13. [Production Best Practices](#production-best-practices)
14. [Common Mistakes](#common-mistakes)
15. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is Semantic Cache?

A **semantic cache** stores LLM responses keyed by the **meaning** of the request, not the exact byte string. When a new request arrives, the cache embeds it, runs a vector similarity search against previously cached requests, and — if any cached request is "close enough" — returns its stored response.

```
Traditional cache (exact match):
─────────────────────────────────────────────────────
"What is your refund policy?" → key = sha256("What is your refund policy?")
"What's your refund policy?"  → key = sha256("What's your refund policy?")
   ↑ DIFFERENT KEYS → cache miss, second call hits the LLM

Semantic cache (meaning match):
─────────────────────────────────────────────────────
"What is your refund policy?" → embed → vector A
"What's your refund policy?"  → embed → vector B
   cos(A, B) = 0.97 → cache HIT, response served from cache
```

**Origin:** Popularised by GPTCache (2023) and Redis Vector Library (2024); now standard in production LLM gateways.

### The Problem It Solves

Cache-aside with hash keys breaks down on natural-language traffic. Users phrase the same question dozens of ways:

```
All of these mean the same thing:
- "How do I reset my password?"
- "I forgot my password, what now?"
- "Password reset steps?"
- "I can't log in — how do I get a new password?"

Hash-based cache hit rate: 4–8% (only exact repeats)
Semantic cache hit rate:   60–85% (paraphrases match)
```

The economic gap is enormous: at 1M queries/day and $0.01 per LLM call, going from 8% to 75% cache hits saves **$2,400/day** ($72K/month).

---

## Why Semantic Cache Matters

### 1. Natural Language is Inherently Variable

Humans never ask the same question twice the same way. Hash-based caches assume identity that doesn't exist. Semantic caches encode the question into a 768- or 1536-dimensional vector and compare those — small surface differences disappear.

### 2. LLM Responses Are Expensive and Slow

| Operation | Latency | Cost |
|---|---|---|
| GPT-4 / Opus call | 800–4000 ms | $0.01–0.10 |
| Embedding lookup | 80–150 ms | $0.00002 |
| Vector search (10K vectors) | 5–15 ms | $0 |
| **Semantic cache hit total** | **~100 ms** | **~$0.00002** |

A semantic cache hit is **20–40× faster** and **500–5000× cheaper** than a fresh LLM call.

### 3. Resilience During Outages

When the LLM provider is down, a populated semantic cache continues to serve the most-asked questions. We've seen production systems retain 60–70% functional capacity through multi-hour Anthropic/OpenAI incidents purely from semantic cache hits.

### 4. Real Incident: No Semantic Cache

```
SaaS support chatbot, 80K queries/day
─────────────────────────────────────────────────────
Hash cache hit rate:   6%
LLM calls/day:         75,200
Daily LLM spend:       $902
P95 latency:           2.4s
Provider outage 14:00–16:30 → 100% error rate
```

```
After adding semantic cache (threshold 0.93)
─────────────────────────────────────────────────────
Semantic hit rate:     74%
LLM calls/day:         20,800
Daily LLM spend:       $250  (-72%)
P95 latency:           340ms (-86%)
Provider outage:       still served 74% of traffic
```

---

## How It Works

### The Lookup Flow

```
                      ┌──────────────────┐
   user query  ─────► │  embed query     │  (~80 ms, cached models)
                      └────────┬─────────┘
                               ▼
                      ┌──────────────────┐
                      │ vector search    │  top-1, cosine
                      │ (cache index)    │
                      └────────┬─────────┘
                               ▼
                      ┌──────────────────┐
                  ┌── │ similarity ≥ τ ? │ ── no ──► call LLM
                  yes └──────────────────┘             │
                  ▼                                     ▼
          ┌──────────────────┐               ┌──────────────────┐
          │ optional verify  │               │ store {query,    │
          │ (LLM-as-judge or │               │  embedding,      │
          │  rerank)         │               │  response}       │
          └────────┬─────────┘               └────────┬─────────┘
                   ▼                                  ▼
              return cached                      return fresh
```

### The Two Datastores

A semantic cache is **always two stores working together**:

| Store | Holds | Used For |
|---|---|---|
| Vector index (e.g., Redis VSS, pgvector, Pinecone) | `(id, embedding, query_text)` | Fast nearest-neighbour search |
| Key-value store (e.g., Redis, DynamoDB) | `id → full response payload` | O(1) lookup of the cached response |

You *can* store the response inside the vector record's metadata, but separating them is cleaner: vector indexes are tuned for ANN search, not large-blob retrieval.

---

## Core Components

### 1. Embedder

Turns text into a vector. Choice matters — see [Similarity Thresholds & Embedding Models](#similarity-thresholds--embedding-models).

### 2. Vector Index

Performs approximate nearest neighbour (ANN) search. Common choices:

| Backend | When to Use |
|---|---|
| Redis VSS / Stack | Already running Redis; want one box for cache + vector |
| pgvector | Postgres-native; transactional consistency with app data |
| Pinecone, Weaviate, Qdrant | Dedicated, managed, billions of vectors |
| FAISS in-process | Single-node, read-mostly, no network hop |

### 3. Response Store

KV store for the actual cached response (text + token counts + metadata).

### 4. Similarity Threshold (τ)

The cosine-similarity floor for accepting a hit. Typical values: **0.92–0.97**. Lower = more hits but more wrong answers; higher = safer but lower hit rate.

### 5. Eviction & TTL

Vector indexes don't expire automatically — you need a janitor that deletes both the vector and the response after TTL.

### 6. Optional: Verifier

A cheap second-stage check (e.g., a small reranker or a Haiku-as-judge call) that confirms the cached answer actually matches the new query. Eliminates most false positives.

---

## Building a Semantic Cache

### Minimal Implementation (Anthropic SDK + Redis VSS)

```javascript
import Anthropic from "@anthropic-ai/sdk";
import { createClient, SchemaFieldTypes, VectorAlgorithms } from "redis";
import { randomUUID, createHash } from "crypto";

const anthropic = new Anthropic();
const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

const INDEX = "semcache_idx";
const PREFIX = "semcache:";
const DIM = 1536; // depends on embedder

// Create index once at boot
async function ensureIndex() {
  try {
    await redis.ft.create(
      INDEX,
      {
        embedding: {
          type: SchemaFieldTypes.VECTOR,
          ALGORITHM: VectorAlgorithms.HNSW,
          TYPE: "FLOAT32",
          DIM,
          DISTANCE_METRIC: "COSINE",
        },
        query: { type: SchemaFieldTypes.TEXT },
        namespace: { type: SchemaFieldTypes.TAG },
      },
      { ON: "HASH", PREFIX },
    );
  } catch (e) {
    if (!String(e).includes("Index already exists")) throw e;
  }
}

async function embed(text) {
  // Use any embedding provider; this example uses OpenAI's small model
  const r = await fetch("https://api.openai.com/v1/embeddings", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "text-embedding-3-small",
      input: text,
    }),
  }).then((r) => r.json());
  return Float32Array.from(r.data[0].embedding);
}

async function lookup(query, namespace, threshold = 0.94) {
  const vec = await embed(query);
  const results = await redis.ft.search(
    INDEX,
    `(@namespace:{${namespace}})=>[KNN 1 @embedding $V AS score]`,
    {
      PARAMS: { V: Buffer.from(vec.buffer) },
      DIALECT: 2,
      RETURN: ["score", "query", "response"],
    },
  );
  if (!results.documents.length) return null;
  const hit = results.documents[0].value;
  // Redis returns cosine *distance*; similarity = 1 - distance
  const similarity = 1 - parseFloat(hit.score);
  if (similarity < threshold) return null;
  return {
    response: JSON.parse(hit.response),
    similarity,
    matched: hit.query,
  };
}

async function store(query, namespace, response, ttlSeconds = 3600) {
  const vec = await embed(query);
  const id = randomUUID();
  const key = `${PREFIX}${id}`;
  await redis.hSet(key, {
    embedding: Buffer.from(vec.buffer),
    query,
    namespace,
    response: JSON.stringify(response),
  });
  await redis.expire(key, ttlSeconds);
}

export async function complete(query, { namespace = "default", threshold = 0.94 } = {}) {
  const hit = await lookup(query, namespace, threshold);
  if (hit) {
    return { ...hit.response, source: "semantic_cache", similarity: hit.similarity };
  }

  const msg = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    messages: [{ role: "user", content: query }],
  });

  const response = {
    text: msg.content[0].text,
    usage: msg.usage,
  };
  await store(query, namespace, response);
  return { ...response, source: "llm" };
}
```

### Integration Test

```javascript
await ensureIndex();

const a = await complete("How do I reset my password?");
// → source: "llm" (cold cache)

const b = await complete("I forgot my password, how do I reset it?");
// → source: "semantic_cache", similarity: 0.95+
```

---

## Similarity Thresholds & Embedding Models

### Picking the Threshold

The threshold is the most important knob. Calibrate it on real production traffic:

```
threshold sweep on 5,000 query pairs labelled "same intent y/n"
─────────────────────────────────────────────────────────────────
τ      hit rate    false-positive rate    avg savings
0.85   89%         18%        ← too loose (wrong answers)
0.90   78%         7%
0.92   71%         3%         ← typical sweet spot
0.94   62%         1.2%
0.96   48%         0.4%
0.98   22%         0.05%      ← too strict (low ROI)
```

**Rules of thumb:**

- Start at **0.93–0.94** for general chatbots.
- Use **0.96+** for compliance/medical/legal contexts where wrong answers carry liability.
- Use **0.88–0.91** for FAQ bots whose answer set is small and tightly clustered.
- **Never** use a fixed threshold without measuring false-positive rate on labelled data.

### Embedding Model Choice

| Model | Dim | Latency | Cost / 1M tok | Best For |
|---|---|---|---|---|
| `text-embedding-3-small` | 1536 | ~80ms | $0.02 | General-purpose semantic cache |
| `text-embedding-3-large` | 3072 | ~150ms | $0.13 | Higher recall, multilingual |
| `voyage-3-lite` | 512 | ~60ms | $0.02 | Cost-sensitive, English |
| `cohere embed-multilingual-v3` | 1024 | ~120ms | $0.10 | Non-English / mixed-language traffic |
| `bge-small-en` (self-hosted) | 384 | ~10ms (GPU) | infra-only | High-volume, no per-token cost |

**Critical:** Never change embedding models without re-embedding the entire cache. Different models live in incompatible vector spaces.

---

## Cache Key Design

A query alone is rarely enough. Cached responses depend on context that surface text doesn't capture.

### Namespacing

Always namespace by anything that changes the correct answer:

```javascript
const namespace = [
  tenantId,           // multi-tenant isolation — REQUIRED
  locale,             // "en-US" vs "fr-FR"
  productVersion,     // pricing/feature changes between v3 and v4
  userRole,           // admin sees more than viewer
  modelId,            // claude-opus-4-7 vs claude-haiku-4-5
].join(":");
```

A semantic cache without tenant isolation is a **data leak waiting to happen** — Tenant A's query could match Tenant B's cached private answer.

### Excluding Personal Data from the Key

Embed the *intent*, not the *identity*:

```
Bad:  "What's the balance of account 4521-8839?"
       ↑ embedding becomes account-specific, never reused

Good: extract intent → "account balance lookup"
      embed "account balance lookup"
      cache hit → call LLM with the user's actual account
```

The cache stores the *answer template*, not the personalised answer. The personalised step happens after the cache hit.

---

## Negative Caching & Stale-Hit Prevention

### The False-Positive Problem

```
Query 1: "How do I cancel my subscription?"        → cached
Query 2: "How do I cancel an annual subscription?" → similarity = 0.94
                                                     hit returned
     ↑ but the correct answer differs (annual has prorating rules)
```

Two defences:

### 1. Verification Layer (cheap re-check)

```javascript
async function verifyHit(query, cachedQuery) {
  const r = await anthropic.messages.create({
    model: "claude-haiku-4-5-20251001",
    max_tokens: 10,
    messages: [
      {
        role: "user",
        content:
          `Are these two questions asking for the *exact same* answer?\n` +
          `A: ${cachedQuery}\nB: ${query}\nReply with only YES or NO.`,
      },
    ],
  });
  return r.content[0].text.trim().toUpperCase().startsWith("YES");
}
```

Cost: ~$0.0001 per verification — still ~100× cheaper than the avoided Sonnet/Opus call. Run it whenever the similarity is in the "grey zone" (0.92–0.96).

### 2. Negative Cache

When the LLM produces "I don't have information about that," cache that **non-answer** too. Otherwise the same unanswerable query hits the LLM forever.

### 3. Invalidation on Source Change

If the cached answer is grounded in a document, tag the cache entry with the document's content hash. When the document changes, invalidate by tag.

---

## AI-Specific Use Cases

### 1. Customer Support Chatbots

The single highest-ROI deployment. Support traffic is dominated by ~50–200 recurring intents.

```
Hit rate:     65–85%
Cost cut:     60–80%
Latency cut:  P95 from 2–4s to 200–400ms
```

### 2. RAG Question-Answering

Cache the *final synthesised answer*, not the retrieval step. Because the same question fetches the same chunks fetches the same answer, caching at the top eliminates everything below it.

```
"What is our PTO policy?"     → semantic cache hit
                                  skips: embed, vector search,
                                         rerank, prompt assembly,
                                         LLM call
```

### 3. Code Assistants

Cache deterministic completions ("write a Python function to reverse a string") but **not** stochastic ones (creative refactors). Use `temperature == 0` as a cache eligibility check.

### 4. Translation Pipelines

Cache `(source_text, target_language)` → translation. Semantic similarity matches paraphrases of the same source text.

---

## Advanced Patterns

### Hierarchical Semantic Cache

Two indexes: a small fast in-memory one (recent N=10K queries) and a large persistent one. Check the hot index first.

```
L1: in-process FAISS, 10K vectors, sub-ms search
L2: Redis VSS,        10M vectors, 5–15ms search
L3: LLM
```

Promote L2 hits into L1 on access.

### Semantic Cache + Prompt Cache

These are **complementary**, not alternatives. Stack them:

```
1. Semantic cache hit?         → return                (skip LLM entirely)
2. Miss → call LLM with prompt cache enabled
   → 90% of input tokens hit Anthropic's prompt cache
   → response goes into semantic cache for next time
```

### Adaptive Threshold by Query Type

A single global threshold is a crude tool. Detect query category and choose τ:

```javascript
const TAU = {
  factual_lookup: 0.97,   // strict — wrong fact = wrong answer
  intent_classify: 0.90,  // loose — paraphrases all map to same intent
  creative: null,         // never cache
  pii_present: null,      // never cache
};
```

### Time-Decay Scoring

Down-weight older cache entries so the system gradually replaces them with fresher equivalents:

```
score = cosine_similarity − λ × age_in_hours / TTL_hours
```

Useful when source data slowly drifts (pricing, policies).

---

## Monitoring & Observability

### Required Metrics

| Metric | Why It Matters | Alert If |
|---|---|---|
| `cache_hit_rate` | Core KPI | < 50% (something broke) |
| `cache_false_positive_rate` | Quality | > 2% (lower threshold or add verifier) |
| `lookup_latency_p99` | UX | > 200ms (resize index / shard) |
| `index_size_vectors` | Capacity planning | growth diverges from query growth |
| `cost_avoided_usd_per_day` | ROI | trending down (model/threshold drift) |
| `eviction_rate` | TTL appropriateness | too high = TTL too short |

### Sample Dashboard Queries (Prometheus)

```
# hit rate over 5min
sum(rate(semcache_hits_total[5m]))
  / sum(rate(semcache_lookups_total[5m]))

# cost avoided per hour
sum(rate(semcache_hits_total[1h])) * 3600 * 0.012   # $0.012 per avoided call

# similarity distribution of hits (p50, p90, p99)
histogram_quantile(0.99, rate(semcache_hit_similarity_bucket[5m]))
```

### Shadow Evaluation

Continuously sample 1% of cache hits, also call the LLM, and compare. If divergence rises, the threshold has drifted.

---

## Integration with Other Patterns

### + Cache-Aside (hash) — first

Run hash cache *in front of* semantic cache. Exact repeats skip embedding altogether.

```
hash hit (1ms) → return
hash miss → semantic lookup (100ms) → return / fall through to LLM
```

### + Circuit Breaker

Wrap the embedder and vector index in circuit breakers. On open, **fall through** to direct LLM calls — never fail the user request because the cache is degraded.

### + RAG

Place the semantic cache between the user and the RAG pipeline (not inside it). RAG itself becomes the cache miss path.

### + Prompt Cache

Stack as described above. They optimise different stages: semantic skips the call entirely, prompt cache accelerates the call when it happens.

### + Cost Guardrails

A semantic cache *changes* your cost model — recompute budgets after deployment, then attach guardrails to the residual LLM traffic.

---

## Production Best Practices

### 1. Always Multi-Tenant Namespace

Even in single-tenant systems, namespace by environment (`prod`, `staging`) so test traffic never poisons prod responses.

### 2. Cache the Sanitised Output

Strip PII, names, addresses, IDs from the response *before* caching. Otherwise a cache hit for User A could leak User B's data.

### 3. Cap Response Size

Reject caching responses > N KB (typical: 32 KB). Massive responses kill memory and rarely repeat.

### 4. TTL Defaults

```
factual content (docs, FAQs):  24h–7d
policy answers:                1h–6h  (changes need to propagate)
session summaries:             never cache
streaming responses:           cache only the final assembled text
```

### 5. Don't Semantic-Cache When `temperature > 0.3`

Stochastic responses will never match prior outputs reliably; you'll cache one sample and serve it forever, defeating the purpose of temperature.

### 6. Versioning

Stamp every cache entry with `(embedder_version, model_version, prompt_version)`. Bumping any one of them must invalidate.

### 7. Pre-Warm

At deploy time, replay the top-N intents through the system to populate the cache before traffic hits. Prevents launch-time stampede.

---

## Common Mistakes

1. **No false-positive measurement.** "Hit rate is 80%" means nothing if 30% of those hits are wrong.
2. **Sharing namespace across tenants.** Catastrophic data leak.
3. **Caching personalised responses.** "Your balance is $5,031" served to another user.
4. **Embedding model drift.** Upgrading the embedder without rebuilding the index — every lookup becomes a near-random hit.
5. **No TTL.** Cache fills, eviction is random, oldest stale answers persist forever.
6. **Threshold by vibes.** Pick τ on labelled data, not by intuition.
7. **Caching streaming responses prematurely.** Store only after the stream completes successfully.
8. **Treating semantic cache as a database.** It's a hint store. Source of truth lives elsewhere.

---

## Interview Cheat Sheet

**Q: What's the difference between cache-aside and semantic cache?**
A: Cache-aside keys by exact bytes (or hash thereof). Semantic cache keys by meaning, via embedding + ANN search. Hash cache hit rate on natural-language traffic is typically <10%; semantic cache reaches 60–85% on the same workload.

**Q: How do you avoid wrong answers from a semantic cache?**
A: Three layers — (1) calibrate the similarity threshold on labelled pairs, (2) add a cheap verifier (Haiku-as-judge) for grey-zone hits, (3) namespace aggressively so context-dependent answers never collide.

**Q: When should you NOT use a semantic cache?**
A: Stochastic outputs (high temperature), per-user personalised content, real-time data, regulated outputs that must be re-derived for audit, and queries with PII embedded in them.

**Q: What does the cache cost?**
A: Embedding latency (~80ms) + vector search (~5–15ms) + storage. Net win unless your hit rate is < ~15%, in which case the embedding-on-miss tax outweighs the savings.

**Q: How does this interact with prompt caching?**
A: They compose. Semantic cache eliminates the LLM call. Prompt cache makes the call cheap when it does happen (90%+ input-token discount on Anthropic). Use both.

**Q: Vector DB choice?**
A: Redis VSS if you already run Redis. pgvector for transactional consistency. Pinecone/Qdrant/Weaviate at >10M vectors. FAISS in-process for read-mostly single-node.

---

**Document Version:** 1.0
**Last Updated:** May 2026
