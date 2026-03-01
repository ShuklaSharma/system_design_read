# Redis Distributed Cache for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why Distributed Redis Matters for AI Systems](#why-distributed-redis-matters)
3. [Distribution Strategies](#distribution-strategies)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Monitoring & Observability](#monitoring-observability)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Real-World Case Studies](#real-world-case-studies)
10. [Production Best Practices](#production-best-practices)

---

## Introduction

### What is a Distributed Redis Cache?

**Distributed Redis** is the practice of running multiple Redis instances that coordinate together to serve a single logical cache — spreading data, load, and failure risk across many nodes instead of one. Rather than one Redis handling everything, data is partitioned across a cluster of nodes, each owning a portion of the keyspace, with replicas providing redundancy.

**Analogy:** Like a bank with multiple branches — instead of one branch serving all customers (single point of failure, long queues), each branch serves its local customers. The branches are connected, share policies, and if one branch closes temporarily, nearby branches cover for it. Customers are automatically directed to the right branch for their account, every time.

### The Problem With a Single Redis Instance

```
Scenario: AI chatbot serving 500,000 users/day
          Cache: single Redis, 16GB RAM

Without Distributed Redis:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Month 1:  50K users   → 4GB used  → 2ms latency  ✅
Month 3: 150K users   → 11GB used → 3ms latency  ⚠️
Month 5: 300K users   → 16GB used → RAM full      ❌
Month 5: Redis starts evicting hot keys
          Cache hit rate drops: 91% → 54%
          Every miss → OpenAI API call → $0.03
          Daily API cost: $270 → $830 (3x spike)
Month 6: Traffic spike — 50,000 concurrent users
          Single Redis: CPU 100%, 890ms latency
          App servers timing out waiting for cache
          All 10 app servers hammering one Redis
Month 6: Redis primary crashes
          All 10 app servers: 100% cache miss
          OpenAI rate limit hit within 60 seconds
          Platform down: 23 minutes

Impact:
- $18,000/month unexpected API overage
- 23-minute full outage
- No failover — single node = single point of failure
- Can't scale without restarting everything
```

### The Solution With Distributed Redis

```
With Redis Cluster (6 nodes: 3 primaries + 3 replicas):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Each primary: 16GB RAM → Total: 48GB cache capacity
16,384 hash slots split evenly: ~5,461 slots per primary

Month 5: 300K users  → 16GB used across cluster → 1.8ms latency ✅
Month 6: Traffic spike — 50,000 concurrent users
          Reads distributed across 3 primaries + 3 replicas
          Each node handles ~8,333 concurrent users
          Latency: 2.1ms (unchanged)
Month 6: Primary 1 crashes
          Replica 1 promoted automatically in ~8 seconds
          8 seconds of slightly higher miss rate
          Zero full outage

Impact:
- API costs stable (high hit rate maintained)
- Zero full outage on node failure
- Scale: add node pairs without downtime
- Cache capacity triples with 3x the nodes
```

---

## Why Distributed Redis Matters for AI Systems

### Unique AI System Challenges

1. **Cache is a Cost Control Mechanism, Not Just Performance**
   - Every cache miss on an LLM query = $0.01–$0.05 in API costs
   - A single Redis crash at peak → thousands of simultaneous misses → rate limits hit
   - At 500K queries/day with 85% hit rate: miss = $7,500/day; single-node failure for 23 min = $2,400 in wasted API spend
   - Distributed Redis makes cache a financially critical, fault-tolerant system

2. **AI Workloads Have Extreme Memory Requirements**
   - Each embedding: 1,536 floats × 4 bytes = ~6KB
   - 10M cached embeddings = 60GB — impossible on one node
   - LLM response cache: avg 4KB per entry × 5M entries = 20GB
   - Semantic cache indices add another 10–20GB
   - Distribution is the only way to hold this in memory

3. **Read Traffic is Massively Asymmetric**
   - 10 app servers all read from the same cache simultaneously
   - Single Redis: CPU bound by 10x concurrent connections
   - Distributed Redis: reads can go to any replica — linear read scaling
   - AI inference is read-dominant: 95% reads, 5% writes

4. **Cache Stampede Risk is Amplified**
   - At scale, cache expiry hits hundreds of keys simultaneously
   - All 10 app servers miss → all hit OpenAI API → rate limit 429
   - Distributed Redis with proper locking prevents the stampede at each node level
   - Without distribution, one lock per cluster vs. many isolated stampedes

### Real Incident: Single Redis Instance

**AI Writing Assistant SaaS — 80K users**

```
Timeline:
Tuesday 14:00 — Platform running normally
               Redis: 14.2GB used / 16GB capacity
               Cache hit rate: 88%

14:30 — Marketing sends email blast to 200K users
14:32 — Traffic: 200 req/sec → 1,400 req/sec (7x spike)
14:33 — Redis memory: 16GB → OOM → starts evicting LRU keys
14:34 — Cache hit rate: 88% → 41% (evictions killing hot keys)
14:35 — OpenAI API requests spike: 180/min → 1,100/min
14:36 — OpenAI rate limit hit: 429 errors flooding in
14:37 — Redis CPU: 100% (10 app servers hammering one instance)
14:38 — Redis starts dropping connections: "max clients reached"
14:39 — App servers: connection pool exhausted → 503 errors to users
14:45 — Engineers notice, start emergency scaling
15:00 — New Redis provisioned but cold (empty cache)
15:00 → 15:45 — 45 min of 100% miss rate → $2,700 in API costs
16:00 — Cache warms up, platform stabilizes

Impact:
- 82 minutes of degraded/down service
- $4,100 in emergency API costs (vs $180 normal for that window)
- 3,400 users abandoned during outage (est. $68K LTV loss)
- 7,200 support tickets
- SLA breach for 14 enterprise customers
- Total incident cost: ~$180,000
```

**Same Incident WITH Distributed Redis Cluster:**

```
Tuesday 14:00 — Platform running normally
               Redis Cluster: 3 primaries × 16GB = 48GB total
               Cache hit rate: 88%

14:30 — Marketing sends email blast
14:32 — Traffic: 200 → 1,400 req/sec
14:33 — Cluster memory: 14.2GB / 48GB (29% used — plenty of headroom)
         Zero evictions needed
14:34 — Cache hit rate: 88% → 87% (minor dip, no eviction cascade)
14:35 — Reads distributed across 3 primaries + 3 replicas
         Each node: ~233 req/sec (well within limits)
14:36 — OpenAI requests: stable (cache absorbing spike)
14:40 — Traffic begins normalizing
15:00 — Platform stable throughout

Impact:
- Zero downtime
- Zero rate limit errors
- API costs: normal
- Engineers unaware anything unusual happened until they checked metrics
```

---

## Distribution Strategies

### Strategy 1: Single Primary + Read Replicas

The simplest step toward distribution. One primary handles all writes; replicas handle reads.

```
All Writes ──────────────────────► Primary (10.0.0.1:6379)
                                        │
                              async replication
                                   ↙        ↘
App servers read ──► Replica 1          Replica 2
                     (10.0.0.2)         (10.0.0.3)
```

**How it works:** Primary replicates to replicas asynchronously. Your app directs writes to primary, reads to any replica.

**Pros:** Simple to set up, reduces read load on primary, extra copies of data
**Cons:** Primary is still a single point of failure for writes; data is NOT sharded (all nodes hold all data — capacity doesn't scale)
**Use For:** Read-heavy workloads where you need read scale but data fits on one node

```javascript
const redis = require('ioredis');

// Separate clients for write path and read path
const primary = new redis({ host: 'redis-primary', port: 6379 });
const replicas = [
  new redis({ host: 'redis-replica-1', port: 6379 }),
  new redis({ host: 'redis-replica-2', port: 6379 })
];

// Round-robin across replicas for reads
let replicaIndex = 0;
function getReadClient() {
  const client = replicas[replicaIndex % replicas.length];
  replicaIndex++;
  return client;
}

async function cacheGet(key) {
  return getReadClient().get(key);      // Reads distributed across replicas
}

async function cacheSet(key, value, ttl) {
  return primary.setex(key, ttl, value); // Writes always go to primary
}
```

### Strategy 2: Redis Sentinel (Automatic Failover)

Sentinel adds automatic failover on top of primary/replica. Three Sentinel processes monitor your Redis nodes and vote to promote a replica if the primary dies.

```
                    ┌─► Sentinel 1 ──┐
App Servers ──► Ask │─► Sentinel 2 ──┼─► "Primary is at 10.0.0.1"
  "where is        └─► Sentinel 3 ──┘    (majority vote required)
   primary?"
   
Primary (10.0.0.1) ──replicates──► Replica 1 (10.0.0.2)
                    ──replicates──► Replica 2 (10.0.0.3)

Primary crashes:
  Sentinel 1: "Primary unreachable — vote to promote Replica 1"
  Sentinel 2: "Agreed"
  Sentinel 3: "Agreed" (majority = 2/3)
  → Replica 1 promoted to primary (~30 seconds)
  → App clients automatically reconnect to new primary
  → Replica 2 now replicates from new primary
```

**Pros:** Automatic failover, no manual intervention, clients reconnect automatically
**Cons:** Data still NOT sharded — capacity is still one primary's worth of RAM; 30-second failover window
**Use For:** High availability without sharding needs; medium-traffic AI platforms

```javascript
const redis = require('ioredis');

const client = new redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26379 },
    { host: 'sentinel-3', port: 26379 }
  ],
  name: 'mymaster',              // Name of the primary group
  sentinelRetryStrategy: (times) => Math.min(times * 200, 2000),
  role: 'master'                 // 'master' for writes, 'slave' for reads
});

// For read replicas via Sentinel:
const readClient = new redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26379 },
    { host: 'sentinel-3', port: 26379 }
  ],
  name: 'mymaster',
  role: 'slave'                  // ioredis routes to a replica automatically
});

// Failover is transparent — client reconnects after ~30s
await client.set('key', 'value');        // → always goes to current primary
await readClient.get('key');             // → goes to a replica
```

### Strategy 3: Redis Cluster (Sharding + HA — Production Standard)

Redis Cluster is the full solution: automatic sharding across nodes AND automatic failover. This is what you use when data volume or write throughput exceeds a single node.

#### How Hash Slots Work

Redis Cluster divides the entire keyspace into **16,384 slots** (numbered 0–16,383). Every key maps to exactly one slot via a deterministic formula:

```
slot = CRC16(key) % 16384

"user:123"         → CRC16 → slot 8,121  → Node A
"session:xyz"      → CRC16 → slot 2,891  → Node B
"model:gpt4:cache" → CRC16 → slot 14,302 → Node C
```

Each node owns a contiguous range of slots:

```
Node A (Primary): slots     0 –  5,460   (⅓ of keyspace)
Node B (Primary): slots 5,461 – 10,922   (⅓ of keyspace)
Node C (Primary): slots 10,923 – 16,383  (⅓ of keyspace)

Each primary has a replica for HA:
Node A-Replica ← replicates from Node A
Node B-Replica ← replicates from Node B
Node C-Replica ← replicates from Node C
```

#### What Happens When Your App Reads or Writes

```
App: SET "user:123" "{...}"
  Step 1: Client computes CRC16("user:123") % 16384 = slot 8,121
  Step 2: Client checks its slot map: slot 8,121 → Node A
  Step 3: Client sends SET directly to Node A
  Step 4: Node A stores the key and returns OK

App: GET "user:123"
  Step 1: Client computes slot 8,121
  Step 2: Client routes GET to Node A (or Node A-Replica for reads)
  Step 3: Node A returns the value

Wrong node (MOVED redirect):
  App sends GET "user:123" to Node B (stale slot map)
  Node B: "I don't own slot 8,121 — MOVED 8121 10.0.0.1:6379"
  Client: updates its slot map, retries against Node A
  (This is handled automatically by cluster-aware clients)
```

#### Node Failure in Redis Cluster

```
Normal state:
  Node A Primary (slots 0–5,460) ──► Node A Replica

Node A Primary crashes:
  T+0s:   Cluster detects heartbeat failure
  T+5s:   Cluster confirms Node A unreachable (gossip consensus)
  T+8s:   Node A Replica promoted to primary automatically
  T+8s:   Cluster gossip spreads new slot ownership to all nodes
  T+10s:  Clients receive CLUSTERDOWN error briefly, then reconnect
  T+15s:  Slots 0–5,460 now served by former Node A Replica
  T+?:    Add a new replica for the promoted node to restore HA

Data loss: Zero (if replication was in sync)
Failover window: ~8–15 seconds of elevated errors
```

### Strategy 4: Client-Side Consistent Hashing

Without Redis Cluster mode, your application layer does the sharding using consistent hashing — the same approach described in the Sharding Pattern guide. This gives you control over routing without the cluster protocol overhead.

```
App Client:
  hash("user:123") → 8,121
  Ring lookup: 8,121 → Node B
  Request goes directly to Node B

Adding Node D:
  Old ring: A(0–5k), B(5k–10k), C(10k–16k)
  New ring: A(0–4k), B(4k–8k), C(8k–12k), D(12k–16k)
  Only ~25% of keys remapped (not all of them)
```

**Use For:** When you control the client layer completely, want to avoid cluster protocol complexity, or use Redis providers that don't support cluster mode.

---

## Implementation Patterns

### Pattern 1: Redis Cluster Setup and Connection

```javascript
const { Cluster } = require('ioredis');

class RedisClusterClient {
  constructor(nodes, options = {}) {
    this.cluster = new Cluster(nodes, {
      // ioredis cluster options
      clusterRetryStrategy: (times) => {
        if (times > 10) return null; // Stop retrying after 10 attempts
        return Math.min(100 * times, 3000); // Backoff
      },
      redisOptions: {
        password: options.password,
        tls: options.tls,
        connectTimeout: 5000,
        maxRetriesPerRequest: 3
      },
      // Read from replicas to spread read load
      scaleReads: 'slave',         // 'master' | 'slave' | 'all'
      maxRedirections: 16,         // Max MOVED/ASK redirections to follow
      enableOfflineQueue: true     // Queue commands during reconnect
    });

    this.cluster.on('connect', () => console.log('Redis Cluster connected'));
    this.cluster.on('error', (err) => console.error('Redis Cluster error:', err));
    this.cluster.on('+node', (node) => console.log('Node added:', node.options.host));
    this.cluster.on('-node', (node) => console.warn('Node removed:', node.options.host));
  }

  async get(key) {
    return this.cluster.get(key);
  }

  async set(key, value, ttlSeconds) {
    if (ttlSeconds) {
      return this.cluster.setex(key, ttlSeconds, value);
    }
    return this.cluster.set(key, value);
  }

  async del(key) {
    return this.cluster.del(key);
  }

  async getClusterInfo() {
    // ioredis provides access to individual nodes
    const nodes = this.cluster.nodes('master');
    const info = await Promise.all(
      nodes.map(async node => ({
        host: node.options.host,
        port: node.options.port,
        info: await node.info('replication')
      }))
    );
    return info;
  }
}

// Usage
const cache = new RedisClusterClient([
  { host: 'redis-node-a', port: 6379 },  // You only need to list SOME nodes
  { host: 'redis-node-b', port: 6379 },  // Cluster discovers the rest automatically
  { host: 'redis-node-c', port: 6379 }
], {
  password: process.env.REDIS_PASSWORD,
  tls: process.env.NODE_ENV === 'production'
});

// Transparent routing — client handles everything
await cache.set('user:123', JSON.stringify(user), 3600);
const user = await cache.get('user:123');
// "user:123" always lands on the same node — no matter which app server calls it
```

### Pattern 2: Hash Tags — Forcing Related Keys to the Same Node

The biggest gotcha in Redis Cluster: you cannot run multi-key operations (pipelines, MULTI/EXEC transactions, Lua scripts) across keys on different nodes. Hash tags solve this by forcing keys to share the same slot.

```javascript
// ─── WITHOUT hash tags — keys scatter across nodes ───────────────────────

await cache.set('user:123:profile', profileData);   // → slot 8121 → Node A
await cache.set('user:123:settings', settingsData); // → slot 4902 → Node B
await cache.set('user:123:tokens', tokenCount);     // → slot 3841 → Node B

// PROBLEM: This pipeline spans Node A and Node B → CROSSSLOT error!
const pipeline = cache.cluster.pipeline();
pipeline.get('user:123:profile');
pipeline.get('user:123:settings');
pipeline.get('user:123:tokens');
await pipeline.exec(); // ❌ Error: CROSSSLOT Keys in request don't hash to the same slot

// ─── WITH hash tags — keys are forced to the same slot ───────────────────

// The {} part is what Redis hashes. Everything outside {} is ignored for routing.
// All three keys hash on "user:123" → same slot → same node

await cache.set('{user:123}:profile', profileData);   // → slot 8121 → Node A
await cache.set('{user:123}:settings', settingsData); // → slot 8121 → Node A
await cache.set('{user:123}:tokens', tokenCount);     // → slot 8121 → Node A

// ✅ NOW this pipeline works — all keys on same node
const pipeline = cache.cluster.pipeline();
pipeline.get('{user:123}:profile');
pipeline.get('{user:123}:settings');
pipeline.get('{user:123}:tokens');
const [profile, settings, tokens] = await pipeline.exec();

// ─── AI caching example with hash tags ───────────────────────────────────

class AIUserCache {
  constructor(cluster) {
    this.cluster = cluster;
  }

  // All user-scoped AI data co-located on same node
  userKey(userId, type) {
    return `{user:${userId}}:${type}`;
  }

  async storeUserContext(userId, { embedding, history, preferences }) {
    const pipeline = this.cluster.pipeline();
    pipeline.setex(this.userKey(userId, 'embedding'), 86400, JSON.stringify(embedding));
    pipeline.setex(this.userKey(userId, 'history'), 3600, JSON.stringify(history));
    pipeline.setex(this.userKey(userId, 'preferences'), 86400 * 7, JSON.stringify(preferences));
    return pipeline.exec(); // ✅ All on same node — one round trip
  }

  async loadUserContext(userId) {
    const pipeline = this.cluster.pipeline();
    pipeline.get(this.userKey(userId, 'embedding'));
    pipeline.get(this.userKey(userId, 'history'));
    pipeline.get(this.userKey(userId, 'preferences'));
    const [[, embedding], [, history], [, preferences]] = await pipeline.exec();
    return {
      embedding: JSON.parse(embedding),
      history: JSON.parse(history),
      preferences: JSON.parse(preferences)
    };
  }
}
```

### Pattern 3: Consistent Hashing (Client-Side Sharding)

For scenarios where Redis Cluster mode is unavailable (e.g., managed Redis tiers that don't support cluster), implement sharding in your application layer.

```javascript
const crypto = require('crypto');
const redis = require('ioredis');

class ConsistentHashRedisPool {
  constructor(nodeConfigs, { virtualNodes = 150 } = {}) {
    // Connect to each Redis instance independently
    this.nodes = nodeConfigs.map(cfg => ({
      client: new redis(cfg),
      config: cfg
    }));

    // Build the consistent hash ring
    this.ring = [];
    for (let nodeIdx = 0; nodeIdx < this.nodes.length; nodeIdx++) {
      for (let v = 0; v < virtualNodes; v++) {
        const hash = this._hash(`${nodeConfigs[nodeIdx].host}:${nodeConfigs[nodeIdx].port}:${v}`);
        this.ring.push({ hash, nodeIdx });
      }
    }
    this.ring.sort((a, b) => a.hash - b.hash);
  }

  _hash(str) {
    const hex = crypto.createHash('md5').update(str).digest('hex');
    return parseInt(hex.substring(0, 8), 16);
  }

  // Find which node owns a given key (clockwise lookup on the ring)
  getNodeIndex(key) {
    const keyHash = this._hash(key);
    for (const point of this.ring) {
      if (keyHash <= point.hash) return point.nodeIdx;
    }
    return this.ring[0].nodeIdx; // Wrap around to first node
  }

  getNode(key) {
    return this.nodes[this.getNodeIndex(key)].client;
  }

  async get(key) {
    return this.getNode(key).get(key);
  }

  async set(key, value, ttl) {
    const node = this.getNode(key);
    return ttl ? node.setex(key, ttl, value) : node.set(key, value);
  }

  async del(key) {
    return this.getNode(key).del(key);
  }

  // When a node is added: only ~1/n keys need to move (not all)
  addNode(config) {
    const newNodeIdx = this.nodes.length;
    this.nodes.push({ client: new redis(config), config });

    const virtualNodes = 150;
    for (let v = 0; v < virtualNodes; v++) {
      const hash = this._hash(`${config.host}:${config.port}:${v}`);
      this.ring.push({ hash, nodeIdx: newNodeIdx });
    }
    this.ring.sort((a, b) => a.hash - b.hash);

    console.log(`Node ${config.host} added. ~${Math.round(100 / this.nodes.length)}% of keys remapped.`);
  }

  getDistribution() {
    const counts = new Array(this.nodes.length).fill(0);
    this.ring.forEach(p => counts[p.nodeIdx]++);
    return counts.map((count, i) => ({
      node: this.nodes[i].config.host,
      virtualNodeCount: count,
      estimatedKeyPercentage: `${((count / this.ring.length) * 100).toFixed(1)}%`
    }));
  }
}

// Usage
const pool = new ConsistentHashRedisPool([
  { host: 'redis-0', port: 6379 },
  { host: 'redis-1', port: 6379 },
  { host: 'redis-2', port: 6379 }
]);

await pool.set('user:123', JSON.stringify(user), 3600);
// Same key always routes to same node
const result = await pool.get('user:123');

// Add a node — only ~25% of keys remapped, not all
pool.addNode({ host: 'redis-3', port: 6379 });
```

### Pattern 4: Cache Invalidation Across Instances

When data changes, all app servers and cache nodes must know to discard the stale value. Redis Pub/Sub makes this possible without polling.

```javascript
class DistributedCacheInvalidator {
  constructor(cluster) {
    // Dedicated subscriber connection (can't mix pub/sub with regular commands)
    this.publisher = cluster;
    this.subscriber = cluster.duplicate();
    this.localCache = new Map(); // Optional L1 in-memory cache per app server

    this._listenForInvalidations();
  }

  _listenForInvalidations() {
    // Subscribe to invalidation events published by any node in the cluster
    this.subscriber.subscribe('cache:invalidate', (err) => {
      if (err) console.error('Subscription error:', err);
    });

    this.subscriber.on('message', (channel, key) => {
      if (channel === 'cache:invalidate') {
        // Clear from local in-memory cache (L1)
        this.localCache.delete(key);
        console.log(`[Cache] Invalidated local cache for key: ${key}`);
      }
    });
  }

  // Store with optional local cache
  async set(key, value, ttlSeconds) {
    const serialized = JSON.stringify(value);
    await this.publisher.setex(key, ttlSeconds, serialized);
    // Also store in local L1 cache for ultra-fast subsequent reads
    this.localCache.set(key, { value, expiresAt: Date.now() + ttlSeconds * 1000 });
  }

  // Read: L1 → L2 (Redis Cluster) → miss
  async get(key) {
    // Check local memory first (sub-millisecond)
    const local = this.localCache.get(key);
    if (local && local.expiresAt > Date.now()) {
      return local.value;
    }

    // Check Redis cluster
    const serialized = await this.publisher.get(key);
    if (!serialized) return null;

    const value = JSON.parse(serialized);
    // Populate local cache
    this.localCache.set(key, { value, expiresAt: Date.now() + 60_000 });
    return value;
  }

  // Invalidate across ALL app server instances via pub/sub
  async invalidate(key) {
    // Delete from Redis cluster (all nodes)
    await this.publisher.del(key);

    // Publish invalidation event — all subscribed app servers clear their L1 cache
    await this.publisher.publish('cache:invalidate', key);
    console.log(`[Cache] Published invalidation for: ${key}`);
  }

  // Invalidate a pattern (e.g., all keys for a tenant)
  async invalidatePattern(pattern) {
    // In cluster mode: must scan each node individually
    const masterNodes = this.publisher.nodes('master');

    for (const node of masterNodes) {
      let cursor = '0';
      do {
        const [nextCursor, keys] = await node.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
        cursor = nextCursor;

        for (const key of keys) {
          await node.del(key);
          await this.publisher.publish('cache:invalidate', key);
        }
      } while (cursor !== '0');
    }
  }
}

// Usage
const cache = new DistributedCacheInvalidator(redisCluster);

// Store AI response
await cache.set('llm:user:123:query:abc', { text: 'AI response...' }, 3600);

// When user updates their profile — invalidate all their cached responses
await cache.invalidate('llm:user:123:query:abc');

// When a document is updated — invalidate all RAG responses for that doc
await cache.invalidatePattern('{doc:456}:*');
```

---

## AI-Specific Use Cases

### Use Case 1: Distributed Semantic Cache for LLM Queries

```javascript
class DistributedSemanticCache {
  constructor({ cluster, embedder, threshold = 0.93 }) {
    this.cluster = cluster;
    this.embedder = embedder;
    this.threshold = threshold;
    // Semantic index stored in Redis itself using sorted sets + vector data
  }

  // Store an LLM response with its embedding for future similarity lookups
  async set(query, response, ttlSeconds = 3600) {
    const { embedding } = await this.embedder.embed(query);
    const queryHash = hashString(query);

    // Use hash tags to co-locate the embedding + response on the same node
    const embeddingKey = `{semcache:${queryHash}}:embedding`;
    const responseKey  = `{semcache:${queryHash}}:response`;
    const metaKey      = `{semcache:${queryHash}}:meta`;

    const pipeline = this.cluster.pipeline();
    pipeline.setex(embeddingKey, ttlSeconds, JSON.stringify(embedding));
    pipeline.setex(responseKey,  ttlSeconds, JSON.stringify(response));
    pipeline.setex(metaKey,      ttlSeconds, JSON.stringify({ query, cachedAt: new Date() }));
    await pipeline.exec();

    // Register this hash in the semantic index (for lookup)
    await this.cluster.zadd('semcache:index', Date.now(), queryHash);
    await this.cluster.expire('semcache:index', ttlSeconds);

    console.log(`[SemanticCache] Stored response for: "${query.substring(0, 50)}..."`);
  }

  // Find a semantically similar cached response
  async get(query) {
    const { embedding: queryEmbedding } = await this.embedder.embed(query);

    // Get all cached query hashes
    const hashes = await this.cluster.zrange('semcache:index', 0, -1);

    let bestMatch = null;
    let bestSimilarity = 0;

    // Compare against cached embeddings in parallel (batched for efficiency)
    const BATCH_SIZE = 50;
    for (let i = 0; i < hashes.length; i += BATCH_SIZE) {
      const batch = hashes.slice(i, i + BATCH_SIZE);

      const embeddings = await Promise.all(
        batch.map(hash => this.cluster.get(`{semcache:${hash}}:embedding`))
      );

      for (let j = 0; j < batch.length; j++) {
        if (!embeddings[j]) continue;
        const cachedEmbedding = JSON.parse(embeddings[j]);
        const similarity = cosineSimilarity(queryEmbedding, cachedEmbedding);

        if (similarity > bestSimilarity) {
          bestSimilarity = similarity;
          bestMatch = batch[j];
        }
      }
    }

    if (bestSimilarity >= this.threshold && bestMatch) {
      const response = await this.cluster.get(`{semcache:${bestMatch}}:response`);
      console.log(`[SemanticCache] HIT — similarity: ${(bestSimilarity * 100).toFixed(1)}%`);
      return JSON.parse(response);
    }

    console.log(`[SemanticCache] MISS — best similarity: ${(bestSimilarity * 100).toFixed(1)}%`);
    return null;
  }
}
```

### Use Case 2: Per-Tenant Isolated Cache in Cluster

```javascript
class TenantAwareCache {
  constructor(cluster) {
    this.cluster = cluster;
  }

  // Tenant keys use the tenant ID as the hash tag
  // → All data for one tenant lands on the same node
  tenantKey(tenantId, type, id) {
    return `{tenant:${tenantId}}:${type}:${id}`;
  }

  async getEmbedding(tenantId, documentId) {
    return this.cluster.get(this.tenantKey(tenantId, 'embedding', documentId));
  }

  async setEmbedding(tenantId, documentId, embedding, ttl = 86400 * 30) {
    return this.cluster.setex(
      this.tenantKey(tenantId, 'embedding', documentId),
      ttl,
      JSON.stringify(embedding)
    );
  }

  // One pipeline call to load all tenant context (co-located via hash tag)
  async loadTenantContext(tenantId, documentIds) {
    const pipeline = this.cluster.pipeline();
    documentIds.forEach(docId => {
      pipeline.get(this.tenantKey(tenantId, 'embedding', docId));
      pipeline.get(this.tenantKey(tenantId, 'metadata', docId));
    });
    const results = await pipeline.exec();
    return results.map(([, value]) => value ? JSON.parse(value) : null);
  }

  // Invalidate ALL cache for a tenant when their data is deleted (GDPR)
  async purgeTenant(tenantId) {
    // Find the node that owns {tenant:X} keys
    const sampleKey = `{tenant:${tenantId}}:_sentinel`;
    const nodeForTenant = this.cluster.getInfoFromServer('master', sampleKey);

    let cursor = '0';
    let purged = 0;
    do {
      const [nextCursor, keys] = await nodeForTenant.scan(
        cursor, 'MATCH', `{tenant:${tenantId}}:*`, 'COUNT', 100
      );
      cursor = nextCursor;
      if (keys.length > 0) {
        await nodeForTenant.del(...keys);
        purged += keys.length;
      }
    } while (cursor !== '0');

    console.log(`[Cache] Purged ${purged} keys for tenant ${tenantId} (GDPR)`);
    return purged;
  }
}
```

### Use Case 3: Distributed Rate Limiting with Redis Cluster

```javascript
class DistributedRateLimiter {
  constructor(cluster) {
    this.cluster = cluster;
  }

  // Sliding window rate limiter — works correctly across distributed Redis
  // Hash tag ensures all rate limit keys for one user land on the same node
  async isAllowed(userId, { maxRequests = 100, windowSeconds = 60 } = {}) {
    const now = Date.now();
    const windowStart = now - windowSeconds * 1000;

    // All keys for this user use {user:userId} hash tag → same node → atomic ops
    const key = `{ratelimit:${userId}}:requests`;

    const pipeline = this.cluster.pipeline();
    // Remove expired entries from the sorted set
    pipeline.zremrangebyscore(key, '-inf', windowStart);
    // Add current timestamp
    pipeline.zadd(key, now, `${now}-${Math.random()}`);
    // Count total in window
    pipeline.zcard(key);
    // Set TTL
    pipeline.expire(key, windowSeconds * 2);

    const results = await pipeline.exec();
    const requestCount = results[2][1]; // Result of ZCARD

    if (requestCount > maxRequests) {
      const oldestEntry = await this.cluster.zrange(key, 0, 0, 'WITHSCORES');
      const retryAfterMs = oldestEntry.length > 1
        ? (parseInt(oldestEntry[1]) + windowSeconds * 1000) - now
        : windowSeconds * 1000;

      return {
        allowed: false,
        requestCount,
        maxRequests,
        retryAfterMs: Math.ceil(retryAfterMs)
      };
    }

    return { allowed: true, requestCount, maxRequests };
  }
}

// Express middleware
const rateLimiter = new DistributedRateLimiter(redisCluster);

app.use('/api/generate', async (req, res, next) => {
  const result = await rateLimiter.isAllowed(req.user.id, {
    maxRequests: req.user.tier === 'premium' ? 500 : 100,
    windowSeconds: 60
  });

  if (!result.allowed) {
    return res.status(429).json({
      error: 'Rate limit exceeded',
      retryAfterMs: result.retryAfterMs
    });
  }
  next();
});
```

---

## Advanced Patterns

### Pattern: Stampede Protection with Distributed Locking

When a hot cache key expires, all instances try to recompute it simultaneously — the cache stampede. Use a distributed lock so only one instance recomputes while others wait.

```javascript
class StampedeProtectedCache {
  constructor(cluster, { lockTtlSeconds = 30 } = {}) {
    this.cluster = cluster;
    this.lockTtl = lockTtlSeconds;
  }

  async getOrCompute(key, computeFn, ttlSeconds = 3600) {
    // Fast path: check cache
    const cached = await this.cluster.get(key);
    if (cached) return JSON.parse(cached);

    // Cache miss — try to acquire distributed lock
    const lockKey = `lock:${key}`;
    const lockValue = `${process.pid}:${Date.now()}`;  // Unique to this instance

    // SET NX EX = "set only if not exists, with expiry" — atomic in Redis
    const lockAcquired = await this.cluster.set(lockKey, lockValue, 'NX', 'EX', this.lockTtl);

    if (lockAcquired === 'OK') {
      // We hold the lock — compute the value
      console.log(`[Cache] Lock acquired for key: ${key} — computing`);
      try {
        const value = await computeFn();
        await this.cluster.setex(key, ttlSeconds, JSON.stringify(value));
        return value;
      } finally {
        // Release lock only if we still own it (Lua for atomicity)
        await this.cluster.eval(
          `if redis.call("get", KEYS[1]) == ARGV[1] then
             return redis.call("del", KEYS[1])
           else return 0 end`,
          1, lockKey, lockValue
        );
      }
    } else {
      // Another instance is computing — wait and retry
      console.log(`[Cache] Lock held by another instance — waiting for: ${key}`);
      await new Promise(r => setTimeout(r, 200 + Math.random() * 300));

      // Retry: the lock-holder should have populated cache by now
      const recheckCached = await this.cluster.get(key);
      if (recheckCached) return JSON.parse(recheckCached);

      // If still not cached (compute took too long), compute ourselves
      const value = await computeFn();
      await this.cluster.setex(key, ttlSeconds, JSON.stringify(value));
      return value;
    }
  }
}

// Usage: prevents 500 simultaneous OpenAI calls when a popular cache key expires
const stampedeCache = new StampedeProtectedCache(redisCluster);

async function getLLMResponse(prompt) {
  const key = `llm:${hashString(prompt)}`;
  return stampedeCache.getOrCompute(
    key,
    () => openai.chat.completions.create({ model: 'gpt-4', messages: [{ role: 'user', content: prompt }] }),
    3600
  );
}
```

### Pattern: Tiered Cache (L1 In-Memory + L2 Redis Cluster)

```javascript
const NodeCache = require('node-cache');

class TieredDistributedCache {
  constructor(cluster, { l1TtlSeconds = 30, l2TtlSeconds = 3600 } = {}) {
    // L1: In-process memory — sub-millisecond, but per-instance and small
    this.l1 = new NodeCache({ stdTTL: l1TtlSeconds, useClones: false });
    // L2: Redis Cluster — 2–5ms, shared across all instances, large
    this.l2 = cluster;
    this.l2Ttl = l2TtlSeconds;
  }

  async get(key) {
    // L1: check in-memory first (~0.01ms)
    const l1Value = this.l1.get(key);
    if (l1Value !== undefined) {
      return { value: l1Value, tier: 'L1' };
    }

    // L2: check Redis cluster (~2–5ms)
    const l2Value = await this.l2.get(key);
    if (l2Value) {
      const parsed = JSON.parse(l2Value);
      this.l1.set(key, parsed); // Promote to L1
      return { value: parsed, tier: 'L2' };
    }

    return { value: null, tier: 'MISS' };
  }

  async set(key, value) {
    this.l1.set(key, value);
    await this.l2.setex(key, this.l2Ttl, JSON.stringify(value));
  }

  async invalidate(key) {
    this.l1.del(key);
    await this.l2.del(key);
    // Broadcast to all app server L1 caches via pub/sub
    await this.l2.publish('cache:invalidate:l1', key);
  }
}

// Typical latencies in production:
// L1 hit: 0.01ms  (in-process memory)
// L2 hit: 2–5ms   (Redis cluster, local network)
// Miss:   500–2000ms (LLM API call)
```

### Pattern: Cross-Slot Lua Scripts for Atomic Operations

When you need atomic operations across multiple keys that share a hash tag (same slot), Lua scripts run atomically on Redis — no other commands can interleave.

```javascript
// Atomic: check quota AND increment — cannot have a race condition
const checkAndIncrementQuota = `
  local current = tonumber(redis.call('GET', KEYS[1]) or '0')
  local limit = tonumber(ARGV[1])
  if current >= limit then
    return -1  -- quota exceeded
  end
  redis.call('INCR', KEYS[1])
  redis.call('EXPIRE', KEYS[1], ARGV[2])
  return current + 1
`;

async function consumeTokenQuota(tenantId, tokensUsed, dailyLimit) {
  // Hash tag ensures quota key and usage key are on same node
  const quotaKey = `{tenant:${tenantId}}:token_quota:${today()}`;

  const result = await cluster.eval(
    checkAndIncrementQuota,
    1,          // number of keys
    quotaKey,   // KEYS[1]
    dailyLimit, // ARGV[1]
    86400       // ARGV[2] — 24h TTL
  );

  if (result === -1) {
    throw new Error(`Tenant ${tenantId} has exceeded daily token quota of ${dailyLimit}`);
  }

  return { used: result, limit: dailyLimit, remaining: dailyLimit - result };
}
```

---

## Monitoring & Observability

### Key Metrics to Track

```javascript
class RedisClusterMonitor {
  constructor(cluster) {
    this.cluster = cluster;
  }

  async getClusterHealth() {
    const masterNodes = this.cluster.nodes('master');
    const allNodes    = this.cluster.nodes('all');

    const nodeStats = await Promise.all(
      masterNodes.map(async node => {
        const info = await node.info();
        const stats = this._parseInfo(info);

        return {
          host:              node.options.host,
          port:              node.options.port,
          role:              'master',
          usedMemoryMB:      (parseInt(stats['used_memory']) / 1024 / 1024).toFixed(1),
          maxMemoryMB:       (parseInt(stats['maxmemory'] || 0) / 1024 / 1024).toFixed(1),
          memoryUsagePct:    stats['maxmemory'] > 0
                               ? ((stats['used_memory'] / stats['maxmemory']) * 100).toFixed(1) + '%'
                               : 'no limit',
          connectedClients:  stats['connected_clients'],
          keyspaceHits:      stats['keyspace_hits'],
          keyspaceMisses:    stats['keyspace_misses'],
          hitRate:           this._calcHitRate(stats['keyspace_hits'], stats['keyspace_misses']),
          opsPerSecond:      stats['instantaneous_ops_per_sec'],
          evictedKeys:       stats['evicted_keys'],         // ⚠️ > 0 means RAM pressure
          replicationLagMs:  stats['master_repl_offset']
        };
      })
    );

    const clusterInfo = await masterNodes[0].cluster('INFO');
    const clusterOk = clusterInfo.includes('cluster_state:ok');

    return {
      clusterState:      clusterOk ? 'OK' : 'DEGRADED',
      totalNodes:        allNodes.length,
      masterCount:       masterNodes.length,
      replicaCount:      allNodes.length - masterNodes.length,
      nodes:             nodeStats,
      alerts:            this._generateAlerts(nodeStats, clusterOk)
    };
  }

  _calcHitRate(hits, misses) {
    const total = parseInt(hits) + parseInt(misses);
    if (total === 0) return '0%';
    return `${((parseInt(hits) / total) * 100).toFixed(1)}%`;
  }

  _generateAlerts(nodeStats, clusterOk) {
    const alerts = [];
    if (!clusterOk) alerts.push({ level: 'CRITICAL', message: 'Cluster state is not OK' });

    for (const node of nodeStats) {
      const memPct = parseFloat(node.memoryUsagePct);
      if (memPct > 85) alerts.push({ level: 'CRITICAL', message: `Node ${node.host} memory at ${memPct}%` });
      else if (memPct > 70) alerts.push({ level: 'WARNING', message: `Node ${node.host} memory at ${memPct}%` });

      if (parseInt(node.evictedKeys) > 0) {
        alerts.push({ level: 'WARNING', message: `Node ${node.host} evicting keys — increase memory` });
      }

      const hitRate = parseFloat(node.hitRate);
      if (hitRate < 70) alerts.push({ level: 'WARNING', message: `Node ${node.host} hit rate ${hitRate}% — investigate TTLs` });
    }

    return alerts;
  }

  _parseInfo(infoString) {
    return Object.fromEntries(
      infoString.split('\r\n')
        .filter(line => line.includes(':'))
        .map(line => line.split(':'))
    );
  }
}

// Express health endpoint
app.get('/health/redis', async (req, res) => {
  const health = await monitor.getClusterHealth();
  const statusCode = health.clusterState === 'OK' ? 200 : 503;
  res.status(statusCode).json(health);
});

// Example output:
// {
//   "clusterState": "OK",
//   "totalNodes": 6,
//   "masterCount": 3,
//   "replicaCount": 3,
//   "nodes": [
//     {
//       "host": "redis-a",
//       "usedMemoryMB": "8241.3",
//       "memoryUsagePct": "51.5%",
//       "hitRate": "91.2%",
//       "opsPerSecond": "4821",
//       "evictedKeys": "0"
//     },
//     ...
//   ],
//   "alerts": []
// }
```

### Prometheus Alerting Rules

```yaml
# Redis Cluster alerting rules
groups:
  - name: redis_cluster
    rules:
      - alert: RedisClusterDegraded
        expr: redis_cluster_state != 1
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Redis Cluster is not OK"
          description: "Cluster state: {{ $value }}. Check for failed nodes."

      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis node {{ $labels.instance }} memory > 85%"
          description: "Memory at {{ $value | humanizePercentage }}. Add nodes or increase limits."

      - alert: RedisHighEvictionRate
        expr: increase(redis_evicted_keys_total[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Redis evicting keys on {{ $labels.instance }}"
          description: "RAM is full — cache hit rate will drop. Increase memory or add nodes."

      - alert: RedisCacheHitRateLow
        expr: |
          rate(redis_keyspace_hits_total[5m]) /
          (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) < 0.70
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis cache hit rate below 70% on {{ $labels.instance }}"
          description: "Hit rate: {{ $value | humanizePercentage }}. API costs will spike."
```

---

## Integration with Other Patterns

### Redis Cluster + Cache-Aside

Redis Cluster IS the backing store for the Cache-Aside pattern. The cluster handles routing, failover, and capacity transparently — the Cache-Aside logic stays the same regardless of whether there's one Redis or twenty.

```javascript
class ClusterBackedCacheAside {
  constructor(cluster, llm) {
    this.cluster = cluster;
    this.llm = llm;
  }

  async complete(prompt) {
    const key = `llm:${hashString(prompt)}`;

    // Cache-Aside logic unchanged — cluster handles distribution
    const cached = await this.cluster.get(key);
    if (cached) return { ...JSON.parse(cached), source: 'cache' };

    const response = await this.llm.complete(prompt);
    await this.cluster.setex(key, 3600, JSON.stringify(response));
    return { ...response, source: 'api' };
  }
}
```

### Redis Cluster + Bulkhead

Each tier of your bulkhead can have its own Redis cluster or namespace, preventing one tenant's cache activity from impacting another's.

```javascript
// Premium tier: dedicated Redis cluster for guaranteed performance
// Free tier: shared Redis cluster with lower priority
const premiumCache = new RedisClusterClient(premiumClusterNodes);
const freeCache    = new RedisClusterClient(freeClusterNodes);

async function getCachedResponse(userId, tier, prompt) {
  const cache = tier === 'premium' ? premiumCache : freeCache;
  return cache.get(`llm:${hashString(prompt)}`);
}
```

### Redis Cluster + Circuit Breaker

Wrap Redis operations in a circuit breaker. If the cluster is unreachable, fail fast and fall through to the AI provider directly (degraded but functional).

```javascript
const CircuitBreaker = require('opossum');

class FaultTolerantClusterCache {
  constructor(cluster) {
    this.breaker = new CircuitBreaker(
      async (key) => cluster.get(key),
      { timeout: 500, errorThresholdPercentage: 25, resetTimeout: 10000 }
    );

    this.breaker.on('open', () =>
      console.error('Redis cluster circuit OPEN — bypassing cache')
    );
  }

  async get(key) {
    try {
      return await this.breaker.fire(key);
    } catch (err) {
      // Circuit open or Redis error → return null (cache miss)
      // App falls through to AI API — degraded but not broken
      return null;
    }
  }
}
```

### Redis Cluster + Sharding (Combined)

When you shard your application data by tenant, use hash tags to ensure each tenant's cache data lands on a predictable node — enabling per-tenant cache eviction and GDPR purges.

```javascript
// Tenant-sharded application data + Redis Cluster
// Hash tag {tenant:X} guarantees all tenant cache on same Redis node
// This also means you can evict a single tenant's cache without scanning all nodes

const tenantCacheKey = (tenantId, type, id) => `{tenant:${tenantId}}:${type}:${id}`;
```

---

## Real-World Case Studies

### Case Study 1: AI Writing Platform — Scaling Through a Viral Moment

**Problem:** Single Redis instance, 16GB. Email blast → 7x traffic spike → Redis OOM → eviction cascade → API cost explosion → platform degradation.

**Solution:** Redis Cluster (3 primaries × 16GB = 48GB). Reads distributed to replicas. Semantic cache layer for LLM responses.

**Results:**
- ✅ Cache capacity: 16GB → 48GB (3x with same per-node cost)
- ✅ Read throughput: 3x (reads served by replicas)
- ✅ Viral event survived with zero degradation
- ✅ API cost during spike: $4,100 → $180 (cache absorbed it)
- ✅ Node failure impact: 8 seconds vs 23 minutes full outage

### Case Study 2: Multi-Tenant AI Analytics SaaS

**Problem:** 200 enterprise tenants sharing one Redis. Large tenant's cache activity (10M+ keys) triggered LRU eviction for smaller tenants, dropping their hit rates. Noisy neighbor in cache layer.

**Solution:** Hash tag-based tenant isolation in Redis Cluster. All `{tenant:X}:*` keys land on the same node. Large tenants assigned dedicated node pairs via Redis Cluster's `CLUSTER SETSLOT` to physically isolate their keyspace.

**Results:**
- ✅ Small tenant hit rate: 71% → 94% (no more eviction from large tenants)
- ✅ Large tenant migration: <2 hours, zero downtime
- ✅ GDPR purge per tenant: automated scan of single node (was full cluster scan)
- ✅ Per-tenant cache cost attribution: now possible (count keys per hash tag prefix)

### Case Study 3: AI Inference API — Global Distribution

**Problem:** Single-region Redis. EU users: 180ms cache read latency (trans-Atlantic). US users: 2ms. EU customers demanding SLA improvement.

**Solution:** Redis Cluster per region (US-EAST + EU-WEST). Write-through from origin region. Reads always go to local cluster. Global semantic cache index synchronized via stream replication.

```
User request (EU) → EU Redis Cluster → cache hit → 2ms response
                                     → cache miss → US origin API
                                                  → populate EU cluster
                                                  → 210ms (first miss only)
```

**Results:**
- ✅ EU cache read latency: 180ms → 2.4ms
- ✅ EU cache hit rate: 91% (after warm-up period)
- ✅ EU user satisfaction: NPS +34 points
- ✅ Cross-region API calls: reduced 89% (EU cluster handles most)
- ✅ GDPR compliance: EU data stays in EU cluster

---

## Production Best Practices

### 1. Cluster Sizing Formula

```javascript
class ClusterSizingCalculator {
  static recommend({
    totalDataGB,
    dailyRequestsMillions,
    targetHitRate = 0.85,
    avgValueSizeKB = 4,
    peakMultiplier = 3,        // Peak traffic vs average
    memoryHeadroomPct = 0.70   // Keep nodes at 70% max to avoid eviction
  }) {
    // How much RAM do we need to hold the working set?
    const workingSetGB = totalDataGB * targetHitRate;
    const ramNeededGB  = workingSetGB / memoryHeadroomPct;

    // How many nodes (16GB each is common)?
    const nodeRAMGB   = 16;
    const primariesNeeded = Math.ceil(ramNeededGB / nodeRAMGB);

    // Peak read ops per second
    const avgRPS  = (dailyRequestsMillions * 1_000_000) / 86400;
    const peakRPS = avgRPS * peakMultiplier;

    // Each Redis node handles ~100K ops/sec
    const nodesForThroughput = Math.ceil(peakRPS / 100_000);

    const primaries = Math.max(primariesNeeded, nodesForThroughput, 3); // Minimum 3 for cluster

    return {
      primaries,
      replicas: primaries,       // 1 replica per primary
      totalNodes: primaries * 2,
      totalRAMGB: primaries * nodeRAMGB,
      estimatedHitRate: `${(targetHitRate * 100).toFixed(0)}%`,
      peakOpsPerSecond: Math.round(peakRPS),
      recommendation: `Redis Cluster: ${primaries} primaries + ${primaries} replicas`
    };
  }
}

// Example for mid-size AI platform
const plan = ClusterSizingCalculator.recommend({
  totalDataGB: 80,               // 80GB of cacheable data
  dailyRequestsMillions: 50,     // 50M requests/day
  targetHitRate: 0.88,
  peakMultiplier: 5              // 5x spike possible (viral, marketing)
});
// → { primaries: 6, totalNodes: 12, totalRAMGB: 96, peakOpsPerSecond: 2893519 }
```

### 2. Key Design Rules

```javascript
// ✅ GOOD key patterns
'llm:{userId}:response:{queryHash}'    // Hash tag for user-level operations
'{tenant:acme}:embed:{docId}'          // Hash tag for tenant isolation
'rate:{userId}:minute:{minute}'        // Rate limiting key with time window
'session:{sessionId}'                  // No hash tag needed — sessions are independent

// ❌ BAD key patterns
'response:' + JSON.stringify(fullPrompt)  // Key too long — hash the content
'cache:' + new Date().toString()          // Volatile key — can't reliably retrieve
'user_data'                               // Too generic — collisions guaranteed
'{global}:all_users'                      // Hash tag forcing ALL keys to one node

// ✅ Key naming convention
// Format: {hash-tag}:resource-type:identifier[:qualifier]
// Example: {tenant:acme}:embedding:doc-123:v2
```

### 3. TTL Strategy for AI Workloads

```javascript
const AI_CACHE_TTL = {
  // Stable content — long TTL
  documentEmbeddings:    86400 * 30,  // 30 days (documents rarely change)
  modelMetadata:         86400 * 7,   // 7 days

  // Moderately stable
  llmFaqResponses:       3600,        // 1 hour
  searchResults:         1800,        // 30 minutes
  userEmbeddings:        86400,       // 24 hours (user preferences change slowly)

  // Volatile
  rateLimitCounters:     60,          // 1 minute rolling window
  sessionData:           1800,        // 30 minute session
  streamingTokens:       30,          // Very short — streaming completions

  // Never cache in Redis
  personalizedContent:   0,           // Per-user, per-context — not cacheable
  realtimeData:          0,           // Stock prices, live feeds
  transactionState:      0            // Use a proper DB for this
};
```

### 4. Testing Distributed Cache Behavior

```javascript
describe('Redis Cluster Cache', () => {
  it('should route the same key to the same node consistently', async () => {
    const key = 'test:consistency:key1';
    await cluster.set(key, 'value-a');

    // Retrieve 10 times — must always get the same value
    const results = await Promise.all(
      Array.from({ length: 10 }, () => cluster.get(key))
    );
    expect(new Set(results).size).toBe(1); // All identical
    expect(results[0]).toBe('value-a');
  });

  it('should pipeline keys with same hash tag atomically', async () => {
    const pipeline = cluster.pipeline();
    pipeline.set('{user:123}:a', 'value-a');
    pipeline.set('{user:123}:b', 'value-b');
    pipeline.set('{user:123}:c', 'value-c');
    const results = await pipeline.exec();

    // No CROSSSLOT errors — all on same node
    results.forEach(([err]) => expect(err).toBeNull());
  });

  it('should survive node failure and continue serving requests', async () => {
    // Write some test data
    await cluster.set('failover:test', 'value');

    // Simulate node failure (stop one Redis process)
    // In test: disconnect a node via cluster FAILOVER command
    const nodes = cluster.nodes('master');
    await nodes[0].cluster('FAILOVER', 'FORCE'); // Force replica takeover

    // Wait for failover (~10 seconds)
    await new Promise(r => setTimeout(r, 15000));

    // Cache should still work after failover
    const result = await cluster.get('failover:test');
    expect(result).toBe('value');
  }, 30000);
});
```

### When to Use Each Redis Distribution Strategy

```
Single Primary + Replica:
  ✅ Data fits in one node's RAM (< 12GB working set)
  ✅ Need read scaling but not write scaling
  ✅ Acceptable to have manual failover (or Sentinel)
  ❌ Do NOT use if a 30s+ failover is unacceptable

Redis Sentinel:
  ✅ Need automatic failover (~30s)
  ✅ Data still fits in one node's RAM
  ✅ Simple ops team, limited Redis expertise
  ❌ Do NOT use if data exceeds a single node's capacity

Redis Cluster:
  ✅ Data exceeds single-node capacity (> 12GB working set)
  ✅ Need both horizontal scale AND high availability
  ✅ Peak traffic justifies distributed read load
  ❌ More complex: no cross-slot multi-key ops without hash tags
  ❌ Do NOT use for single-server hobby projects

Client-Side Consistent Hashing:
  ✅ Managed Redis provider doesn't support cluster mode
  ✅ Want full control over routing logic
  ✅ Need custom placement policies (e.g., always put tenant on node 2)
  ❌ You own the sharding logic — more code to maintain
```

### ROI Summary

**Investment:**
- Redis Cluster setup + migration: 1–2 weeks = $8K
- Cluster-aware client refactoring: 3 days = $3K
- Monitoring + alerting: 2 days = $2K
- **Total: ~$13K**

**Returns (Annual, mid-size AI SaaS):**
- API cost reduction from sustained high hit rate: $180K
- Prevented outage incidents (2/year × $180K): $360K
- Engineering time saved on capacity firefighting: $60K
- Enterprise deals unlocked by SLA compliance: $200K
- **Total: $800K/year**

**ROI: 6,154% first year**

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
