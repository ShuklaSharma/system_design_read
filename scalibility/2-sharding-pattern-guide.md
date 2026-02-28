# Sharding Pattern for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why Sharding Matters for AI Systems](#why-sharding-matters)
3. [Sharding Strategies](#sharding-strategies)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Monitoring & Observability](#monitoring-observability)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Real-World Case Studies](#real-world-case-studies)
10. [Production Best Practices](#production-best-practices)

---

## Introduction

### What is Sharding?

**Sharding** is a horizontal scaling pattern that partitions data or workload across multiple independent nodes (shards), each responsible for a subset of the total dataset or traffic. Instead of one large database or service handling everything, many smaller ones share the load.

**Analogy:** Like a library with thousands of books — instead of one librarian handling every request from every patron, you divide the library into sections (A–F, G–M, N–Z). Each librarian manages their section independently. Any patron needing a book goes directly to the right librarian — fast, parallel, scalable.

### The Problem Without Sharding

```
Scenario: AI platform storing 500M vector embeddings

Without Sharding:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Single PostgreSQL + pgvector instance

Month 1:   10M vectors   → 200ms query latency  ✅
Month 3:   50M vectors   → 450ms query latency  ⚠️
Month 6:  150M vectors   → 1.2s query latency   ❌
Month 9:  300M vectors   → 4.8s query latency   ❌❌
Month 12: 500M vectors   → DB crashes on query  💥

Other failures:
- Adding index: 18-hour table lock → full outage
- Single point of failure: DB goes down = all AI features down
- Can't scale reads independently from writes
- Backup takes 14 hours → long recovery time
- One noisy tenant slows all queries for everyone
```

### The Solution With Sharding

```
With Sharding (8 shards):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Each shard: 62.5M vectors

Month 1:   10M vectors   → 18ms query latency   ✅
Month 3:   50M vectors   → 22ms query latency   ✅
Month 6:  150M vectors   → 31ms query latency   ✅
Month 9:  300M vectors   → 38ms query latency   ✅
Month 12: 500M vectors   → 45ms query latency   ✅

Additional benefits:
- Adding index per shard: 2.5 hours, no global lock
- Shard failure: only 1/8 of data affected
- Read replicas per shard for read scaling
- Backup per shard: 1.8 hours, parallel
- Tenant isolation: noisy neighbor contained
```

---

## Why Sharding Matters for AI Systems

### Unique AI System Challenges

1. **Massive Vector Datasets**
   - Each embedding: 1,536 floats × 4 bytes = 6KB per vector
   - 100M embeddings = 600GB of raw vector data
   - Similarity search complexity: O(n) without index, O(log n) with
   - Index size can exceed raw data size (HNSW indices are large)

2. **Mixed Read/Write Workloads**
   - RAG pipelines: heavy reads (search) + heavy writes (indexing)
   - Model serving: high read throughput, low write frequency
   - Fine-tuning pipelines: batch write bursts, then heavy reads
   - Real-time personalization: equal reads and writes

3. **Multi-Tenant AI Platforms**
   - Enterprise customers have 10M+ documents each
   - Free tier users share infrastructure with paying customers
   - Tenant data must be isolated for compliance (GDPR, HIPAA)
   - One tenant's heavy indexing job cannot affect others

4. **AI Model Weight Storage**
   - Large language model weights: 7B params = ~14GB per shard
   - Serving multiple models requires distributing across GPUs
   - Fine-tuned model variants multiply storage requirements
   - Inference routing must consider data locality

### Real Incident: No Sharding

**AI Document Intelligence Platform — Series B Startup**

```
Timeline:
Jan 2024 - Platform launches: 500 customers, 5M documents
Mar 2024 - Growth: 2,000 customers, 25M documents
           → Query latency: 380ms (acceptable)
Jun 2024 - Viral growth: 8,000 customers, 100M documents
           → Query latency: 3.2 seconds (users complaining)
Jul 2024 - Enterprise customer onboards with 80M documents
Jul 10   - Onboarding job starts indexing 80M documents
Jul 10   - 14:00: Indexing job begins
           14:30: CPU usage: 94%
           15:00: All other tenant queries: 12s latency
           15:30: Timeouts across platform
           16:00: 400+ support tickets in 1 hour
           16:15: Manually kill indexing job
           16:30: Performance recovers (but enterprise customer angry)
Aug 2024 - Engineering scramble: emergency sharding project
           4 engineers × 6 weeks = $120K engineering cost
           Platform half-frozen during migration

Impact:
- 6 weeks of degraded performance
- Lost 23% of customers due to reliability issues
- Enterprise customer demanded SLA credits: $180K
- Emergency engineering cost: $120K
- Total impact: $1.4M
```

**Same Scenario WITH Sharding from Day One:**

```
Timeline:
Jan 2024 - Platform launches: 8 shards, data spread evenly
Jun 2024 - Growth: 100M documents → 12.5M per shard
           → Query latency: 28ms (excellent)
Jul 2024 - Enterprise customer onboards with 80M documents
           → Assigned to dedicated shard group (tenant isolation)
Jul 10   - 14:00: Enterprise indexing job begins
           → Isolated to enterprise shard group
           14:30: Enterprise shard CPU: 90%
           15:00: All other tenant queries: 28ms (unaffected!)
           16:00: Indexing completes, enterprise shard normalizes

Impact:
- Zero degradation for other tenants
- Enterprise customer onboarded successfully
- No support tickets related to indexing
- Platform continues growing normally
```

---

## Sharding Strategies

### 1. Hash-Based Sharding

A hash function maps a key (e.g., `tenant_id`, `document_id`) to a shard. Distributes data evenly.

```
shard = hash(key) % num_shards

hash("tenant_001") % 8 = Shard 2
hash("tenant_002") % 8 = Shard 5
hash("tenant_003") % 8 = Shard 2
```

**Pros:** Even distribution, simple to implement
**Cons:** Resharding is painful (most keys move when shard count changes)
**Use For:** User data, session data, tenant isolation

### 2. Range-Based Sharding

Data is split by value ranges (alphabetical, numeric, date).

```
Shard 0: document_id  0 – 10,000,000
Shard 1: document_id  10,000,001 – 20,000,000
Shard 2: document_id  20,000,001 – 30,000,000
```

**Pros:** Efficient range queries, easy to add shards at the end
**Cons:** Hot spots if data is not evenly distributed (e.g., recent dates)
**Use For:** Time-series AI logs, document ingestion pipelines, ordered IDs

### 3. Directory-Based Sharding

A lookup table (shard directory) maps keys to shards. Flexible but adds a lookup hop.

```
Shard Directory:
  tenant_001 → Shard 3 (large tenant, isolated)
  tenant_002 → Shard 1
  tenant_003 → Shard 1
  tenant_004 → Shard 7 (enterprise, dedicated)
```

**Pros:** Full control over placement, easy to move tenants between shards
**Cons:** Directory is a single point of failure, adds latency
**Use For:** Multi-tenant SaaS, VIP tenant isolation

### 4. Consistent Hashing

Uses a virtual ring. Nodes are placed on the ring; keys route to the next node clockwise. Adding/removing nodes only remaps a fraction of keys.

```
          Node A (0°)
         /           \
  Node D (270°)    Node B (90°)
         \           /
          Node C (180°)

Key "doc_123" hashes to 140° → routes to Node C
Add Node E at 160° → only keys 140°–160° remapped
```

**Pros:** Minimal data movement when scaling, no rehashing all keys
**Cons:** Can cause uneven distribution without virtual nodes
**Use For:** Distributed caches, vector DBs, AI embedding stores

### 5. Geographic Sharding

Data is partitioned by region (US, EU, APAC). Satisfies data residency requirements and reduces latency.

```
EU Shard:   All documents from European tenants (GDPR)
US Shard:   All documents from US tenants
APAC Shard: All documents from Asia-Pacific tenants
```

**Pros:** Regulatory compliance, low latency for regional users
**Cons:** Cross-region queries are expensive, harder to rebalance
**Use For:** GDPR-compliant AI platforms, global SaaS

---

## Implementation Patterns

### Pattern 1: Hash-Based Shard Router

```javascript
const crypto = require('crypto');
const { Pool } = require('pg');

class ShardRouter {
  constructor(shardConfigs) {
    // Initialize a connection pool per shard
    this.shards = shardConfigs.map(config => new Pool(config));
    this.numShards = shardConfigs.length;
  }

  // Determine which shard owns a given key
  getShardIndex(key) {
    const hash = crypto.createHash('md5').update(String(key)).digest('hex');
    const hashInt = parseInt(hash.substring(0, 8), 16);
    return hashInt % this.numShards;
  }

  getShard(key) {
    return this.shards[this.getShardIndex(key)];
  }

  // Execute a query on the correct shard
  async query(key, sql, params = []) {
    const shard = this.getShard(key);
    const shardIndex = this.getShardIndex(key);
    console.log(`Routing key "${key}" to shard ${shardIndex}`);
    return shard.query(sql, params);
  }

  // Scatter-gather: query ALL shards, merge results
  async queryAll(sql, params = []) {
    const promises = this.shards.map(shard => shard.query(sql, params));
    const results = await Promise.all(promises);
    return results.flatMap(r => r.rows);
  }
}

// Configuration: 4 PostgreSQL shards
const router = new ShardRouter([
  { host: 'shard-0.db.internal', database: 'ai_vectors', port: 5432 },
  { host: 'shard-1.db.internal', database: 'ai_vectors', port: 5432 },
  { host: 'shard-2.db.internal', database: 'ai_vectors', port: 5432 },
  { host: 'shard-3.db.internal', database: 'ai_vectors', port: 5432 }
]);

// Store a document embedding — routes to correct shard
async function storeEmbedding(tenantId, documentId, embedding) {
  return router.query(
    tenantId, // Shard by tenantId (tenant isolation)
    `INSERT INTO embeddings (tenant_id, document_id, embedding, created_at)
     VALUES ($1, $2, $3, NOW())
     ON CONFLICT (document_id) DO UPDATE SET embedding = $3`,
    [tenantId, documentId, JSON.stringify(embedding)]
  );
}

// Retrieve embedding — routes to same shard automatically
async function getEmbedding(tenantId, documentId) {
  const result = await router.query(
    tenantId,
    `SELECT embedding FROM embeddings WHERE tenant_id = $1 AND document_id = $2`,
    [tenantId, documentId]
  );
  return result.rows[0]?.embedding;
}

// Cross-shard search (scatter-gather)
async function globalSearch(query) {
  return router.queryAll(
    `SELECT document_id, similarity FROM embeddings
     ORDER BY embedding <-> $1 LIMIT 10`,
    [query]
  );
}
```

### Pattern 2: Consistent Hashing for Vector Store

```javascript
class ConsistentHashRing {
  constructor(nodes, virtualNodes = 150) {
    this.ring = new Map();
    this.sortedKeys = [];
    this.virtualNodes = virtualNodes;

    for (const node of nodes) {
      this.addNode(node);
    }
  }

  _hash(key) {
    const hash = crypto.createHash('md5').update(key).digest('hex');
    return parseInt(hash.substring(0, 8), 16);
  }

  addNode(node) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = this._hash(`${node}:${i}`);
      this.ring.set(virtualKey, node);
      this.sortedKeys.push(virtualKey);
    }
    this.sortedKeys.sort((a, b) => a - b);
    console.log(`Node "${node}" added with ${this.virtualNodes} virtual nodes`);
  }

  removeNode(node) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = this._hash(`${node}:${i}`);
      this.ring.delete(virtualKey);
    }
    this.sortedKeys = this.sortedKeys.filter(k => this.ring.has(k));
    console.log(`Node "${node}" removed. Data redistributed to neighbors.`);
  }

  getNode(key) {
    if (this.ring.size === 0) throw new Error('No nodes in ring');

    const hash = this._hash(key);

    // Find the first node clockwise from this hash
    for (const ringKey of this.sortedKeys) {
      if (hash <= ringKey) return this.ring.get(ringKey);
    }

    // Wrap around to the first node
    return this.ring.get(this.sortedKeys[0]);
  }
}

class ShardedVectorStore {
  constructor(nodeUrls) {
    this.ring = new ConsistentHashRing(nodeUrls);
    this.clients = {};

    for (const url of nodeUrls) {
      this.clients[url] = new VectorDBClient(url);
    }
  }

  async upsert(id, embedding, metadata = {}) {
    const node = this.ring.getNode(id);
    return this.clients[node].upsert({ id, embedding, metadata });
  }

  async get(id) {
    const node = this.ring.getNode(id);
    return this.clients[node].get(id);
  }

  // Add capacity — only ~1/n keys remapped (not all)
  async addNode(url) {
    this.clients[url] = new VectorDBClient(url);
    this.ring.addNode(url);
    // In production: trigger data migration for remapped keys
  }
}

// Usage
const store = new ShardedVectorStore([
  'http://qdrant-0:6333',
  'http://qdrant-1:6333',
  'http://qdrant-2:6333',
  'http://qdrant-3:6333'
]);

// Upsert routes to correct node
await store.upsert('doc_123', embeddingVector, { title: 'AI Guide' });

// Adding node 4 — only ~25% of keys migrate (not all 4 million)
await store.addNode('http://qdrant-4:6333');
```

### Pattern 3: Tenant-Isolated Sharding

```javascript
class TenantShardManager {
  constructor({ redis, shardPools }) {
    this.redis = redis;           // Shard directory (tenant → shard mapping)
    this.shardPools = shardPools; // Array of DB connection pools
    this.directoryKey = 'shard:directory';
  }

  // Get (or assign) a shard for a tenant
  async getShardForTenant(tenantId) {
    // Check directory first
    const assigned = await this.redis.hGet(this.directoryKey, tenantId);
    if (assigned !== null) return parseInt(assigned);

    // New tenant — assign to least-loaded shard
    const shardIndex = await this._leastLoadedShard();
    await this.redis.hSet(this.directoryKey, tenantId, String(shardIndex));
    console.log(`Tenant "${tenantId}" assigned to shard ${shardIndex}`);
    return shardIndex;
  }

  async _leastLoadedShard() {
    const loads = await Promise.all(
      this.shardPools.map(async (pool, i) => {
        const result = await pool.query(
          `SELECT COUNT(*) as count FROM documents`
        );
        return { index: i, count: parseInt(result.rows[0].count) };
      })
    );
    loads.sort((a, b) => a.count - b.count);
    return loads[0].index;
  }

  async query(tenantId, sql, params = []) {
    const shardIndex = await this.getShardForTenant(tenantId);
    const pool = this.shardPools[shardIndex];
    return pool.query(sql, params);
  }

  // Move a tenant to a different shard (e.g., after growth)
  async migrateTenant(tenantId, targetShardIndex) {
    const currentShardIndex = await this.getShardForTenant(tenantId);
    const currentPool = this.shardPools[currentShardIndex];
    const targetPool = this.shardPools[targetShardIndex];

    console.log(`Migrating tenant ${tenantId}: shard ${currentShardIndex} → ${targetShardIndex}`);

    // 1. Copy data to target shard
    const docs = await currentPool.query(
      `SELECT * FROM documents WHERE tenant_id = $1`, [tenantId]
    );

    for (const doc of docs.rows) {
      await targetPool.query(
        `INSERT INTO documents VALUES ($1, $2, $3, $4)
         ON CONFLICT (id) DO UPDATE SET content = $3, embedding = $4`,
        [doc.id, doc.tenant_id, doc.content, doc.embedding]
      );
    }

    // 2. Update directory
    await this.redis.hSet(this.directoryKey, tenantId, String(targetShardIndex));

    // 3. Delete from old shard (after confirming migration)
    await currentPool.query(
      `DELETE FROM documents WHERE tenant_id = $1`, [tenantId]
    );

    console.log(`Migration complete: ${docs.rows.length} documents moved`);
  }
}
```

### Pattern 4: API-Level Shard Router (Express Middleware)

```javascript
const express = require('express');

class ShardAwareAPIGateway {
  constructor({ shardManager, cache }) {
    this.shardManager = shardManager;
    this.cache = cache;
    this.app = express();
    this._setupRoutes();
  }

  // Middleware: identify shard from request context
  shardMiddleware() {
    return async (req, res, next) => {
      const tenantId = req.headers['x-tenant-id'] || req.query.tenant_id;
      if (!tenantId) return res.status(400).json({ error: 'Missing tenant_id' });

      req.tenantId = tenantId;
      req.shardIndex = await this.shardManager.getShardForTenant(tenantId);
      req.shard = this.shardManager.shardPools[req.shardIndex];

      next();
    };
  }

  _setupRoutes() {
    this.app.use(express.json());
    this.app.use(this.shardMiddleware());

    // Upsert document embedding
    this.app.post('/documents', async (req, res) => {
      const { documentId, content, embedding } = req.body;
      try {
        await req.shard.query(
          `INSERT INTO documents (id, tenant_id, content, embedding)
           VALUES ($1, $2, $3, $4)
           ON CONFLICT (id) DO UPDATE SET content = $3, embedding = $4`,
          [documentId, req.tenantId, content, JSON.stringify(embedding)]
        );
        res.json({ success: true, shard: req.shardIndex });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Semantic search — tenant-scoped, single shard
    this.app.post('/search', async (req, res) => {
      const { queryEmbedding, limit = 10 } = req.body;
      try {
        const result = await req.shard.query(
          `SELECT id, content, 1 - (embedding <=> $1) AS similarity
           FROM documents
           WHERE tenant_id = $2
           ORDER BY embedding <=> $1
           LIMIT $3`,
          [JSON.stringify(queryEmbedding), req.tenantId, limit]
        );
        res.json({ results: result.rows, shard: req.shardIndex });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Health per shard
    this.app.get('/health/shards', async (req, res) => {
      const health = await Promise.all(
        this.shardManager.shardPools.map(async (pool, i) => {
          try {
            const r = await pool.query('SELECT COUNT(*) AS docs FROM documents');
            return { shard: i, status: 'healthy', docCount: parseInt(r.rows[0].docs) };
          } catch {
            return { shard: i, status: 'unhealthy', docCount: 0 };
          }
        })
      );
      res.json({ shards: health });
    });
  }
}
```

---

## AI-Specific Use Cases

### Use Case 1: Sharding a Vector Database for RAG

```
Problem: Single Qdrant/Pinecone instance with 200M embeddings
         → Search latency: 4.2 seconds
         → Cannot add more vectors without degradation

Sharding Strategy: Hash-based by document_id, 8 shards

Before Sharding:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
200M vectors in 1 node
Search: full scan = 4.2s P99 latency
Index rebuild: 72 hours of downtime

After Sharding:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
25M vectors per shard × 8 shards
Search: parallel across shards = 0.18s P99 latency
Index rebuild per shard: 9 hours, rolling (no downtime)
```

```javascript
class ShardedRAGPipeline {
  constructor({ shardedVectorStore, llm, embedder }) {
    this.vectorStore = shardedVectorStore;
    this.llm = llm;
    this.embedder = embedder;
  }

  async index(tenantId, documentId, chunks) {
    // Embed all chunks in parallel
    const embeddings = await Promise.all(
      chunks.map(chunk => this.embedder.embed(chunk.text))
    );

    // Upsert to correct shard (keyed by tenantId)
    await Promise.all(
      chunks.map((chunk, i) =>
        this.vectorStore.upsert(
          `${tenantId}:${documentId}:${i}`,
          embeddings[i].embedding,
          { tenantId, documentId, text: chunk.text }
        )
      )
    );
  }

  async search(tenantId, question, topK = 10) {
    const { embedding } = await this.embedder.embed(question);

    // Tenant data lives on one shard — single-shard query
    const results = await this.vectorStore.tenantSearch(tenantId, embedding, topK);

    const context = results.map(r => r.metadata.text).join('\n\n');

    return this.llm.complete('gpt-4', [
      { role: 'system', content: `Answer using this context:\n${context}` },
      { role: 'user', content: question }
    ]);
  }
}
```

### Use Case 2: Sharding AI Model Weights Across GPUs

```python
import torch
import torch.distributed as dist

class ModelShardManager:
    """Distribute a large language model across multiple GPUs."""

    def __init__(self, model_name: str, num_gpus: int):
        self.model_name = model_name
        self.num_gpus = num_gpus
        self.shards = {}

    def load_sharded(self):
        """Load model layers distributed across available GPUs."""
        from transformers import AutoModelForCausalLM, AutoConfig

        config = AutoConfig.from_pretrained(self.model_name)
        total_layers = config.num_hidden_layers
        layers_per_gpu = total_layers // self.num_gpus

        print(f"Distributing {total_layers} layers across {self.num_gpus} GPUs")
        print(f"Layers per GPU: {layers_per_gpu}")

        # Device map: assign each layer to a GPU
        device_map = {}
        for layer_idx in range(total_layers):
            gpu_idx = layer_idx // layers_per_gpu
            gpu_idx = min(gpu_idx, self.num_gpus - 1)  # Clamp to last GPU
            device_map[f"model.layers.{layer_idx}"] = f"cuda:{gpu_idx}"

        # Embedding and LM head on GPU 0
        device_map["model.embed_tokens"] = "cuda:0"
        device_map["model.norm"] = f"cuda:{self.num_gpus - 1}"
        device_map["lm_head"] = f"cuda:{self.num_gpus - 1}"

        self.model = AutoModelForCausalLM.from_pretrained(
            self.model_name,
            device_map=device_map,
            torch_dtype=torch.float16
        )

        print("Model shards loaded:")
        for gpu_idx in range(self.num_gpus):
            gpu_layers = [k for k, v in device_map.items() if v == f"cuda:{gpu_idx}"]
            mem = torch.cuda.memory_allocated(gpu_idx) / 1e9
            print(f"  GPU {gpu_idx}: {len(gpu_layers)} layers, {mem:.1f}GB used")

    def infer(self, input_ids: torch.Tensor) -> torch.Tensor:
        """Run inference — data flows through GPU shards automatically."""
        with torch.no_grad():
            outputs = self.model(input_ids.to("cuda:0"))
        return outputs.logits

# Usage: 70B model across 4 GPUs
manager = ModelShardManager("meta-llama/Llama-2-70b-hf", num_gpus=4)
manager.load_sharded()
# GPU 0: layers 0–19   (17.5GB)
# GPU 1: layers 20–39  (17.5GB)
# GPU 2: layers 40–59  (17.5GB)
# GPU 3: layers 60–79  (17.5GB)
```

### Use Case 3: Sharding AI Training Data

```python
class ShardedDatasetPipeline:
    """
    Shard a large fine-tuning dataset across workers for parallel training.
    """

    def __init__(self, dataset_path: str, num_shards: int, worker_id: int):
        self.dataset_path = dataset_path
        self.num_shards = num_shards
        self.worker_id = worker_id  # This worker's shard index

    def get_shard(self):
        """Load only this worker's portion of the dataset."""
        from datasets import load_dataset

        dataset = load_dataset(
            self.dataset_path,
            split=f"train[{self.worker_id * 100 // self.num_shards}%:"
                  f"{(self.worker_id + 1) * 100 // self.num_shards}%]"
        )

        print(f"Worker {self.worker_id}: loaded {len(dataset)} examples "
              f"(shard {self.worker_id + 1}/{self.num_shards})")
        return dataset

    @staticmethod
    def merge_checkpoints(checkpoint_dir: str, num_shards: int):
        """After training, merge per-shard model checkpoints."""
        import glob

        shard_paths = sorted(glob.glob(f"{checkpoint_dir}/shard_*/model.pt"))
        assert len(shard_paths) == num_shards, \
            f"Expected {num_shards} shards, found {len(shard_paths)}"

        merged_state = {}
        for i, path in enumerate(shard_paths):
            shard_state = torch.load(path)
            # Average weights across shards (federated averaging)
            for key, value in shard_state.items():
                if key in merged_state:
                    merged_state[key] += value / num_shards
                else:
                    merged_state[key] = value / num_shards

        torch.save(merged_state, f"{checkpoint_dir}/merged_model.pt")
        print(f"Merged {num_shards} checkpoint shards → merged_model.pt")

# Distributed training: 8 workers, each trains on 1/8 of the data
pipeline = ShardedDatasetPipeline(
    dataset_path="my_org/training_data",
    num_shards=8,
    worker_id=int(os.environ.get("WORKER_ID", 0))
)
shard = pipeline.get_shard()
```

---

## Advanced Patterns

### Pattern: Dynamic Resharding (Online Resharding)

Reshard without downtime by double-writing to old and new shard configurations during migration.

```javascript
class OnlineReshardingManager {
  constructor({ oldRouter, newRouter, redis }) {
    this.old = oldRouter;
    this.new = newRouter;
    this.redis = redis;
    this.migrationKey = 'resharding:migrated_keys';
    this.mode = 'old'; // 'old' | 'dual-write' | 'new'
  }

  async startMigration() {
    this.mode = 'dual-write';
    console.log('Resharding: dual-write mode activated');
  }

  async write(key, data) {
    if (this.mode === 'old') {
      return this.old.write(key, data);
    }

    if (this.mode === 'dual-write') {
      // Write to BOTH old and new shards
      await Promise.all([
        this.old.write(key, data),
        this.new.write(key, data)
      ]);
      await this.redis.sAdd(this.migrationKey, key);
      return;
    }

    return this.new.write(key, data);
  }

  async read(key) {
    if (this.mode === 'new') return this.new.read(key);

    if (this.mode === 'dual-write') {
      // Check if key has been migrated
      const migrated = await this.redis.sIsMember(this.migrationKey, key);
      return migrated ? this.new.read(key) : this.old.read(key);
    }

    return this.old.read(key);
  }

  async backfillOldData(batchSize = 1000) {
    // Migrate existing data from old to new shards in background
    let cursor = '0';
    let migrated = 0;

    do {
      const [nextCursor, keys] = await this.redis.scan(cursor, { COUNT: batchSize });
      cursor = nextCursor;

      await Promise.all(
        keys.map(async key => {
          const data = await this.old.read(key);
          if (data) {
            await this.new.write(key, data);
            await this.redis.sAdd(this.migrationKey, key);
            migrated++;
          }
        })
      );

      console.log(`Backfill progress: ${migrated} keys migrated`);
    } while (cursor !== '0');

    console.log('Backfill complete — switching to new shards');
    this.mode = 'new';
  }
}
```

### Pattern: Cross-Shard Scatter-Gather Query

For queries that must touch all shards (e.g., global search, analytics), scatter requests in parallel and merge results.

```javascript
class ScatterGatherExecutor {
  constructor({ shards, timeout = 5000 }) {
    this.shards = shards;
    this.timeout = timeout;
  }

  async scatter(queryFn, mergeStrategy = 'concat') {
    const startTime = Date.now();

    // Fire queries at all shards simultaneously
    const shardPromises = this.shards.map(async (shard, i) => {
      try {
        const result = await Promise.race([
          queryFn(shard, i),
          new Promise((_, reject) =>
            setTimeout(() => reject(new Error(`Shard ${i} timeout`)), this.timeout)
          )
        ]);
        return { shardIndex: i, result, success: true };
      } catch (error) {
        console.warn(`Shard ${i} failed: ${error.message}`);
        return { shardIndex: i, result: [], success: false, error: error.message };
      }
    });

    const shardResults = await Promise.all(shardPromises);
    const elapsed = Date.now() - startTime;

    const successCount = shardResults.filter(r => r.success).length;
    console.log(`Scatter-gather: ${successCount}/${this.shards.length} shards responded in ${elapsed}ms`);

    return this._merge(shardResults, mergeStrategy);
  }

  _merge(shardResults, strategy) {
    const allResults = shardResults.flatMap(r => r.result || []);

    if (strategy === 'concat') return allResults;

    if (strategy === 'top_k') {
      // Merge and re-rank by similarity score
      return allResults
        .sort((a, b) => b.similarity - a.similarity)
        .slice(0, 10);
    }

    if (strategy === 'sum') {
      return allResults.reduce((acc, r) => acc + r, 0);
    }

    return allResults;
  }
}

// Usage: Global semantic search across all tenant shards (admin use case)
const executor = new ScatterGatherExecutor({ shards: vectorShards });

const globalResults = await executor.scatter(
  async (shard, shardIndex) => {
    const results = await shard.search(queryEmbedding, { limit: 20 });
    return results.map(r => ({ ...r, shardIndex }));
  },
  'top_k' // Merge and return top 10 across all shards
);
```

### Pattern: Shard-Level Circuit Breaker

Isolate failures to individual shards — if shard 3 goes down, other shards continue serving.

```javascript
const CircuitBreaker = require('opossum');

class FaultTolerantShardRouter {
  constructor(shardClients) {
    // Wrap each shard in its own circuit breaker
    this.breakers = shardClients.map((client, i) =>
      new CircuitBreaker(
        async (operation) => operation(client),
        {
          timeout: 3000,
          errorThresholdPercentage: 25,
          resetTimeout: 15000,
          name: `shard-${i}`
        }
      )
    );

    this.breakers.forEach((breaker, i) => {
      breaker.on('open', () =>
        console.error(`⚠️  Shard ${i} circuit OPEN — routing around it`)
      );
      breaker.on('close', () =>
        console.log(`✅ Shard ${i} circuit CLOSED — back online`)
      );
    });
  }

  async execute(shardIndex, operation) {
    try {
      return await this.breakers[shardIndex].fire(operation);
    } catch (error) {
      if (error.code === 'EOPENBREAKER') {
        // Shard is down — try a replica if available
        throw new Error(`Shard ${shardIndex} unavailable. Try again shortly.`);
      }
      throw error;
    }
  }

  getShardHealth() {
    return this.breakers.map((breaker, i) => ({
      shard: i,
      state: breaker.opened ? 'OPEN' : breaker.halfOpen ? 'HALF_OPEN' : 'CLOSED',
      stats: breaker.stats
    }));
  }
}
```

---

## Monitoring & Observability

### Key Metrics to Track

```javascript
class ShardMetricsCollector {
  constructor({ shards, prometheus }) {
    this.shards = shards;
    
    // Prometheus metrics
    this.queryLatency = prometheus.createHistogram({
      name: 'shard_query_duration_seconds',
      help: 'Query duration per shard',
      labelNames: ['shard_index', 'operation']
    });

    this.shardSize = prometheus.createGauge({
      name: 'shard_document_count',
      help: 'Number of documents per shard',
      labelNames: ['shard_index']
    });

    this.hotSpotRatio = prometheus.createGauge({
      name: 'shard_hotspot_ratio',
      help: 'Ratio of queries to the hottest vs average shard',
      labelNames: ['shard_index']
    });
  }

  async collectShardStats() {
    const stats = await Promise.all(
      this.shards.map(async (shard, i) => {
        const start = Date.now();
        const result = await shard.query(
          `SELECT COUNT(*) as doc_count,
                  pg_database_size(current_database()) as size_bytes
           FROM documents`
        );
        const latency = (Date.now() - start) / 1000;

        return {
          shardIndex: i,
          docCount: parseInt(result.rows[0].doc_count),
          sizeBytes: parseInt(result.rows[0].size_bytes),
          healthCheckLatencyMs: latency * 1000
        };
      })
    );

    // Detect hot spots
    const counts = stats.map(s => s.docCount);
    const avg = counts.reduce((a, b) => a + b, 0) / counts.length;
    const max = Math.max(...counts);
    const imbalanceRatio = max / avg;

    if (imbalanceRatio > 1.5) {
      console.warn(`Shard imbalance detected! Hot shard has ${imbalanceRatio.toFixed(2)}x avg load`);
    }

    return { stats, imbalanceRatio };
  }

  async getReport() {
    const { stats, imbalanceRatio } = await this.collectShardStats();
    const totalDocs = stats.reduce((a, s) => a + s.docCount, 0);

    return {
      totalDocuments: totalDocs,
      imbalanceRatio: imbalanceRatio.toFixed(2),
      shards: stats.map(s => ({
        ...s,
        percentageOfTotal: `${((s.docCount / totalDocs) * 100).toFixed(1)}%`,
        sizeMB: (s.sizeBytes / 1024 / 1024).toFixed(1)
      }))
    };
  }
}

// Example report output:
// {
//   "totalDocuments": 200000000,
//   "imbalanceRatio": "1.08",     ← Good! (< 1.5 is healthy)
//   "shards": [
//     { "shardIndex": 0, "docCount": 25100000, "percentageOfTotal": "12.6%", "sizeMB": "148" },
//     { "shardIndex": 1, "docCount": 24800000, "percentageOfTotal": "12.4%", "sizeMB": "145" },
//     ...
//   ]
// }
```

### Alerting Rules

```yaml
# Prometheus alerting rules for sharding
groups:
  - name: sharding
    rules:
      - alert: ShardImbalance
        expr: shard_hotspot_ratio > 2.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Shard {{ $labels.shard_index }} is overloaded"
          description: "Shard carrying {{ $value }}x average load — consider resharding"

      - alert: ShardDown
        expr: up{job="shard"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Shard {{ $labels.shard_index }} is DOWN"
          description: "Shard offline — {{ $value | humanize }}% of data unavailable"

      - alert: ShardQueryLatencyHigh
        expr: histogram_quantile(0.99, shard_query_duration_seconds) > 2.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency on shard {{ $labels.shard_index }}"
          description: "P99 = {{ $value | humanizeDuration }} — possible hot spot"
```

---

## Integration with Other Patterns

### Sharding + Cache-Aside

Cache shard routing decisions to avoid repeated directory lookups.

```javascript
class CachedShardRouter {
  constructor({ tenantManager, redis, cacheTtl = 300 }) {
    this.tenantManager = tenantManager;
    this.redis = redis;
    this.cacheTtl = cacheTtl;
  }

  async getShardForTenant(tenantId) {
    const cacheKey = `shard:tenant:${tenantId}`;

    // Cache-aside: check local cache first
    const cached = await this.redis.get(cacheKey);
    if (cached !== null) return parseInt(cached);

    // Miss: look up in shard directory
    const shardIndex = await this.tenantManager.getShardForTenant(tenantId);

    // Store in cache (shard assignments rarely change)
    await this.redis.setEx(cacheKey, this.cacheTtl, String(shardIndex));

    return shardIndex;
  }
}
```

### Sharding + Bulkhead

Each shard gets its own bulkhead — a shard under high load doesn't borrow capacity from others.

```javascript
const pLimit = require('p-limit');

class BulkheadedShardRouter {
  constructor({ shards, maxConcurrentPerShard = 50 }) {
    this.shards = shards;
    // Each shard has its own concurrency limiter
    this.limiters = shards.map(() => pLimit(maxConcurrentPerShard));
  }

  async query(shardIndex, queryFn) {
    const limiter = this.limiters[shardIndex];

    // If this shard's limiter is full, queue here — don't steal from other shards
    return limiter(async () => {
      return queryFn(this.shards[shardIndex]);
    });
  }

  getUtilization() {
    return this.limiters.map((limiter, i) => ({
      shard: i,
      active: limiter.activeCount,
      pending: limiter.pendingCount,
      utilizationPct: ((limiter.activeCount / limiter.concurrency) * 100).toFixed(0)
    }));
  }
}
```

### Sharding + Circuit Breaker + Retry

Full resilience stack: circuit breaker per shard, retry on transient errors, fallback to replica.

```javascript
class FullyResilientShardRouter {
  constructor({ primaryShards, replicaShards }) {
    this.primaries = new FaultTolerantShardRouter(primaryShards);
    this.replicas = new FaultTolerantShardRouter(replicaShards);
  }

  async read(shardIndex, queryFn, retries = 2) {
    for (let attempt = 1; attempt <= retries; attempt++) {
      try {
        // Try primary shard first
        return await this.primaries.execute(shardIndex, queryFn);
      } catch (error) {
        console.warn(`Primary shard ${shardIndex} failed (attempt ${attempt}): ${error.message}`);

        try {
          // Fall back to replica
          return await this.replicas.execute(shardIndex, queryFn);
        } catch (replicaError) {
          if (attempt === retries) throw replicaError;
          await new Promise(r => setTimeout(r, 500 * attempt)); // Backoff
        }
      }
    }
  }

  async write(shardIndex, queryFn) {
    // Writes always go to primary (no fallback — consistency matters)
    return this.primaries.execute(shardIndex, queryFn);
  }
}
```

---

## Real-World Case Studies

### Case Study 1: AI-Powered Legal Research Platform

**Problem:**
- 800M legal documents, growing by 2M/day
- Single Elasticsearch cluster at 95% capacity
- Full-text + semantic (vector) search in one query
- P99 search latency: 8.4 seconds (unusable)

**Solution:**

```javascript
// 12 shards: 4 for full-text, 4 for vector, 4 for metadata
const legalSearchRouter = new ShardRouter([
  ...fullTextShards,    // Elasticsearch nodes (text search)
  ...vectorShards,      // Qdrant nodes (semantic search)
  ...metadataShards     // PostgreSQL nodes (filters, facets)
]);

// Hybrid search: parallel scatter to all shard types
async function hybridLegalSearch(query, filters) {
  const [textResults, vectorResults] = await Promise.all([
    executor.scatter(shard => shard.textSearch(query, filters)),
    executor.scatter(shard => shard.vectorSearch(queryEmbedding, filters))
  ]);

  // Reciprocal Rank Fusion to merge results
  return reciprocalRankFusion(textResults, vectorResults);
}
```

**Results:**
- ✅ P99 latency: 8.4s → 0.38s (96% reduction)
- ✅ Indexing throughput: 2M docs/day without query impact
- ✅ Shard failure: only 1/12 of search degraded (not full outage)
- ✅ Compliance: EU documents on EU shards (GDPR satisfied)
- ✅ Revenue impact: $3.2M ARR gained from reliability improvement

### Case Study 2: Real-Time AI Personalization Engine

**Problem:**
- 50M users, each with a personalization embedding (interests, behavior)
- Cold write latency on single DB: 340ms
- Single point of failure for recommendation feature
- Black Friday: 5x traffic spike crashed the database

**Solution:**

```javascript
// Consistent hashing: 16 shards by userId
const personalizationStore = new ShardedVectorStore([
  ...Array.from({ length: 16 }, (_, i) => `http://redis-shard-${i}:6379`)
]);

// Each user routes to exactly one shard — no scatter needed
async function updateUserEmbedding(userId, newEmbedding) {
  return personalizationStore.upsert(userId, newEmbedding);
  // Auto-routes: hash(userId) → shard 7 → updates that shard only
}

async function getRecommendations(userId) {
  const userEmbedding = await personalizationStore.get(userId);
  // Single-shard read: < 2ms
  return vectorSearch(userEmbedding, { limit: 20 });
}
```

**Results:**
- ✅ Write latency: 340ms → 18ms (95% reduction)
- ✅ Black Friday: 5x spike absorbed by parallel shards
- ✅ Shard failure: 1/16 users affected vs entire platform
- ✅ Read P99: 140ms → 8ms
- ✅ Incremental scaling: add 2 shards every quarter as users grow

### Case Study 3: Enterprise Multi-Tenant AI Platform

**Problem:**
- 200 enterprise customers, each with 1–100M documents
- Top 3 customers share 60% of the data
- Other customers see slow queries due to large tenant hot spots
- GDPR: EU customer data must not leave EU region

**Solution:**

```javascript
// Directory-based sharding: explicit tenant → shard assignment
const shardManager = new TenantShardManager({ redis, shardPools });

// VIP tenants get dedicated shards
await redis.hSet('shard:directory', 'enterprise_acme', '8');   // Dedicated shard 8
await redis.hSet('shard:directory', 'enterprise_globex', '9'); // Dedicated shard 9
await redis.hSet('shard:directory', 'enterprise_eu_corp', '10'); // EU-region shard

// Smaller tenants share shards 0–7 (auto-assigned by least-loaded)
// New tenants auto-assigned on first request
```

**Results:**
- ✅ VIP tenants: zero performance impact from other customers
- ✅ Small tenants: P99 latency 2.1s → 0.24s (hot spot eliminated)
- ✅ EU customers: data confirmed to stay in EU shards (GDPR audit passed)
- ✅ Tenant migration: moved 3 growing customers to dedicated shards in <2 hours
- ✅ Customer churn: reduced 40% after reliability improvement

---

## Production Best Practices

### 1. Choosing the Right Number of Shards

```javascript
class ShardCalculator {
  static recommend({ totalDataGB, expectedGrowthPctPerYear, targetQueryMs,
                     singleNodeMaxGB = 500, safetyFactor = 2 }) {
    // Current need
    const currentShards = Math.ceil(totalDataGB / singleNodeMaxGB);

    // Growth-adjusted (plan for 2 years)
    const projectedDataGB = totalDataGB * Math.pow(1 + expectedGrowthPctPerYear / 100, 2);
    const growthShards = Math.ceil(projectedDataGB / singleNodeMaxGB);

    // Round up to next power of 2 (makes resharding easier)
    const recommended = Math.pow(2, Math.ceil(Math.log2(growthShards * safetyFactor)));

    return {
      currentNeed: currentShards,
      twoYearProjection: growthShards,
      recommended,
      rationale: `Power-of-2 shards (${recommended}) simplify future resharding`
    };
  }
}

// Example
const plan = ShardCalculator.recommend({
  totalDataGB: 800,           // Current dataset
  expectedGrowthPctPerYear: 120, // 2.2x growth/year
  targetQueryMs: 100,
  singleNodeMaxGB: 300        // Conservative per-node limit
});
// { currentNeed: 3, twoYearProjection: 14, recommended: 16, ... }
```

### 2. Avoiding Hot Spots

```javascript
// ❌ BAD: Shard by timestamp → all writes go to "latest" shard
shard = hash(document.createdAt) % numShards;

// ✅ GOOD: Shard by content hash → writes spread evenly
shard = hash(document.id) % numShards;

// ❌ BAD: Shard by first letter → "The..." dominates one shard
shard = document.title[0].charCodeAt(0) % numShards;

// ✅ GOOD: Shard by tenant ID (balances multi-tenant platforms)
shard = hash(document.tenantId) % numShards;

// ✅ GOOD FOR GROWTH: Consistent hashing → minimal remapping on scale
shard = consistentHashRing.getNode(document.id);
```

### 3. Testing Sharded Systems

```javascript
describe('ShardRouter', () => {
  it('should route the same key to the same shard consistently', () => {
    const router = new ShardRouter(testShardConfigs);
    const key = 'tenant_abc';

    const shard1 = router.getShardIndex(key);
    const shard2 = router.getShardIndex(key);
    const shard3 = router.getShardIndex(key);

    expect(shard1).toBe(shard2);
    expect(shard2).toBe(shard3);
  });

  it('should distribute keys roughly evenly across shards', () => {
    const router = new ShardRouter(testShardConfigs);
    const counts = new Array(router.numShards).fill(0);

    for (let i = 0; i < 10000; i++) {
      const shardIndex = router.getShardIndex(`key_${i}`);
      counts[shardIndex]++;
    }

    // No shard should have more than 1.3x the average
    const avg = 10000 / router.numShards;
    counts.forEach((count, i) => {
      expect(count).toBeLessThan(avg * 1.3);
      expect(count).toBeGreaterThan(avg * 0.7);
    });
  });

  it('should handle shard failure gracefully', async () => {
    const router = new FaultTolerantShardRouter(shardClients);

    // Simulate shard 2 going down
    shardClients[2].query = () => Promise.reject(new Error('Connection refused'));

    // Queries to shard 2 should fail fast (not hang)
    const start = Date.now();
    await expect(router.execute(2, q => q.query('SELECT 1'))).rejects.toThrow();
    expect(Date.now() - start).toBeLessThan(5000); // Should fail fast
  });
});
```

### 4. Rebalancing Strategy

```
When to Rebalance:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Imbalance ratio > 2.0 (hottest shard has 2x average load)
✅ Single shard exceeds 80% of target max size
✅ P99 query latency on one shard > 3x others
✅ Adding a new major tenant who will dominate a shard

How to Rebalance Safely:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Enable dual-write mode (writes go to old + new shards)
2. Backfill old data to new shards in background
3. Validate new shards have identical data
4. Switch reads to new shards (canary: 5% → 25% → 100%)
5. Disable old shard writes
6. Decommission old shards after 48h monitoring

Never:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
❌ Do a hard cutover without dual-write period
❌ Rebalance during peak traffic hours
❌ Assume resharding is instant (plan for hours or days)
```

---

## Conclusion

### Key Takeaways

1. **Sharding Enables Unlimited Scale**: No single node can grow forever; sharding removes that ceiling
2. **Essential for Large Vector Datasets**: 100M+ embeddings demand distributed storage and search
3. **Tenant Isolation Solves the Noisy Neighbor Problem**: Dedicated shards for large customers protect everyone else
4. **Choose the Right Strategy**: Hash for even distribution, Range for ordered data, Directory for control, Consistent Hashing for elastic scaling
5. **Plan for Resharding**: Design with resharding in mind from day one — it will happen
6. **Monitor Imbalance Actively**: A balanced shard topology requires ongoing attention
7. **Combine with Other Patterns**: Sharding works best with caching, bulkheads, and circuit breakers

### When to Use Sharding

✅ **Use Sharding For:**
- Vector databases with 10M+ embeddings
- Multi-tenant AI platforms requiring data isolation
- Training datasets too large for one machine
- Model weight distribution across GPUs (tensor/pipeline parallelism)
- AI systems that must meet regional data residency requirements
- Any dataset growing beyond a single node's comfortable capacity

❌ **Don't Use Sharding For:**
- Datasets under 10GB (single node is simpler)
- Applications with frequent cross-shard queries (use a different data model)
- Systems where strict ACID transactions span multiple records
- Early-stage products (add sharding when scale demands it)
- Teams without operational maturity to manage distributed systems

### ROI Summary

**Investment:**
- Architecture design: 1 week = $5K
- Implementation: 3–5 weeks = $25K
- Migration & testing: 2 weeks = $10K
- Monitoring & runbooks: 1 week = $5K
- **Total: ~$45K**

**Returns (Annual, enterprise platform):**
- Prevented major outages (2/year × $500K): $1M
- Query latency improvement (conversion gains): $400K
- Engineering time saved (no more emergency firefighting): $200K
- Enterprise customer retention (SLA compliance): $800K
- **Total: $2.4M/year**

**ROI: 5,233% first year**

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Maintained by:** [Your Organization]
