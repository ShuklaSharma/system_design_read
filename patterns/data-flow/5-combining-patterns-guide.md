# Combining Patterns for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why Combine Patterns?](#why-combine-patterns)
3. [Common Pattern Combinations](#common-pattern-combinations)
4. [AI Platform Reference Architecture](#ai-platform-reference-architecture)
5. [Implementation: Full Stack AI Pipeline](#implementation-full-stack-ai-pipeline)
6. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
7. [Migration Strategy](#migration-strategy)
8. [Real-World Case Studies](#real-world-case-studies)
9. [Combination Decision Guide](#combination-decision-guide)

---

## Introduction

### Why Patterns Rarely Work Alone

Each resilience and architecture pattern solves a specific problem in isolation. But production AI systems face *multiple* problems simultaneously: cascading failures, data inconsistency, write/read bottlenecks, audit requirements, cost explosions, and provider outages — often all at once.

The real skill is knowing which patterns to combine, in what order to introduce them, and how they interact with each other. A circuit breaker without retry is too aggressive. Retry without backpressure causes thundering herds. Event sourcing without CQRS produces slow queries. Sagas without idempotency cause double-charges.

**Analogy:** A single safety feature in a car (seatbelt) helps, but the real safety comes from the combination: seatbelt + airbags + ABS + crumple zones + lane assist. Each pattern addresses a different failure mode. Combined, they create a system that degrades gracefully under almost any condition.

### Pattern Interaction Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AI APPLICATION LAYER                             │
│                                                                     │
│  User Request → [Backpressure] → [Bulkhead] → [Circuit Breaker]    │
│                                                   ↓                │
│                                              [Retry + Backoff]     │
│                                                   ↓                │
│                 ┌─────────────────────────────────┘                │
│                 ↓                                                   │
│           [Saga / 2PC]  ←── Coordinates cross-service writes       │
│                 ↓                                                   │
│      ┌──────────┴──────────┐                                       │
│      ↓                     ↓                                       │
│  Write Side            Read Side                                   │
│  [Event Sourcing]      [CQRS]                                      │
│      ↓                     ↓                                       │
│  Event Store         Read Models (ClickHouse, Redis, ES)           │
│      ↓                                                             │
│  [Cache-Aside] ← Caches read model results                         │
│      ↓                                                             │
│  [Sharding] ← Distributes data across nodes                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Why Combine Patterns?

### What Each Pattern Protects

| Pattern | Protects Against |
|---|---|
| **Backpressure** | Overload, cost explosion, OOM |
| **Bulkhead** | Cascading failures, noisy neighbors |
| **Circuit Breaker** | Slow provider, cascading timeouts |
| **Retry + Backoff** | Transient failures, rate limits |
| **Cache-Aside** | Repeated API cost, high latency |
| **Sharding** | Data scale limits, single-node bottleneck |
| **Saga** | Partial failures in multi-step pipelines |
| **2PC** | Data inconsistency across stores |
| **CQRS** | Read/write contention, schema mismatch |
| **Event Sourcing** | Lost history, lack of reproducibility |

### Gaps When Used Alone

```
Circuit Breaker alone:
  ✅ Stops cascading failures
  ❌ Does NOT retry — failed requests are just lost
  ❌ Does NOT limit total concurrent load

Retry alone:
  ✅ Handles transient failures
  ❌ Does NOT stop thundering herd (all retries at same time)
  ❌ Does NOT protect downstream from overload

Cache-Aside alone:
  ✅ Reduces API calls for repeated queries
  ❌ Does NOT handle what happens on cache miss under load (stampede)
  ❌ Does NOT prevent cascading failures when cache backend is down

Saga alone:
  ✅ Handles multi-step failure compensation
  ❌ Does NOT prevent concurrent sagas overwhelming downstream
  ❌ Does NOT protect against provider outages mid-saga

Event Sourcing alone:
  ✅ Provides full audit history
  ❌ Read performance degrades with event count (needs CQRS)
  ❌ Does NOT coordinate multi-service writes (needs Saga or 2PC)
```

---

## Common Pattern Combinations

### Combination 1: Retry + Circuit Breaker + Backoff (The Reliability Trio)

The most fundamental combination for any external AI API call.

```javascript
const CircuitBreaker = require('opossum');

class ReliableAIClient {
  constructor({ openai }) {
    this.openai = openai;

    // Circuit breaker wraps the API call
    this.breaker = new CircuitBreaker(
      async (messages) => this._callAPI(messages),
      {
        timeout: 30000,                  // Fail if > 30s
        errorThresholdPercentage: 30,    // Open if > 30% errors
        resetTimeout: 60000              // Try again after 60s
      }
    );
  }

  async _callAPI(messages) {
    return this.openai.chat.completions.create({
      model: 'gpt-4',
      messages
    });
  }

  async complete(messages, { maxRetries = 3 } = {}) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        // Circuit breaker fires the call
        return await this.breaker.fire(messages);

      } catch (error) {
        const isRetryable = error.status === 429 || error.status === 503;
        const circuitOpen = error.code === 'EOPENBREAKER';

        if (circuitOpen) {
          // Circuit is open — don't retry, fail fast
          throw new Error('AI provider circuit open. Try again in 60 seconds.');
        }

        if (!isRetryable || attempt === maxRetries) throw error;

        // Exponential backoff with jitter
        const delay = Math.min(
          (Math.pow(2, attempt) * 1000) + Math.random() * 500,
          30000
        );
        console.log(`Attempt ${attempt} failed (${error.status}). Retrying in ${delay}ms`);
        await new Promise(r => setTimeout(r, delay));
      }
    }
  }
}
```

### Combination 2: Cache-Aside + Circuit Breaker (Cache as Fallback)

When the AI provider's circuit is open, serve from cache. Cache becomes a resilience layer, not just a performance optimization.

```javascript
class CacheFirstResilience {
  constructor({ cache, aiClient }) {
    this.cache = cache;
    this.aiClient = aiClient;
  }

  async complete(prompt, options = {}) {
    const cacheKey = `llm:${hashString(prompt)}`;

    // Always check cache first
    const cached = await this.cache.get(cacheKey);
    if (cached) return { ...cached, source: 'cache' };

    try {
      // Try live API (with circuit breaker + retry inside aiClient)
      const response = await this.aiClient.complete([
        { role: 'user', content: prompt }
      ]);

      // Store successful response in cache
      await this.cache.set(cacheKey, response, options.ttl || 3600);
      return { ...response, source: 'api' };

    } catch (error) {
      if (error.message.includes('circuit open')) {
        // Provider is down — check if we have ANY cached version (even stale)
        const staleCache = await this.cache.getStale(cacheKey);
        if (staleCache) {
          console.warn('Serving stale cache — circuit open');
          return { ...staleCache, source: 'stale_cache', warning: 'May be outdated' };
        }
      }
      throw error;
    }
  }
}
```

### Combination 3: Bulkhead + Backpressure + Cache (Load Management Stack)

Control incoming load at multiple levels before hitting the expensive AI provider.

```javascript
const pLimit = require('p-limit');

class LoadManagedAIService {
  constructor({ cache, aiClient, config }) {
    this.cache = cache;
    this.aiClient = aiClient;

    // Level 1: Bulkhead — max concurrent AI calls
    this.concurrencyLimiter = pLimit(config.maxConcurrent || 20);

    // Level 2: Backpressure — max queue size
    this.maxQueueDepth = config.maxQueueDepth || 40;
  }

  async complete(prompt) {
    // Level 0: Cache check (free — no queuing needed)
    const cacheKey = `llm:${hashString(prompt)}`;
    const cached = await this.cache.get(cacheKey);
    if (cached) return cached;

    // Level 1: Backpressure — reject if queue too deep
    const queueDepth = this.concurrencyLimiter.pendingCount;
    if (queueDepth > this.maxQueueDepth) {
      throw Object.assign(new Error('Service at capacity. Try again in a moment.'), {
        status: 429,
        retryAfterMs: 5000
      });
    }

    // Level 2: Bulkhead — limit concurrency through the rate-limited API
    return this.concurrencyLimiter(async () => {
      // Re-check cache (another request may have populated it while we waited)
      const recheckCache = await this.cache.get(cacheKey);
      if (recheckCache) return recheckCache;

      const response = await this.aiClient.complete(prompt);
      await this.cache.set(cacheKey, response, 3600);
      return response;
    });
  }

  getStatus() {
    return {
      active: this.concurrencyLimiter.activeCount,
      queued: this.concurrencyLimiter.pendingCount,
      maxConcurrent: this.concurrencyLimiter.concurrency,
      maxQueue: this.maxQueueDepth,
      status: this.concurrencyLimiter.pendingCount > this.maxQueueDepth * 0.8
        ? 'HIGH_LOAD' : 'HEALTHY'
    };
  }
}
```

### Combination 4: CQRS + Event Sourcing + Cache (Read Performance Stack)

The canonical combination for high-performance AI platforms with audit requirements.

```javascript
class EventSourcedCQRSWithCache {
  constructor({ eventStore, repository, queryDB, cache }) {
    this.events = eventStore;
    this.repo = repository;
    this.queryDB = queryDB;   // ClickHouse or similar
    this.cache = cache;       // Redis

    // Start projection worker
    this.projector = new ReadModelProjector({ eventStore, queryDB });
    this.projector.start();
  }

  // ─── COMMAND SIDE ───────────────────────────────────────
  async createModel(command) {
    const model = MLModelAggregate.create(command.modelId, command);
    await this.repo.save(model);
    // Events automatically flow to projector → read model updated
    return { modelId: command.modelId };
  }

  async completeTraining(command) {
    const model = await this.repo.load(command.modelId);
    model.completeTraining(command);
    await this.repo.save(model);

    // Invalidate cached queries for this model
    await this.cache.del(`model:${command.modelId}:*`);
    return { modelId: command.modelId };
  }

  // ─── QUERY SIDE ─────────────────────────────────────────
  async getModelHistory(modelId) {
    // Historical event data — from event store directly
    const events = await this.events.getEvents(modelId);
    return events.map(e => ({ type: e.type, data: e.data, timestamp: e.timestamp }));
  }

  async getDashboard(tenantId, dateRange) {
    // Cache dashboard queries (they're expensive but stable)
    const cacheKey = `dashboard:${tenantId}:${dateRange.start}:${dateRange.end}`;
    const cached = await this.cache.get(cacheKey);
    if (cached) return cached;

    // Query the CQRS read model (not the event store — too slow for aggregation)
    const data = await this.queryDB.query(
      `SELECT model_id, SUM(tokens_used), AVG(latency_ms), COUNT(*) as inference_count
       FROM inference_metrics
       WHERE tenant_id = {tenantId:String}
         AND timestamp BETWEEN {start:DateTime} AND {end:DateTime}
       GROUP BY model_id`,
      { tenantId, ...dateRange }
    );

    await this.cache.set(cacheKey, data, 300); // Cache for 5 minutes
    return data;
  }

  async getModelAtTime(modelId, timestamp) {
    // Time-travel query — only possible with Event Sourcing
    return this.repo.loadAt(modelId, timestamp);
  }
}
```

### Combination 5: Saga + Bulkhead + Retry (Resilient Multi-Step Pipeline)

The pattern for AI generation pipelines that must be both reliable and not overwhelm providers.

```javascript
class ResilientAIPipeline {
  constructor({ sagaOrchestrator, paymentService, llmClient, imageClient }) {
    this.saga = sagaOrchestrator;

    // Each service in the saga has its own bulkhead
    this.bulkheads = {
      payment: pLimit(50),      // Stripe can handle high concurrency
      llm: pLimit(10),          // OpenAI has rate limits
      image: pLimit(3)          // Image generation is expensive
    };

    // Each service has its own retry policy
    this.retryPolicies = {
      payment: { maxRetries: 3, backoff: 'exponential' },
      llm: { maxRetries: 5, backoff: 'exponential_jitter' },
      image: { maxRetries: 2, backoff: 'linear', delayMs: 10000 }
    };
  }

  async runContentGenerationSaga(jobId, { userId, prompt, imagePrompt, amount }) {
    return this.saga.execute(jobId, [
      {
        name: 'chargePayment',
        execute: async (ctx) =>
          this.bulkheads.payment(() =>
            this._withRetry('payment', () =>
              paymentService.charge(ctx.userId, ctx.amount)
            )
          ),
        compensate: async (ctx) =>
          paymentService.refund(ctx.chargeId)
      },
      {
        name: 'generateText',
        execute: async (ctx) =>
          this.bulkheads.llm(() =>
            this._withRetry('llm', () =>
              llmClient.complete(ctx.prompt)
            )
          ),
        compensate: async (ctx) =>
          llmClient.discard(ctx.textId)
      },
      {
        name: 'generateImage',
        execute: async (ctx) =>
          this.bulkheads.image(() =>
            this._withRetry('image', () =>
              imageClient.generate(ctx.imagePrompt)
            )
          ),
        compensate: async (ctx) =>
          imageClient.discard(ctx.imageId)
      }
    ], { userId, prompt, imagePrompt, amount });
  }

  async _withRetry(service, fn) {
    const policy = this.retryPolicies[service];
    for (let attempt = 1; attempt <= policy.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (err) {
        if (attempt === policy.maxRetries) throw err;
        const delay = this._calculateDelay(policy, attempt);
        await new Promise(r => setTimeout(r, delay));
      }
    }
  }

  _calculateDelay(policy, attempt) {
    if (policy.backoff === 'exponential') return Math.pow(2, attempt) * 1000;
    if (policy.backoff === 'exponential_jitter') return Math.pow(2, attempt) * 1000 + Math.random() * 500;
    if (policy.backoff === 'linear') return policy.delayMs * attempt;
    return 1000;
  }
}
```

---

## AI Platform Reference Architecture

This is the full pattern stack for a production-grade multi-tenant AI SaaS platform.

```
┌──────────────────────────────────────────────────────────────────┐
│  CLIENT / API GATEWAY                                            │
│  ┌──────────────────┐                                           │
│  │  Rate Limiter    │ ← Backpressure (protect from DDoS/spikes) │
│  │  Auth / Tenant   │                                           │
│  └──────────────────┘                                           │
└──────────────────────────────┬───────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│  COMMAND SERVICE                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Bulkhead    │  │  Saga        │  │  2PC (for atomic      │  │
│  │  (per tenant)│  │  Orchestrator│  │  config changes)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│         ↓                  ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Event Store (Append-only, immutable)                    │   │
│  │  ← Event Sourcing + CQRS write side                      │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬───────────────────────────────────┘
                               ↓ event stream
┌──────────────────────────────────────────────────────────────────┐
│  PROJECTION LAYER (CQRS Read Side)                               │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │  Dashboard       │  │  Billing Model   │                     │
│  │  Projection      │  │  Projection      │                     │
│  │  (ClickHouse)    │  │  (PostgreSQL)    │                     │
│  └──────────────────┘  └──────────────────┘                     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Cache Layer (Redis)   ← Cache-Aside on top of read models│   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│  EXTERNAL AI PROVIDER CALLS                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Circuit     │  │  Retry +     │  │  Cache-Aside         │  │
│  │  Breaker     │  │  Backoff     │  │  (semantic cache)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│  DATA LAYER                                                      │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  Sharding (vector DB, document store, per tenant)       │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Implementation: Full Stack AI Pipeline

```javascript
// Putting it all together: a resilient, observable, scalable AI inference service

class ProductionAIInferenceService {
  constructor({
    redis, clickhouse, eventStore,
    openai, anthropic,            // Primary + fallback providers
    shardedVectorDB
  }) {
    // ── Reliability layer ──────────────────────────────────────
    this.primaryClient = new ReliableAIClient({ openai });       // Retry + Circuit Breaker
    this.fallbackClient = new ReliableAIClient({ llm: anthropic });

    // ── Load management ────────────────────────────────────────
    this.bulkheads = {
      premium: pLimit(50),
      free: pLimit(10)
    };

    // ── Caching layer ──────────────────────────────────────────
    this.semanticCache = new SemanticCache({ redis, threshold: 0.93 });

    // ── CQRS + Event Sourcing ──────────────────────────────────
    this.eventStore = eventStore;
    this.readModel = new InferenceReadModel({ clickhouse });

    // ── Sharded storage ────────────────────────────────────────
    this.vectorDB = shardedVectorDB;
  }

  async infer(request) {
    const {
      requestId, tenantId, userId, tier,
      prompt, context, modelPreference
    } = request;

    // ── Step 1: Backpressure check ─────────────────────────────
    const limiter = this.bulkheads[tier] || this.bulkheads.free;
    if (limiter.pendingCount > (tier === 'premium' ? 100 : 20)) {
      throw Object.assign(new Error('Service busy'), { status: 429 });
    }

    return limiter(async () => {
      // ── Step 2: Semantic cache check ───────────────────────────
      const cached = await this.semanticCache.get(prompt);
      if (cached) {
        await this._logInference(requestId, tenantId, userId, prompt, cached, 'cache');
        return { ...cached, source: 'cache' };
      }

      // ── Step 3: RAG context retrieval (sharded vector DB) ──────
      let ragContext = '';
      if (context?.useRAG) {
        const queryEmbedding = await this._embed(prompt);
        const chunks = await this.vectorDB.tenantSearch(tenantId, queryEmbedding, 5);
        ragContext = chunks.map(c => c.text).join('\n\n');
      }

      // ── Step 4: Call AI provider (with fallback) ───────────────
      const messages = [
        ...(ragContext ? [{ role: 'system', content: `Context:\n${ragContext}` }] : []),
        { role: 'user', content: prompt }
      ];

      let response;
      try {
        response = await this.primaryClient.complete(messages);
        response.provider = 'openai';
      } catch (err) {
        console.warn('Primary provider failed, using fallback:', err.message);
        response = await this.fallbackClient.complete(messages);
        response.provider = 'anthropic';
      }

      // ── Step 5: Cache the result ───────────────────────────────
      await this.semanticCache.set(prompt, response);

      // ── Step 6: Log to event store (Event Sourcing) ────────────
      await this._logInference(requestId, tenantId, userId, prompt, response, 'api');

      return response;
    });
  }

  async _logInference(requestId, tenantId, userId, prompt, response, source) {
    // Append to event store — immutable record for audit + CQRS projection
    await this.eventStore.append(requestId, 'InferenceSession', [{
      type: 'InferenceCompleted',
      data: {
        tenantId, userId,
        promptHash: hashString(prompt),
        tokensIn: response.usage?.prompt_tokens,
        tokensOut: response.usage?.completion_tokens,
        latencyMs: response.latencyMs,
        provider: response.provider,
        source  // 'cache' or 'api'
      }
    }], -1);
    // Read model projector picks this up asynchronously → dashboard updated
  }
}
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Over-Engineering from Day One

```
❌ WRONG: Building the full reference architecture on day 1
  Day 1 startup adds: Event Sourcing + CQRS + Saga + Sharding + 2PC
  Result: 6 months to launch, hard to debug, team overwhelmed

✅ RIGHT: Start simple, add patterns as pain emerges
  Month 1:  Basic retry + circuit breaker (immediate need)
  Month 3:  Cache-Aside (costs growing, latency complaints)
  Month 6:  Bulkhead (noisy tenant problem)
  Month 9:  CQRS (dashboard slow, write conflicts)
  Month 12: Event Sourcing (audit requirement, debugging pain)
  Month 18: Saga (multi-step failures, payment inconsistency)
  Month 24: Sharding (data scale hitting limits)
```

### Anti-Pattern 2: Wrong Pattern for the Problem

```
❌ Using 2PC for a 5-minute AI generation pipeline
  → 2PC holds locks for the entire duration
  → Other transactions blocked for 5 minutes
  ✅ Use Saga instead (no locks, compensations)

❌ Using Saga for a 2-store atomic model deployment
  → Saga is eventually consistent → brief inconsistency window
  → During that window: serving wrong model
  ✅ Use 2PC instead (atomic, immediate consistency)

❌ Using Event Sourcing without CQRS
  → Read queries replay thousands of events → unacceptable latency
  ✅ Always pair Event Sourcing with CQRS read models

❌ Applying Cache-Aside to personalized AI responses
  → User A's cached response served to User B
  ✅ Only cache deterministic, non-personal responses
```

### Anti-Pattern 3: Patterns Without Monitoring

```
❌ Circuit breaker with no alerts
  → Circuit opens, team doesn't know for hours
  → Users see failures silently

❌ Saga with no dead-letter queue
  → Compensation fails → money charged, no product delivered
  → No one is paged

❌ Cache-Aside with no hit rate tracking
  → Cache is misconfigured, 0% hit rate
  → Paying full API cost but thinking you're optimized

✅ ALWAYS add monitoring when adding a pattern
  → Circuit breaker state → alert on open
  → Saga compensation failures → alert immediately
  → Cache hit rate → alert if drops below threshold
  → Event store lag → alert if projections fall behind
```

---

## Migration Strategy

### Introducing Patterns Safely

```javascript
// Use feature flags to introduce patterns incrementally

class FeatureFlaggedAIService {
  constructor({ baseService, newService, featureFlags }) {
    this.base = baseService;
    this.new = newService;
    this.flags = featureFlags;
  }

  async complete(request) {
    // Shadow mode: run both, compare, return original result
    if (this.flags.isEnabled('cqrs_shadow_mode', request.tenantId)) {
      const [original, shadow] = await Promise.allSettled([
        this.base.complete(request),
        this.new.complete(request)
      ]);
      this._compareShadow(original, shadow, request);
      return original.value;
    }

    // Canary: send X% of traffic to new pattern
    if (this.flags.isEnabled('cqrs_canary', request.tenantId)) {
      return this.new.complete(request);
    }

    return this.base.complete(request);
  }
}
```

### Rollout Phases

```
Phase 1 — Shadow (0% live traffic)
  New pattern runs in parallel, results compared but not returned
  Goal: Validate correctness, measure performance impact

Phase 2 — Canary (5% of traffic, internal users first)
  Monitor: error rate, latency, consistency
  Rollback trigger: error rate > 1% above baseline

Phase 3 — Gradual rollout (5% → 25% → 50% → 100%)
  Weekly increments with monitoring at each stage
  Rollback trigger: any regression in key metrics

Phase 4 — Full rollout
  Old code path removed after 2 weeks of stable operation
```

---

## Real-World Case Studies

### Case Study 1: AI SaaS Scaling Journey (0 → $10M ARR)

```
$0–$100K ARR — No patterns:
  Basic Express app, single PostgreSQL, direct OpenAI calls
  Problems: occasional 500s, no retry, one outage per week

$100K–$1M ARR — Reliability patterns added:
  Added: Retry + Circuit Breaker + Cache-Aside
  Result: Outages dropped from weekly to monthly
  API costs: -65% (cache hit rate: 73%)

$1M–$3M ARR — Load management added:
  Added: Bulkhead (premium vs free tiers), Backpressure
  Result: Black Friday survived without degradation
  Premium SLA: 99.9% (was 98.2%)

$3M–$7M ARR — Data architecture patterns:
  Added: CQRS (dashboard 38s → 0.4s), Event Sourcing (audit trail)
  Result: Enterprise sales unlocked (audit requirements met)
  DB cost: -72%

$7M–$10M ARR — Scale patterns:
  Added: Sharding (200M vectors), Saga (payment + generation pipeline)
  Result: Handled 10x growth with linear cost scaling
  Zero charge-without-delivery incidents
```

### Case Study 2: Regulated AI Platform (Healthcare)

**Patterns chosen:** Event Sourcing + 2PC + CQRS + Circuit Breaker + Bulkhead

**Why this combination:**
- Event Sourcing: FDA/HIPAA reproducibility requirements
- 2PC: Model + Feature Store + Audit Log must update atomically (patient safety)
- CQRS: Clinical dashboards cannot slow down data ingestion
- Circuit Breaker: If AI provider is slow, clinical workflows cannot block
- Bulkhead: Emergency AI queries always get capacity (cannot be starved by routine queries)

**Results:**
- ✅ Regulatory approval: passed all three FDA requirements
- ✅ Clinical workflow uptime: 99.97%
- ✅ Zero patient record inconsistencies in 2 years
- ✅ Audit response time: 3 weeks → 4 hours (all history in event store)

---

## Combination Decision Guide

### Quick Reference

```
"My AI API calls are failing intermittently"
  → Retry + Backoff first
  → Add Circuit Breaker if failures are sustained

"My AI service crashes under high load"
  → Backpressure first (reject excess requests)
  → Add Bulkhead (isolate tenants/service types)

"My AI API costs are too high"
  → Cache-Aside first (biggest ROI, fastest to implement)
  → Add semantic cache if queries are paraphrased

"One tenant is slowing down others"
  → Bulkhead (tenant isolation)
  → Add Sharding if the issue is data volume

"My dashboard is slow but writes are fast"
  → CQRS (separate read and write models)
  → Add Cache-Aside on top of read model

"I can't reproduce AI model outputs from 3 months ago"
  → Event Sourcing (you needed this from day one)
  → Pair with CQRS for readable history

"Multi-step AI pipeline leaves inconsistent state on failure"
  → Saga (compensating transactions)
  → Add 2PC if the failure is specifically between two data stores

"My vector DB can't handle the data volume"
  → Sharding (distribute across nodes)
  → Add consistent hashing for elastic scaling

"I need all of the above"
  → Follow the reference architecture
  → Add patterns in the order: Reliability → Load → Data → Scale
```

### Pattern Introduction Order

```
Week 1–4:    Retry + Circuit Breaker (always first — biggest safety impact)
Week 4–8:    Cache-Aside (biggest cost impact — 50-90% API cost reduction)
Week 8–12:   Backpressure + Bulkhead (protect under load)
Month 3–6:   CQRS (when dashboard/write conflicts emerge)
Month 6–9:   Event Sourcing (when audit/reproducibility is needed)
Month 6–9:   Saga (when multi-step failures cause inconsistency)
Month 9–12:  Sharding (when data volume hits limits)
As needed:   2PC (for strict consistency across owned data stores)
```

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
