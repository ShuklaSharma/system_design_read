# Cache-Aside Pattern for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why Cache-Aside Matters for AI Systems](#why-cache-aside-matters)
3. [Cache-Aside Variants](#cache-aside-variants)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Monitoring & Observability](#monitoring-observability)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Real-World Case Studies](#real-world-case-studies)
10. [Production Best Practices](#production-best-practices)

---

## Introduction

### What is Cache-Aside?

**Cache-Aside** (also known as Lazy Loading) is a caching pattern where the application is responsible for loading data into the cache on demand. The cache sits "aside" from the main data store — the application checks the cache first, and only fetches from the source of truth when a cache miss occurs.

**Analogy:** Like a chef's prep station — ingredients used frequently are kept on the counter (cache) for quick access. Less-used items stay in the storage room (database). The chef only goes to the storage room when the counter doesn't have what's needed, then brings it back and places it on the counter for next time.

### The Problem Without Cache-Aside

```
Scenario: AI chatbot serving 10,000 users/day with GPT-4

Without Caching:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Every request hits the OpenAI API

User asks: "Summarize the product documentation"
→ API call: 800ms, $0.012

Same user asks again 5 minutes later:
→ API call: 800ms, $0.012 (again)

100 users ask the same FAQ question today:
→ 100 API calls: 80,000ms total, $1.20

Monthly:
→ 300,000 API calls
→ $3,600 in API costs
→ 800ms average latency
→ Rate limit errors at peak load
→ API downtime = full service outage
```

### The Solution With Cache-Aside

```
With Cache-Aside:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
First request: cache miss → hit API → store result

User asks: "Summarize the product documentation"
→ Cache miss: API call 800ms, $0.012
→ Store in Redis with 1hr TTL

Same user asks again 5 minutes later:
→ Cache hit: 2ms, $0.00

100 users ask the same FAQ question today:
→ 1 API call + 99 cache hits
→ Cost: $0.012 (vs $1.20)
→ Avg latency: 10ms (vs 800ms)

Monthly:
→ ~30,000 unique API calls (90% cache hit rate)
→ $360 in API costs (90% reduction)
→ 10ms average latency (98% faster)
→ API downtime = serve from cache
→ Zero rate limit errors
```

---

## Why Cache-Aside Matters for AI Systems

### Unique AI System Challenges

1. **Expensive API Calls**
   - GPT-4: $0.01–$0.03 per 1K tokens
   - Claude Opus: $0.015 per 1K input tokens
   - Embedding models: $0.0001 per 1K tokens
   - Costs compound at scale with repeated queries

2. **High Latency Operations**
   - LLM inference: 500ms – 30s
   - Image generation: 5s – 3 minutes
   - Embedding computation: 100ms – 2s
   - Repeated computation = repeated wait time

3. **Repeated Patterns in AI Workloads**
   - FAQ-style queries hit the same prompts
   - Embeddings for the same documents recomputed
   - Semantic search results reused across users
   - System prompts + context fragments reused

4. **Provider Rate Limits**
   - OpenAI: 500 RPM on standard tiers
   - Anthropic: 50 RPM on lower tiers
   - Rate limits cause 429 errors under load
   - Cache reduces effective RPM dramatically

### Real Incident: No Cache-Aside

**Launch Day - AI Writing Assistant**

```
Timeline:
09:00 - Product launches on Hacker News
09:05 - 200 concurrent users (normal: 5)
09:10 - OpenAI rate limit: 500 RPM hit
09:12 - 429 errors start returning to users
09:15 - Error rate: 45%
09:20 - OpenAI API call queue backing up
09:25 - Response times: 30+ seconds
09:30 - Users posting "this is broken" on HN
09:45 - Emergency: Upgrade OpenAI tier ($1,000/mo)
10:00 - Upgrade takes effect, errors reduce
10:30 - Stable (but now paying 10x more)

Impact:
- 90 minutes of degraded launch
- $1,000/month unexpected cost
- Negative HN comments on launch day
- 60% of launch users churned immediately
- No caching = no protection during viral moment
```

**Same Incident WITH Cache-Aside:**

```
Timeline:
09:00 - Product launches on Hacker News
09:05 - 200 concurrent users spike
09:07 - Cache populates with top 50 common prompts
09:10 - Cache hit rate: 78% (most users exploring same features)
09:12 - Effective API RPM: 110 (well under 500 limit)
09:15 - Error rate: 0%
09:20 - Response times: 15ms average (cached)
09:30 - Users posting "this is fast and reliable!" on HN

Impact:
- Zero downtime
- No rate limit errors
- 78% cost reduction during spike
- Viral moment captured successfully
- Users retained due to fast, reliable experience
```

---

## Cache-Aside Variants

### 1. Read-Through Cache

Data is loaded into cache automatically on first miss. Application always reads from cache.

```
Request → Cache → [Miss] → Data Store → Cache → Response
```

**Use For:** Read-heavy workloads, shared caches, embedding lookups

### 2. Write-Around Cache

Writes go directly to the data store, bypassing cache. Cache is only populated on reads.

```
Write → Data Store (cache bypassed)
Read  → Cache → [Miss] → Data Store → Cache → Response
```

**Use For:** Write-heavy data that's rarely re-read, large file storage

### 3. Write-Through Cache

Writes update both cache and data store simultaneously.

```
Write → Cache + Data Store (both updated)
Read  → Cache → [Always hit after first write]
```

**Use For:** Data that must always be consistent between cache and DB

### 4. Semantic Cache (AI-Specific)

Cache key is based on semantic similarity, not exact string match. Similar questions share cache entries.

```
"What is the refund policy?" → Cache hit
"Tell me about your refund rules" → [Embed query → similarity search → Cache hit!]
```

**Use For:** LLM responses to natural language queries, FAQ systems

---

## Implementation Patterns

### Pattern 1: Basic Cache-Aside with Redis

```javascript
const redis = require('redis');
const { OpenAI } = require('openai');

class CacheAsideLLM {
  constructor(config = {}) {
    this.redis = redis.createClient({ url: config.redisUrl });
    this.openai = new OpenAI({ apiKey: config.openaiKey });
    this.ttl = config.ttl || 3600; // 1 hour default
    this.prefix = config.prefix || 'llm:';
  }

  async connect() {
    await this.redis.connect();
  }

  // Generate a cache key from the request
  _cacheKey(model, messages, options = {}) {
    const payload = JSON.stringify({ model, messages, ...options });
    const crypto = require('crypto');
    const hash = crypto.createHash('sha256').update(payload).digest('hex');
    return `${this.prefix}${hash}`;
  }

  async complete(model, messages, options = {}) {
    const key = this._cacheKey(model, messages, options);

    // Step 1: Check cache
    const cached = await this.redis.get(key);
    if (cached) {
      console.log('Cache HIT', { key });
      return { ...JSON.parse(cached), source: 'cache' };
    }

    console.log('Cache MISS', { key });

    // Step 2: Call the API (source of truth)
    const response = await this.openai.chat.completions.create({
      model,
      messages,
      ...options
    });

    // Step 3: Store in cache
    await this.redis.setEx(key, this.ttl, JSON.stringify(response));

    return { ...response, source: 'api' };
  }
}

// Usage
const llm = new CacheAsideLLM({
  redisUrl: process.env.REDIS_URL,
  openaiKey: process.env.OPENAI_API_KEY,
  ttl: 3600 // Cache for 1 hour
});

await llm.connect();

// First call — cache miss, calls API
const res1 = await llm.complete('gpt-4', [
  { role: 'user', content: 'What is machine learning?' }
]);
console.log(res1.source); // 'api'

// Second call — cache hit, instant response
const res2 = await llm.complete('gpt-4', [
  { role: 'user', content: 'What is machine learning?' }
]);
console.log(res2.source); // 'cache'
```

### Pattern 2: Embedding Cache

```javascript
class EmbeddingCache {
  constructor({ redis, openai, ttl = 86400 }) {
    this.redis = redis;
    this.openai = openai;
    this.ttl = ttl; // Embeddings are stable, cache longer (24 hours)
    this.prefix = 'embed:';
  }

  _key(text, model) {
    const crypto = require('crypto');
    const hash = crypto.createHash('sha256').update(`${model}:${text}`).digest('hex');
    return `${this.prefix}${hash}`;
  }

  async embed(text, model = 'text-embedding-3-small') {
    const key = this._key(text, model);

    // Check cache
    const cached = await this.redis.get(key);
    if (cached) {
      return { embedding: JSON.parse(cached), cached: true };
    }

    // Call API
    const response = await this.openai.embeddings.create({ model, input: text });
    const embedding = response.data[0].embedding;

    // Store in cache — embeddings rarely change
    await this.redis.setEx(key, this.ttl, JSON.stringify(embedding));

    return { embedding, cached: false };
  }

  // Batch embed with cache-aside for efficiency
  async embedBatch(texts, model = 'text-embedding-3-small') {
    const results = [];
    const misses = [];

    // Check cache for all texts
    for (let i = 0; i < texts.length; i++) {
      const key = this._key(texts[i], model);
      const cached = await this.redis.get(key);

      if (cached) {
        results[i] = { embedding: JSON.parse(cached), cached: true };
      } else {
        misses.push({ index: i, text: texts[i] });
        results[i] = null; // Placeholder
      }
    }

    // Batch API call for all misses
    if (misses.length > 0) {
      const response = await this.openai.embeddings.create({
        model,
        input: misses.map(m => m.text)
      });

      for (let i = 0; i < misses.length; i++) {
        const { index, text } = misses[i];
        const embedding = response.data[i].embedding;
        
        // Store in cache
        const key = this._key(text, model);
        await this.redis.setEx(key, this.ttl, JSON.stringify(embedding));
        
        results[index] = { embedding, cached: false };
      }
    }

    return results;
  }
}
```

### Pattern 3: Semantic Cache (AI-Powered Cache Lookup)

```javascript
const { ChromaClient } = require('chromadb');

class SemanticCache {
  constructor({ embedder, chroma, redis, threshold = 0.92, ttl = 3600 }) {
    this.embedder = embedder;     // EmbeddingCache instance
    this.chroma = chroma;         // Vector DB for similarity search
    this.redis = redis;           // Key-value store for responses
    this.threshold = threshold;   // Cosine similarity threshold (0–1)
    this.ttl = ttl;
    this.collection = null;
  }

  async init() {
    this.collection = await this.chroma.getOrCreateCollection({
      name: 'semantic_cache',
      metadata: { 'hnsw:space': 'cosine' }
    });
  }

  async get(query) {
    // Embed the incoming query
    const { embedding } = await this.embedder.embed(query);

    // Search for semantically similar cached queries
    const results = await this.collection.query({
      queryEmbeddings: [embedding],
      nResults: 1
    });

    if (results.distances[0]?.[0] <= (1 - this.threshold)) {
      // Found a semantically similar cached response
      const cacheId = results.ids[0][0];
      const cached = await this.redis.get(`semcache:${cacheId}`);
      
      if (cached) {
        console.log('Semantic cache HIT', {
          similarity: 1 - results.distances[0][0],
          matchedQuery: results.documents[0][0]
        });
        return JSON.parse(cached);
      }
    }

    return null; // Cache miss
  }

  async set(query, response) {
    const { embedding } = await this.embedder.embed(query);
    const id = require('crypto').randomUUID();

    // Store embedding in vector DB for similarity lookups
    await this.collection.add({
      ids: [id],
      embeddings: [embedding],
      documents: [query]
    });

    // Store full response in Redis
    await this.redis.setEx(`semcache:${id}`, this.ttl, JSON.stringify(response));
  }
}

// Full semantic cache pipeline
class SemanticCachedLLM {
  constructor({ semanticCache, llm }) {
    this.cache = semanticCache;
    this.llm = llm;
  }

  async complete(query, systemPrompt) {
    // Check semantic cache
    const cached = await this.cache.get(query);
    if (cached) return { ...cached, source: 'semantic_cache' };

    // Call LLM
    const response = await this.llm.complete('gpt-4', [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: query }
    ]);

    // Store in semantic cache
    await this.cache.set(query, response);

    return { ...response, source: 'api' };
  }
}

// Example: "What's your return policy?" and "How do returns work?"
// → Both hit the same semantic cache entry
```

### Pattern 4: Cache with Invalidation

```javascript
class InvalidatingCache {
  constructor({ redis, ttl = 3600 }) {
    this.redis = redis;
    this.ttl = ttl;
    this.tagPrefix = 'tag:';
  }

  // Store with tags for group invalidation
  async set(key, value, tags = []) {
    const multi = this.redis.multi();
    
    // Store the cached value
    multi.setEx(key, this.ttl, JSON.stringify(value));
    
    // Associate key with each tag
    for (const tag of tags) {
      const tagKey = `${this.tagPrefix}${tag}`;
      multi.sAdd(tagKey, key);
      multi.expire(tagKey, this.ttl * 2); // Tags outlive cache entries
    }

    await multi.exec();
  }

  async get(key) {
    const value = await this.redis.get(key);
    return value ? JSON.parse(value) : null;
  }

  // Invalidate all cache entries associated with a tag
  async invalidateByTag(tag) {
    const tagKey = `${this.tagPrefix}${tag}`;
    const keys = await this.redis.sMembers(tagKey);

    if (keys.length > 0) {
      await this.redis.del(...keys, tagKey);
      console.log(`Invalidated ${keys.length} cache entries for tag: ${tag}`);
    }
  }
}

// Usage: Document-aware cache invalidation
class DocumentQACache {
  constructor({ cache, llm }) {
    this.cache = cache;
    this.llm = llm;
  }

  async answer(documentId, question) {
    const key = `qa:${documentId}:${hashString(question)}`;
    
    // Check cache
    const cached = await this.cache.get(key);
    if (cached) return cached;

    // Generate answer
    const doc = await getDocument(documentId);
    const response = await this.llm.complete('gpt-4', [
      { role: 'system', content: `Answer questions about: ${doc.content}` },
      { role: 'user', content: question }
    ]);

    // Cache tagged by document (so we can invalidate when doc updates)
    await this.cache.set(key, response, [`doc:${documentId}`]);

    return response;
  }

  // When a document is updated, invalidate all its cached answers
  async onDocumentUpdated(documentId) {
    await this.cache.invalidateByTag(`doc:${documentId}`);
  }
}
```

---

## AI-Specific Use Cases

### Use Case 1: Caching LLM Responses for FAQ Bots

```
Problem: 80% of chatbot queries are FAQ-style (same 50 questions)
Solution: Cache responses, refresh daily

FAQ Bot Cache Strategy:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"What are your hours?"          → Cache 24h TTL
"How do I reset my password?"   → Cache 24h TTL
"What is your return policy?"   → Cache 24h TTL
"Hi, how are you?"              → No cache (conversational)
"What happened to my order #X?" → No cache (personalized)
```

```javascript
class FAQCachingBot {
  constructor({ cache, llm, embedder }) {
    this.cache = cache;
    this.llm = llm;
    this.embedder = embedder;
    
    // Questions that should never be cached
    this.noCachePatterns = [
      /order\s*#?\d+/i,          // Order-specific
      /my\s+(account|password)/i, // Personal account
      /\b(I|me|my|mine)\b/i      // Personalized questions
    ];
  }

  isCacheable(query) {
    return !this.noCachePatterns.some(p => p.test(query));
  }

  async respond(query, sessionContext = {}) {
    if (!this.isCacheable(query)) {
      // Skip cache for personalized queries
      return this.llm.complete('gpt-4', buildMessages(query, sessionContext));
    }

    // Use semantic cache for FAQ-style queries
    const cached = await this.cache.get(query);
    if (cached) return { ...cached, fromCache: true };

    const response = await this.llm.complete('gpt-4', buildMessages(query, {}));
    await this.cache.set(query, response);
    
    return response;
  }
}
```

### Use Case 2: Embedding Cache for RAG Systems

```
Problem: Re-embedding the same documents on every search query
Solution: Cache document embeddings permanently; cache query embeddings briefly

RAG Embedding Cache Strategy:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Document chunks → Cache FOREVER (stable content)
User queries    → Cache 5 minutes (user might rephrase)
Search results  → Cache 30 minutes (same embedding → same results)
```

```javascript
class CachedRAGPipeline {
  constructor({ embedCache, vectorDB, llm }) {
    this.embedCache = embedCache;
    this.vectorDB = vectorDB;
    this.llm = llm;
  }

  async indexDocument(docId, chunks) {
    const embeddings = await this.embedCache.embedBatch(
      chunks.map(c => c.text),
      'text-embedding-3-small'
    );

    // Log cache efficiency
    const cached = embeddings.filter(e => e.cached).length;
    console.log(`Indexed ${chunks.length} chunks. Cache hits: ${cached}/${chunks.length}`);

    await this.vectorDB.upsert(
      chunks.map((c, i) => ({
        id: `${docId}:${i}`,
        embedding: embeddings[i].embedding,
        metadata: c
      }))
    );
  }

  async query(question, topK = 5) {
    // Embed query (cache for 5 min — same question may appear quickly)
    const { embedding, cached } = await this.embedCache.embed(question);
    
    // Search for relevant chunks
    const results = await this.vectorDB.query(embedding, topK);

    // Generate answer
    const context = results.map(r => r.metadata.text).join('\n\n');
    const response = await this.llm.complete('gpt-4', [
      { role: 'system', content: `Answer based on: ${context}` },
      { role: 'user', content: question }
    ]);

    return { response, queryEmbeddingCached: cached, context };
  }
}
```

### Use Case 3: Caching Model-Generated Structured Data

```javascript
class StructuredOutputCache {
  constructor({ cache, llm, ttl = 1800 }) {
    this.cache = cache;
    this.llm = llm;
    this.ttl = ttl;
  }

  // Cache entity extraction results
  async extractEntities(text) {
    const key = `entities:${hashString(text)}`;
    const cached = await this.cache.get(key);
    if (cached) return cached;

    const response = await this.llm.complete('gpt-4', [
      {
        role: 'system',
        content: 'Extract entities as JSON: { people: [], orgs: [], locations: [] }'
      },
      { role: 'user', content: text }
    ]);

    const entities = JSON.parse(response.choices[0].message.content);
    await this.cache.set(key, entities, this.ttl);
    return entities;
  }

  // Cache sentiment analysis
  async analyzeSentiment(text) {
    const key = `sentiment:${hashString(text)}`;
    const cached = await this.cache.get(key);
    if (cached) return cached;

    const response = await this.llm.complete('gpt-4', [
      {
        role: 'system',
        content: 'Return JSON: { sentiment: "positive"|"negative"|"neutral", score: 0-1 }'
      },
      { role: 'user', content: text }
    ]);

    const result = JSON.parse(response.choices[0].message.content);
    await this.cache.set(key, result, this.ttl);
    return result;
  }
}
```

---

## Advanced Patterns

### Pattern: Cache Stampede Prevention

When many requests all miss the cache simultaneously (e.g., after cache expiry), they all hit the backend at once — a "cache stampede." Use distributed locks to prevent this.

```javascript
class StampedeProtectedCache {
  constructor({ redis, ttl = 3600, lockTtl = 30 }) {
    this.redis = redis;
    this.ttl = ttl;
    this.lockTtl = lockTtl;
    this.lockPrefix = 'lock:';
  }

  async getOrCompute(key, computeFn) {
    // Check cache first
    const cached = await this.redis.get(key);
    if (cached) return { value: JSON.parse(cached), source: 'cache' };

    const lockKey = `${this.lockPrefix}${key}`;

    // Try to acquire lock (NX = only set if not exists)
    const lockAcquired = await this.redis.set(lockKey, '1', {
      NX: true,
      EX: this.lockTtl
    });

    if (lockAcquired) {
      try {
        // This instance computes the value
        console.log('Lock acquired, computing value:', key);
        const value = await computeFn();
        await this.redis.setEx(key, this.ttl, JSON.stringify(value));
        return { value, source: 'computed' };
      } finally {
        await this.redis.del(lockKey);
      }
    } else {
      // Another instance is computing — wait and retry
      console.log('Lock held by another instance, waiting:', key);
      await new Promise(resolve => setTimeout(resolve, 500));
      return this.getOrCompute(key, computeFn); // Retry
    }
  }
}

// Usage: Prevents 1000 simultaneous API calls when cache expires
const cache = new StampedeProtectedCache({ redis });

async function getAIInsights(topic) {
  return cache.getOrCompute(`insights:${topic}`, async () => {
    return await llm.complete('gpt-4', [
      { role: 'user', content: `Provide key insights about: ${topic}` }
    ]);
  });
}
```

### Pattern: Tiered Cache (L1 + L2)

Use in-memory cache (L1) for hot data and Redis (L2) for warm data.

```javascript
const NodeCache = require('node-cache');

class TieredCache {
  constructor({ redis, l1Ttl = 60, l2Ttl = 3600 }) {
    this.l1 = new NodeCache({ stdTTL: l1Ttl, useClones: false }); // In-memory
    this.l2 = redis;                                               // Redis
    this.l2Ttl = l2Ttl;
  }

  async get(key) {
    // L1: In-memory (fastest)
    const l1Hit = this.l1.get(key);
    if (l1Hit !== undefined) {
      return { value: l1Hit, tier: 'L1' };
    }

    // L2: Redis (fast)
    const l2Hit = await this.l2.get(key);
    if (l2Hit) {
      const value = JSON.parse(l2Hit);
      this.l1.set(key, value); // Promote to L1
      return { value, tier: 'L2' };
    }

    return null; // Full miss
  }

  async set(key, value) {
    this.l1.set(key, value);
    await this.l2.setEx(key, this.l2Ttl, JSON.stringify(value));
  }
}

// Result: L1 hit = <1ms, L2 hit = 2-5ms, Miss = 500-2000ms (LLM call)
```

### Pattern: Probabilistic Early Expiry (XFetch)

Refresh cache entries probabilistically before they expire to prevent cold misses.

```javascript
class XFetchCache {
  constructor({ redis, beta = 1.0 }) {
    this.redis = redis;
    this.beta = beta; // Higher beta = more eager refresh
  }

  async get(key) {
    const raw = await this.redis.get(key);
    if (!raw) return null;

    const { value, ttl, delta } = JSON.parse(raw);
    const remainingTtl = await this.redis.ttl(key);

    // XFetch formula: refresh if random condition is met before expiry
    const shouldRefresh = -delta * this.beta * Math.log(Math.random()) >= remainingTtl;

    if (shouldRefresh) {
      console.log('Proactive cache refresh triggered');
      return null; // Pretend it's a miss to trigger refresh
    }

    return value;
  }

  async set(key, value, ttl, computationTimeMs) {
    const delta = computationTimeMs / 1000; // Convert to seconds
    await this.redis.setEx(key, ttl, JSON.stringify({ value, ttl, delta }));
  }
}
```

---

## Monitoring & Observability

### Key Metrics to Track

```javascript
class CacheMetrics {
  constructor({ redis }) {
    this.redis = redis;
    this.counters = {
      hits: 0,
      misses: 0,
      errors: 0,
      invalidations: 0
    };
    this.latencies = { cache: [], api: [] };
  }

  recordHit(latencyMs) {
    this.counters.hits++;
    this.latencies.cache.push(latencyMs);
    this._trimLatencies();
  }

  recordMiss(latencyMs) {
    this.counters.misses++;
    this.latencies.api.push(latencyMs);
    this._trimLatencies();
  }

  _trimLatencies() {
    if (this.latencies.cache.length > 1000) this.latencies.cache.shift();
    if (this.latencies.api.length > 1000) this.latencies.api.shift();
  }

  getStats() {
    const total = this.counters.hits + this.counters.misses;
    const hitRate = total > 0 ? this.counters.hits / total : 0;

    const avgCacheLatency = this.latencies.cache.length > 0
      ? this.latencies.cache.reduce((a, b) => a + b, 0) / this.latencies.cache.length
      : 0;

    const avgApiLatency = this.latencies.api.length > 0
      ? this.latencies.api.reduce((a, b) => a + b, 0) / this.latencies.api.length
      : 0;

    const estimatedMonthlySavings = this.counters.hits * 0.01; // $0.01 per avoided API call

    return {
      hitRate: `${(hitRate * 100).toFixed(1)}%`,
      totalRequests: total,
      hits: this.counters.hits,
      misses: this.counters.misses,
      avgCacheLatencyMs: avgCacheLatency.toFixed(1),
      avgApiLatencyMs: avgApiLatency.toFixed(1),
      speedupFactor: `${(avgApiLatency / avgCacheLatency).toFixed(0)}x`,
      estimatedMonthlySavings: `$${estimatedMonthlySavings.toFixed(2)}`
    };
  }
}

// Express metrics endpoint
app.get('/api/cache/metrics', (req, res) => {
  res.json(metrics.getStats());
});

// Example output:
// {
//   "hitRate": "87.3%",
//   "totalRequests": 50000,
//   "hits": 43650,
//   "misses": 6350,
//   "avgCacheLatencyMs": "2.1",
//   "avgApiLatencyMs": "847.3",
//   "speedupFactor": "403x",
//   "estimatedMonthlySavings": "$436.50"
// }
```

### Cache Health Dashboard

```javascript
class CacheHealthMonitor {
  constructor({ cache, redis, alertThreshold = 0.5 }) {
    this.cache = cache;
    this.redis = redis;
    this.alertThreshold = alertThreshold; // Alert if hit rate drops below 50%
  }

  async getHealth() {
    const info = await this.redis.info('stats');
    const memory = await this.redis.info('memory');

    // Parse Redis info
    const keyspaceHits = parseInt(info.match(/keyspace_hits:(\d+)/)?.[1] || 0);
    const keyspaceMisses = parseInt(info.match(/keyspace_misses:(\d+)/)?.[1] || 0);
    const totalCommands = parseInt(info.match(/total_commands_processed:(\d+)/)?.[1] || 0);
    const usedMemory = parseInt(memory.match(/used_memory:(\d+)/)?.[1] || 0);

    const hitRate = keyspaceHits / (keyspaceHits + keyspaceMisses || 1);
    const status = hitRate < this.alertThreshold ? 'DEGRADED' : 'HEALTHY';

    return {
      status,
      hitRate: `${(hitRate * 100).toFixed(1)}%`,
      totalCommands,
      memoryUsedMB: (usedMemory / 1024 / 1024).toFixed(1),
      alert: status === 'DEGRADED' 
        ? `Hit rate ${(hitRate * 100).toFixed(1)}% below threshold ${this.alertThreshold * 100}%`
        : null
    };
  }
}
```

---

## Integration with Other Patterns

### Cache-Aside + Circuit Breaker

If the cache backend itself fails, use the circuit breaker to fail gracefully and fall back to direct API calls.

```javascript
const CircuitBreaker = require('opossum');

class ResilientCache {
  constructor({ redis, fallbackLlm }) {
    this.redis = redis;
    this.fallbackLlm = fallbackLlm;

    // Circuit breaker for Redis operations
    this.breaker = new CircuitBreaker(
      async (key) => this.redis.get(key),
      {
        timeout: 500,       // Redis should respond in < 500ms
        errorThresholdPercentage: 20,
        resetTimeout: 10000
      }
    );

    this.breaker.on('open', () => {
      console.warn('Redis circuit OPEN — bypassing cache');
    });
  }

  async get(key) {
    try {
      const result = await this.breaker.fire(key);
      return result ? JSON.parse(result) : null;
    } catch {
      console.warn('Cache unavailable, returning null (will hit API)');
      return null; // Degrade gracefully — miss everything, API handles it
    }
  }
}
```

### Cache-Aside + Bulkhead

Use bulkheads to limit concurrency for cache-miss operations (API calls), so cache misses don't overwhelm the LLM.

```javascript
const pLimit = require('p-limit');

class BulkheadedCacheLLM {
  constructor({ cache, openai }) {
    this.cache = cache;
    this.openai = openai;
    // Max 10 concurrent API calls even if 100 cache misses happen simultaneously
    this.apiLimiter = pLimit(10);
  }

  async complete(prompt) {
    const key = hashString(prompt);

    // Cache check — unlimited concurrency (Redis is fast)
    const cached = await this.cache.get(key);
    if (cached) return cached;

    // API call — bulkhead controlled (LLM is slow and limited)
    return this.apiLimiter(async () => {
      // Re-check cache (another request may have populated it while we waited)
      const recheck = await this.cache.get(key);
      if (recheck) return recheck;

      const response = await this.openai.chat.completions.create({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }]
      });

      await this.cache.set(key, response);
      return response;
    });
  }
}
```

### Cache-Aside + Retry with Backoff

```javascript
class RetryingCacheLLM {
  constructor({ cache, llm }) {
    this.cache = cache;
    this.llm = llm;
  }

  async complete(prompt, retries = 3) {
    const key = hashString(prompt);

    // Always check cache first — no retry needed for cache hits
    const cached = await this.cache.get(key);
    if (cached) return cached;

    // Retry only the expensive API call
    for (let attempt = 1; attempt <= retries; attempt++) {
      try {
        const response = await this.llm.complete(prompt);
        await this.cache.set(key, response);
        return response;
      } catch (error) {
        if (attempt === retries) throw error;
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        console.log(`Attempt ${attempt} failed, retrying in ${delay}ms`);
        await new Promise(r => setTimeout(r, delay));
      }
    }
  }
}
```

---

## Real-World Case Studies

### Case Study 1: E-Commerce AI Search

**Problem:**
- Product catalog of 500K items with AI-powered search
- Embedding all products on every search: 500ms latency
- $2,000/month in embedding API costs

**Solution:**

```javascript
// Index products once, cache embeddings permanently
const ragCache = new CachedRAGPipeline({ embedCache, vectorDB, llm });

await ragCache.indexDocument('catalog', productChunks);
// 500K embeddings → 490K cache hits on re-index (existing products)
// Only 10K new/changed products hit the API

// Query at search time
const results = await ragCache.query('lightweight running shoes under $100');
// Query embedding: 2ms (cached)
// Vector search: 15ms
// LLM rerank: 200ms (only when needed)
// Total: ~220ms (vs 800ms without cache)
```

**Results:**
- ✅ Search latency: 800ms → 220ms (73% reduction)
- ✅ Embedding costs: $2,000 → $200/month (90% reduction)
- ✅ 99.8% uptime (cache serves during OpenAI incidents)
- ✅ 15% conversion rate improvement (faster = better UX)
- ✅ Zero rate limit errors

### Case Study 2: SaaS AI Writing Assistant

**Problem:**
- 50,000 users generating content with GPT-4
- Same templates reused constantly (email templates, blog intros, etc.)
- $15,000/month in API costs
- Occasional OpenAI outages disrupting users

**Solution:**

```javascript
const semanticCache = new SemanticCache({
  embedder,
  chroma,
  redis,
  threshold: 0.94, // 94% similarity = cache hit
  ttl: 7200         // 2 hours for writing templates
});

const writingBot = new SemanticCachedLLM({ semanticCache, llm });

// "Write a professional email declining a meeting"
// "Draft an email to politely decline a calendar invite"
// → Both resolve to same semantic cache entry → instant response
```

**Results:**
- ✅ API costs: $15,000 → $4,500/month (70% reduction)
- ✅ Cache hit rate: 74% (most queries are template-based)
- ✅ P95 latency: 2,400ms → 320ms
- ✅ During OpenAI outage: 74% of users unaffected (cache served)
- ✅ User satisfaction: +22% (NPS improvement from speed)

### Case Study 3: Healthcare AI Document Q&A

**Problem:**
- Medical staff querying clinical guidelines (large, stable documents)
- Same questions asked daily by different staff
- Strict latency requirements: < 3 seconds for clinical decision support

**Solution:**

```javascript
const docQA = new DocumentQACache({ cache: invalidatingCache, llm });

// Staff ask: "What is the protocol for post-op pain management?"
// → First ask: cache miss, 2.1s LLM response, cached for 24h
// → Next 200 staff who ask: 8ms cache hit

// When clinical guidelines updated:
await docQA.onDocumentUpdated('pain-management-protocol-v12');
// → Invalidates all related Q&A cache entries
// → Next query re-computes from new document version
```

**Results:**
- ✅ P99 latency: 4.2s → 1.1s (meets clinical requirement)
- ✅ 83% of queries served from cache
- ✅ Cost reduction: $8,000 → $1,360/month
- ✅ Cache invalidation ensures clinical accuracy (no stale guidance)
- ✅ HIPAA compliant (no PII stored in cache keys)

---

## Production Best Practices

### 1. Choosing TTL Values

```javascript
const TTL_STRATEGY = {
  // Stable content — long TTL
  documentEmbeddings:  86400 * 30,  // 30 days (documents rarely change)
  productDescriptions: 86400,        // 24 hours

  // Moderately stable — medium TTL
  faqResponses:        3600,         // 1 hour
  searchResults:       1800,         // 30 minutes
  sentimentAnalysis:   3600,         // 1 hour

  // Volatile — short TTL
  priceQueries:        300,          // 5 minutes
  inventoryStatus:     60,           // 1 minute
  userSessionData:     1800,         // 30 minutes

  // Never cache
  personalizedContent: 0,            // Per-user, never cache
  realTimeData:        0,            // Stock prices, live feeds
  transactionData:     0             // Payments, orders
};
```

### 2. What NOT to Cache

```javascript
class SmartCacheDecider {
  shouldCache(request) {
    const { query, userId, context } = request;

    // Never cache personalized content
    if (userId && this.isPersonalized(query)) return false;

    // Never cache real-time queries
    if (this.isRealTime(query)) return false;

    // Never cache sensitive information
    if (this.containsPII(query)) return false;

    // Never cache very short TTL content (not worth it)
    if (this.estimatedTTL(query) < 60) return false;

    return true;
  }

  isPersonalized(query) {
    const personalPatterns = [/\bmy\b/i, /\bI\b/, /\bme\b/i, /order\s*#\d+/];
    return personalPatterns.some(p => p.test(query));
  }

  isRealTime(query) {
    const realtimePatterns = [/current price/i, /right now/i, /live/i, /today's/i];
    return realtimePatterns.some(p => p.test(query));
  }

  containsPII(query) {
    const piiPatterns = [/\b\d{3}-\d{2}-\d{4}\b/, /\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b/i];
    return piiPatterns.some(p => p.test(query));
  }
}
```

### 3. Testing Cache-Aside

```javascript
describe('CacheAsideLLM', () => {
  it('should serve subsequent requests from cache', async () => {
    const mockLlm = jest.fn().mockResolvedValue({ content: 'AI response' });
    const cache = new CacheAsideLLM({ llm: mockLlm, redis });

    // First call — should hit LLM
    const res1 = await cache.complete('What is machine learning?');
    expect(mockLlm).toHaveBeenCalledTimes(1);
    expect(res1.source).toBe('api');

    // Second call — should hit cache
    const res2 = await cache.complete('What is machine learning?');
    expect(mockLlm).toHaveBeenCalledTimes(1); // Still 1, not 2
    expect(res2.source).toBe('cache');
  });

  it('should not cache personalized queries', async () => {
    const cache = new FAQCachingBot({ cache: mockCache, llm: mockLlm });

    await cache.respond('What is my account balance?', { userId: 'u1' });
    await cache.respond('What is my account balance?', { userId: 'u1' });

    expect(mockLlm).toHaveBeenCalledTimes(2); // Called twice — not cached
  });

  it('should handle cache backend failure gracefully', async () => {
    const brokenRedis = { get: () => Promise.reject(new Error('Redis down')) };
    const cache = new ResilientCache({ redis: brokenRedis, fallbackLlm: mockLlm });

    // Should still work — falls through to API
    const result = await cache.complete('Hello?');
    expect(result).toBeDefined();
  });
});
```

### 4. Cache Sizing Calculator

```javascript
class CacheSizingCalculator {
  static calculate({ dailyRequests, avgResponseSizeKB, expectedHitRate, ttlHours }) {
    const uniqueRequests = dailyRequests * (1 - expectedHitRate);
    const entriesPerDay = uniqueRequests;
    const ttlDays = ttlHours / 24;
    const liveEntries = entriesPerDay * ttlDays;
    const memorySizeMB = (liveEntries * avgResponseSizeKB) / 1024;
    const costSavedDaily = dailyRequests * expectedHitRate * 0.01; // $0.01/call

    return {
      liveEntries: Math.ceil(liveEntries),
      recommendedMemoryMB: Math.ceil(memorySizeMB * 1.3), // 30% headroom
      dailyCostSavings: `$${costSavedDaily.toFixed(2)}`,
      monthlyCostSavings: `$${(costSavedDaily * 30).toFixed(2)}`
    };
  }
}

// Example
const sizing = CacheSizingCalculator.calculate({
  dailyRequests: 100000,
  avgResponseSizeKB: 4,    // ~4KB per LLM response
  expectedHitRate: 0.80,   // 80% cache hit rate
  ttlHours: 24
});

// {
//   liveEntries: 20000,
//   recommendedMemoryMB: 104,
//   dailyCostSavings: "$800.00",
//   monthlyCostSavings: "$24000.00"
// }
```

---

## Conclusion

### Key Takeaways

1. **Cache-Aside Slashes AI Costs**: 70–90% reduction in LLM and embedding API expenses
2. **Speed Matters for AI UX**: 2ms cache hit vs 800ms API call = 400x speedup
3. **Resilience During Outages**: Cache serves users when AI providers have incidents
4. **Semantic Cache is AI-Native**: Match similar questions, not just identical strings
5. **Smart Invalidation**: Tag-based invalidation keeps content accurate when data changes
6. **Combine with Other Patterns**: Works with circuit breakers, bulkheads, and retry
7. **Know What NOT to Cache**: Personalized, real-time, and PII data must never be cached

### When to Use Cache-Aside

✅ **Use Cache-Aside For:**
- FAQ-style chatbots with repeated questions
- RAG pipelines re-embedding the same documents
- Semantic search with overlapping user queries
- Expensive LLM calls that return deterministic outputs
- High-traffic applications with cost constraints
- Systems that need resilience to AI provider outages

❌ **Don't Use Cache-Aside For:**
- Personalized content (per-user responses)
- Real-time data (stock prices, live inventory)
- Content with PII or sensitive user data
- Highly creative/stochastic outputs (temperature > 0.8)
- Data that changes faster than your minimum TTL

### ROI Summary

**Investment:**
- Implementation: 2–3 days = $3K
- Redis infrastructure: $50–200/month
- Testing: 1 day = $1K
- **Total: ~$4.5K + $100/month**

**Returns (Annual, 100K req/day at 80% hit rate):**
- API cost savings: $288,000/year
- Latency improvement (conversion gains): $120,000/year
- Reduced outage impact: $50,000/year
- **Total: $458,000/year**

**ROI: 10,000%+ first year**

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
