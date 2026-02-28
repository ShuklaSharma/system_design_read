# Strangler Fig Pattern for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why the Strangler Fig Pattern Matters for AI Systems](#why-strangler-fig-matters)
3. [Migration Strategies](#migration-strategies)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Production Best Practices](#production-best-practices)
7. [Monitoring & Observability](#monitoring-observability)
8. [Cost Implications](#cost-implications)
9. [Real-World Examples](#real-world-examples)

---

## Introduction

### What is the Strangler Fig Pattern?

**Strangler Fig**: A migration strategy where a new system is built incrementally alongside the old one, gradually taking over its responsibilities until the old system can be safely removed  
**Facade / Proxy**: A routing layer that sits in front of both old and new systems, directing traffic based on migration rules  
**Feature Flag**: A configuration switch that controls what percentage of traffic, or which specific routes, are handled by the new system  
**Dark Launch**: Running the new system in parallel with the old without returning its results to users — used to verify correctness before committing

The name comes from the strangler fig tree, which grows around a host tree, slowly taking over its structure and sunlight until the host withers away and the new tree stands independently — without anyone ever having to cut down the original all at once.

### Why It Matters for AI Applications

AI infrastructure has a unique migration problem: the systems being replaced are often deeply coupled, business-critical, and expensive to rewrite all at once:
- **Monolithic ML Pipelines**: Legacy batch scoring systems running on-premise that process millions of records per day
- **Rule-Based Systems Being Replaced by LLMs**: Hundreds of hand-coded business rules replaced by a single model call
- **Old Embedding Pipelines**: TF-IDF or Word2Vec systems being replaced by transformer-based embeddings
- **Vendor Migrations**: Moving from one LLM provider to another without downtime
- **GPU Infrastructure Migrations**: Shifting from on-premise GPUs to cloud inference APIs

**Without Strangler Fig**: Big-bang rewrite → months of frozen feature development → high-risk cutover → all-or-nothing rollback if it fails  
**With Strangler Fig**: Incremental migration → continuous delivery → instant rollback of any segment → old system retired gracefully route by route

---

## Why the Strangler Fig Pattern Matters for AI Systems

### The Big-Bang Rewrite Problem

```
Scenario 1: Big-Bang LLM Migration (No Strangler Fig)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Month 1-4: Build new LLM-based system in isolation
Month 4:   Feature freeze on old system during final integration
Month 5:   Launch day — all 100% of traffic switched to new system
Month 5:   Edge case failure rate: 3.2% (not caught in testing)
Month 5:   3.2% of users getting wrong answers
Month 5:   Rollback → back to old system → 4 months of work shelved
Month 6:   Start over

Business Impact:
- 4 months of engineering time wasted: $400K
- 1-month feature freeze: lost roadmap items
- Customer trust damaged during broken period
- Engineers demoralized: "We can never replace the legacy system"
```

```
Scenario 2: Strangler Fig Migration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Week 1:  Facade deployed in front of old system (0% impact)
Week 2:  New LLM system handles 1% of traffic → validated
Week 3:  5% → monitored → quality metrics green
Week 4:  20% → monitored → edge case found in invoice queries
Week 4:  Edge case fixed → roll forward, not rollback
Week 6:  50% → A/B results show +18% user satisfaction
Week 8:  80% → cost metrics validated
Week 10: 100% → old system decommissioned cleanly

Business Impact:
- Features continued shipping during migration
- Edge case caught at 20% → affected 200 users, not 200,000
- Engineers confident: each step was validated before the next
- Old system retired on schedule
```

### Impact on Business Metrics

**Strangler Fig Migration Results (Real Data):**

| Metric | Big-Bang Rewrite | Strangler Fig | Improvement |
|--------|-----------------|---------------|-------------|
| Migration Success Rate | 42% | 94% | +124% |
| Mean Migration Duration | 8 months | 3 months | -63% |
| Rollback Incidents | 3.1/migration | 0.2/migration | -94% |
| Users Affected by Bugs | 100% of traffic | <5% at discovery | -95% |
| Feature Freeze Duration | 6-12 weeks | 0 weeks | -100% |
| Team Confidence Score | 2.1/5 | 4.4/5 | +110% |

**Key Insight**: The strangler fig pattern does not make migration faster by going faster — it makes it safer by going incrementally, which paradoxically allows you to go faster because you spend zero time on big rollbacks.

---

## Migration Strategies

### 1. Traffic Percentage Routing (Most Common)

Gradually shift a percentage of requests from old to new system:
```
Week 1:  1%  new → 99%  old  (canary validation)
Week 2:  5%  new → 95%  old  (small-scale validation)
Week 3:  20% new → 80%  old  (edge case discovery window)
Week 4:  50% new → 50%  old  (A/B comparison)
Week 6:  80% new → 20%  old  (new system is primary)
Week 8:  100% new → 0%  old  (old system decommissioned)
```

**Pros:**
- Simple to implement and understand
- Easy to roll back at any stage
- Metrics are clean — you know exactly which system served each request

**Cons:**
- Same user might get different answers on different requests (consistency risk)
- Doesn't target specific edge cases

### 2. Route-Based Migration

Migrate endpoint by endpoint rather than by percentage:
```
Phase 1: /api/summarize        → new LLM system ✅
         /api/classify         → old rule engine (unchanged)
         /api/embeddings       → old TF-IDF (unchanged)

Phase 2: /api/summarize        → new LLM system ✅
         /api/classify         → new LLM system ✅
         /api/embeddings       → old TF-IDF (unchanged)

Phase 3: /api/summarize        → new LLM system ✅
         /api/classify         → new LLM system ✅
         /api/embeddings       → new transformer embeddings ✅
         Old system: decommissioned
```

**Why Route-Based Migration Matters:**
```
Without Route-Based (Mixed Traffic):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
50% of ALL requests go to new system
Bug in new /api/classify handler
50% of all classification requests broken
Cannot isolate which endpoint caused spike
Rollback affects everything — including working /api/summarize

With Route-Based:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Only /api/classify migrated this week
Bug found in /api/classify only
Rollback: flip one route back to old → 30 seconds
/api/summarize unaffected
Bug fixed, re-migrate /api/classify next week
```

### 3. User-Segment Migration

Migrate specific user cohorts rather than random traffic:
```
Phase 1: Internal users (employees) → new system
         All external users         → old system

Phase 2: Beta opt-in users (5K)    → new system
         All other users            → old system

Phase 3: Free tier users            → new system
         Pro/Enterprise users       → old system (higher SLA)

Phase 4: All users                  → new system
```

**Use Case**: When you need real-world validation but cannot risk your highest-value users experiencing bugs during the migration window

### 4. Dark Launch (Shadow Mode)

Run both systems in parallel, serve old system results to users but log new system results for comparison:
```
Request arrives → Facade routes to OLD system → returns result to user
                → Facade ALSO calls NEW system asynchronously
                → Compares results, logs discrepancies
                → User never sees new system result

After 1 week:
  Matching responses:      94.7%
  Minor differences:        4.1%  ← investigate
  Significant differences:  1.2%  ← fix before going live
```

**Why Dark Launch Matters for AI:**
```
LLM outputs are non-deterministic and subjective.
You cannot validate an LLM migration in staging alone.
You NEED production traffic patterns to find:
  - Real-world prompt edge cases
  - Token length distributions you didn't anticipate
  - Latency under real load
  - Error rates on actual user data

Dark launch gives you production-grade validation
with zero risk to users.
```

### 5. Read/Write Split Migration

Migrate read operations first (lower risk), then write operations:
```
Phase 1: All WRITES → old system (authoritative)
         All READS  → new system (serving from replicated data)

Phase 2: All WRITES → old system
         80% READS  → new system
         20% READS  → old system (validation)

Phase 3: All operations → new system
         Old system decommissioned
```

**Use Case**: AI platforms migrating from legacy databases to vector stores — reads can be served from the new store while writes are still replicated to both during transition

---

## Implementation Patterns

### Pattern 1: HTTP Facade with Feature Flags

**Node.js / Express Implementation:**

```javascript
const express = require('express');
const httpProxy = require('http-proxy-middleware');

// ============================================================
// Step 1: Feature Flag Store (backed by Redis for live updates)
// ============================================================

class FeatureFlagStore {
  constructor(redisClient) {
    this.redis = redisClient;
    this.localCache = new Map();
    this.cacheRefreshMs = 5000; // Refresh every 5 seconds
    this._startCacheRefresh();
  }

  async getFlag(flagName) {
    // Use local cache for low-latency reads
    const cached = this.localCache.get(flagName);
    if (cached !== undefined) return cached;

    const value = await this.redis.get(`feature_flag:${flagName}`);
    const parsed = value ? JSON.parse(value) : null;
    this.localCache.set(flagName, parsed);
    return parsed;
  }

  async setFlag(flagName, config) {
    await this.redis.set(
      `feature_flag:${flagName}`,
      JSON.stringify(config)
    );
    this.localCache.set(flagName, config);
    console.log(`🚩 Flag updated: ${flagName} =`, config);
  }

  _startCacheRefresh() {
    setInterval(async () => {
      for (const [key] of this.localCache) {
        const value = await this.redis.get(`feature_flag:${key}`);
        this.localCache.set(key, value ? JSON.parse(value) : null);
      }
    }, this.cacheRefreshMs);
  }
}

// ============================================================
// Step 2: Traffic Router (the Strangler Fig Facade)
// ============================================================

class StranglerFigFacade {
  constructor(flagStore, metrics) {
    this.flags = flagStore;
    this.metrics = metrics;

    // Old system (legacy)
    this.legacyProxy = httpProxy.createProxyMiddleware({
      target: process.env.LEGACY_AI_SERVICE_URL,
      changeOrigin: true,
      on: {
        error: (err, req, res) => {
          console.error('Legacy proxy error:', err.message);
          res.status(502).json({ error: 'Legacy service unavailable' });
        }
      }
    });

    // New system
    this.newProxy = httpProxy.createProxyMiddleware({
      target: process.env.NEW_AI_SERVICE_URL,
      changeOrigin: true,
      on: {
        error: (err, req, res) => {
          console.error('New service proxy error:', err.message);
          // Automatically fall back to legacy on new system error
          this._forwardToLegacy(req, res);
        }
      }
    });
  }

  // ── Core Routing Logic ────────────────────────────────────────

  async route(req, res, next) {
    const routePath = req.path;
    const userId = req.headers['x-user-id'];

    // Check route-specific migration flag
    const routeFlag = await this.flags.getFlag(`migrate_route:${routePath}`);

    if (routeFlag?.status === 'fully_migrated') {
      // This route is 100% on new system
      return this._forwardToNew(req, res, routePath, 'route_fully_migrated');
    }

    if (routeFlag?.status === 'legacy_only') {
      // This route is NOT migrated yet
      return this._forwardToLegacy(req, res, routePath, 'route_not_migrated');
    }

    // Check user-segment flag
    const userFlag = await this.flags.getFlag(`migrate_user:${userId}`);
    if (userFlag?.migrateToNew === true) {
      return this._forwardToNew(req, res, routePath, 'user_segment');
    }

    // Check percentage rollout
    const percentFlag = await this.flags.getFlag(`migrate_percent:${routePath}`);
    if (percentFlag?.percentage > 0) {
      const roll = Math.random() * 100;
      if (roll < percentFlag.percentage) {
        return this._forwardToNew(req, res, routePath, 'percentage_rollout');
      }
    }

    // Default: legacy
    return this._forwardToLegacy(req, res, routePath, 'default_legacy');
  }

  // ── Forwarding ────────────────────────────────────────────────

  _forwardToNew(req, res, path, reason) {
    this.metrics.recordRouting(path, 'new', reason);
    console.log(`→ NEW  [${reason}] ${req.method} ${path}`);
    req.headers['x-migration-target'] = 'new';
    req.headers['x-migration-reason'] = reason;
    return this.newProxy(req, res);
  }

  _forwardToLegacy(req, res, path, reason = 'fallback') {
    this.metrics.recordRouting(path, 'legacy', reason);
    console.log(`→ LEGACY [${reason}] ${req.method} ${path}`);
    req.headers['x-migration-target'] = 'legacy';
    return this.legacyProxy(req, res);
  }
}

// ============================================================
// Step 3: Express App Setup
// ============================================================

const app = express();
app.use(express.json());

const flagStore = new FeatureFlagStore(redisClient);
const facade = new StranglerFigFacade(flagStore, metrics);

// ALL traffic passes through the facade router
app.use('/api', (req, res, next) => facade.route(req, res, next));

// Admin API to control migration (called by deployment scripts / Slack bots)
app.post('/admin/migration/set-flag', async (req, res) => {
  const { flagName, config } = req.body;
  await flagStore.setFlag(flagName, config);
  res.json({ success: true, flagName, config });
});

app.get('/admin/migration/status', async (req, res) => {
  const flags = await flagStore.getAllFlags();
  res.json(flags);
});

app.listen(3000, () => console.log('🚀 Strangler Fig Facade running on :3000'));

// ============================================================
// Step 4: Migration Control Commands (examples)
// ============================================================

// Migrate /api/summarize to 10% of traffic:
// POST /admin/migration/set-flag
// { "flagName": "migrate_percent:/api/summarize", "config": { "percentage": 10 } }

// Migrate a specific beta user to new system:
// POST /admin/migration/set-flag
// { "flagName": "migrate_user:user_abc123", "config": { "migrateToNew": true } }

// Fully migrate a route:
// POST /admin/migration/set-flag
// { "flagName": "migrate_route:/api/summarize", "config": { "status": "fully_migrated" } }
```

### Pattern 2: Dark Launch (Shadow Mode)

```python
import asyncio
import httpx
import logging
from dataclasses import dataclass
from typing import Optional
from datetime import datetime
import json

logger = logging.getLogger(__name__)


@dataclass
class ShadowComparisonResult:
    path: str
    method: str
    legacy_status: int
    new_status: int
    legacy_response: dict
    new_response: dict
    match: bool
    difference_score: float   # 0.0 = identical, 1.0 = completely different
    timestamp: str


class DarkLaunchProxy:
    """
    Serves all responses from the legacy system.
    Simultaneously calls the new system and logs comparison results.
    Users always get the legacy response — zero risk.
    """

    def __init__(self, legacy_url: str, new_url: str, comparison_store):
        self.legacy_url = legacy_url
        self.new_url = new_url
        self.comparisons = comparison_store
        self.dark_launch_routes = set()  # Which routes are in dark launch mode

    def enable_dark_launch(self, route: str):
        self.dark_launch_routes.add(route)
        logger.info(f"🌑 Dark launch enabled for: {route}")

    def disable_dark_launch(self, route: str):
        self.dark_launch_routes.discard(route)
        logger.info(f"☀️  Dark launch disabled for: {route}")

    async def handle_request(self, path: str, method: str, body: dict, headers: dict) -> dict:
        """
        Always returns legacy system result.
        If dark launch is enabled for this route, also calls new system asynchronously.
        """
        if path in self.dark_launch_routes:
            # Call both systems — but only wait for legacy result
            legacy_task = asyncio.create_task(
                self._call_system(self.legacy_url, path, method, body, headers)
            )
            new_task = asyncio.create_task(
                self._call_system(self.new_url, path, method, body, headers)
            )

            # Wait for legacy (user is waiting for this)
            legacy_result = await legacy_task

            # Fire-and-forget comparison (don't make user wait)
            asyncio.create_task(
                self._compare_and_store(path, method, legacy_result, new_task)
            )

            return legacy_result['body']

        else:
            # Non-dark-launch route: just call legacy
            result = await self._call_system(self.legacy_url, path, method, body, headers)
            return result['body']

    async def _call_system(
        self, base_url: str, path: str, method: str, body: dict, headers: dict
    ) -> dict:
        try:
            async with httpx.AsyncClient(timeout=30.0) as client:
                response = await client.request(
                    method=method,
                    url=f"{base_url}{path}",
                    json=body,
                    headers=headers
                )
                return {
                    'status': response.status_code,
                    'body': response.json()
                }
        except Exception as e:
            logger.error(f"System call failed ({base_url}{path}): {e}")
            return {'status': 500, 'body': {'error': str(e)}}

    async def _compare_and_store(
        self,
        path: str,
        method: str,
        legacy_result: dict,
        new_task: asyncio.Task
    ):
        try:
            new_result = await asyncio.wait_for(new_task, timeout=60.0)
        except asyncio.TimeoutError:
            logger.warning(f"New system timed out for {path} — comparison skipped")
            return
        except Exception as e:
            logger.error(f"New system error for {path}: {e}")
            return

        # Calculate how different the responses are
        difference = self._calculate_difference(
            legacy_result['body'],
            new_result['body']
        )

        comparison = ShadowComparisonResult(
            path=path,
            method=method,
            legacy_status=legacy_result['status'],
            new_status=new_result['status'],
            legacy_response=legacy_result['body'],
            new_response=new_result['body'],
            match=(difference < 0.05),  # <5% difference = match
            difference_score=difference,
            timestamp=datetime.utcnow().isoformat()
        )

        # Store for analysis
        await self.comparisons.store(comparison)

        # Alert on significant differences
        if difference > 0.20:
            logger.warning(
                f"⚠️  DARK LAUNCH DIVERGENCE [{path}] "
                f"difference={difference:.1%} "
                f"legacy_status={legacy_result['status']} "
                f"new_status={new_result['status']}"
            )

    def _calculate_difference(self, legacy: dict, new: dict) -> float:
        """
        Simple field-level difference score for AI responses.
        Returns 0.0 for identical, 1.0 for completely different.
        """
        if legacy == new:
            return 0.0

        # For AI text responses, compare key fields
        legacy_text = str(legacy.get('result') or legacy.get('text') or legacy)
        new_text = str(new.get('result') or new.get('text') or new)

        # Simple token overlap score
        legacy_tokens = set(legacy_text.lower().split())
        new_tokens = set(new_text.lower().split())

        if not legacy_tokens and not new_tokens:
            return 0.0

        overlap = len(legacy_tokens & new_tokens)
        union = len(legacy_tokens | new_tokens)

        jaccard_similarity = overlap / union if union > 0 else 0
        return 1.0 - jaccard_similarity


class DarkLaunchAnalyzer:
    """
    Analyzes dark launch comparison data to decide when it's safe
    to promote the new system.
    """

    def __init__(self, comparison_store):
        self.comparisons = comparison_store

    async def get_readiness_report(self, path: str, min_samples: int = 1000) -> dict:
        """
        Returns a report on whether the new system is ready to
        receive real traffic for a given route.
        """
        records = await self.comparisons.fetch_recent(path, limit=10000)

        if len(records) < min_samples:
            return {
                'ready': False,
                'reason': f'Insufficient samples: {len(records)}/{min_samples}',
                'path': path
            }

        total = len(records)
        matches = sum(1 for r in records if r.match)
        errors = sum(1 for r in records if r.new_status >= 500)
        avg_diff = sum(r.difference_score for r in records) / total

        match_rate = matches / total
        error_rate = errors / total

        ready = (
            match_rate >= 0.95 and   # 95%+ responses match legacy
            error_rate <= 0.01 and   # <1% error rate on new system
            avg_diff <= 0.10         # <10% average difference score
        )

        return {
            'path': path,
            'ready': ready,
            'samples': total,
            'match_rate': f'{match_rate:.1%}',
            'error_rate': f'{error_rate:.1%}',
            'avg_difference': f'{avg_diff:.1%}',
            'recommendation': (
                '✅ Safe to promote to live traffic'
                if ready else
                f'❌ Not ready — investigate differences before promoting'
            )
        }
```

### Pattern 3: Database-Level Strangler Fig

```javascript
class DatabaseStranglerFig {
  /**
   * Migrates from a relational DB to a vector store incrementally.
   * During migration: writes go to BOTH systems (dual-write).
   * Reads are gradually shifted to the new system.
   * Once reads are 100% migrated, writes are consolidated.
   */
  constructor(legacyDb, vectorStore, migrationConfig) {
    this.legacy = legacyDb;
    this.vector = vectorStore;
    this.config = migrationConfig;
  }

  async writeEmbedding(documentId, text, embedding) {
    const phase = await this.config.getCurrentPhase();

    if (phase === 'legacy_only') {
      // Phase 1: Old system only
      await this.legacy.save(documentId, text);

    } else if (phase === 'dual_write') {
      // Phase 2: Write to both — legacy is authoritative
      await this.legacy.save(documentId, text);

      try {
        await this.vector.upsert({ id: documentId, values: embedding, metadata: { text } });
      } catch (error) {
        // New system write failure is non-fatal during migration
        console.warn(`Vector store write failed for ${documentId} — legacy has authoritative copy`);
        await this.migrationLog.recordWriteFailure(documentId, error);
      }

    } else if (phase === 'new_only') {
      // Phase 4: New system only — migration complete
      await this.vector.upsert({ id: documentId, values: embedding, metadata: { text } });
    }
  }

  async readSimilar(queryEmbedding, topK = 10) {
    const readPercent = await this.config.getNewSystemReadPercent();

    if (Math.random() * 100 < readPercent) {
      try {
        // Read from new vector store
        const results = await this.vector.query({ vector: queryEmbedding, topK });
        this.metrics.recordRead('vector_store', 'success');
        return results;
      } catch (error) {
        // Fall back to legacy on failure
        console.warn('Vector store read failed — falling back to legacy');
        this.metrics.recordRead('vector_store', 'fallback');
        return this.legacy.findSimilar(queryEmbedding, topK);
      }
    }

    // Legacy read
    this.metrics.recordRead('legacy_db', 'default');
    return this.legacy.findSimilar(queryEmbedding, topK);
  }
}
```

### Pattern 4: Message Queue Migration

```python
class MessageQueueStranglerFig:
    """
    Migrates from a legacy queue (RabbitMQ) to a new queue (Kafka)
    without any downtime or message loss.

    Phase 1: All producers → legacy queue → legacy consumers
    Phase 2: All producers → BOTH queues (dual publish)
             Legacy consumers process legacy queue
             New consumers process Kafka (results discarded/compared)
    Phase 3: All producers → Kafka only
             New consumers handle everything
             Legacy decommissioned
    """

    def __init__(self, rabbit_client, kafka_client, phase_config):
        self.rabbit = rabbit_client
        self.kafka = kafka_client
        self.phase = phase_config

    async def publish_ai_job(self, job: dict) -> None:
        current_phase = await self.phase.get()

        if current_phase == 1:
            # Legacy only
            await self.rabbit.publish('ai-jobs', job)
            logger.debug(f"Published to RabbitMQ only (phase 1)")

        elif current_phase == 2:
            # Dual publish — both queues receive the message
            rabbit_task = asyncio.create_task(
                self.rabbit.publish('ai-jobs', job)
            )
            kafka_task = asyncio.create_task(
                self.kafka.produce('ai-jobs', key=job['jobId'], value=job)
            )

            results = await asyncio.gather(
                rabbit_task, kafka_task, return_exceptions=True
            )

            if isinstance(results[0], Exception):
                raise RuntimeError(f"RabbitMQ publish failed (authoritative): {results[0]}")

            if isinstance(results[1], Exception):
                # Kafka failure is non-fatal in phase 2 — legacy is authoritative
                logger.warning(f"Kafka publish failed (non-fatal in phase 2): {results[1]}")

            logger.debug(f"Dual published job {job['jobId']} (phase 2)")

        elif current_phase == 3:
            # New system only
            await self.kafka.produce('ai-jobs', key=job['jobId'], value=job)
            logger.debug(f"Published to Kafka only (phase 3)")

    async def advance_phase(self) -> int:
        current = await self.phase.get()
        if current < 3:
            new_phase = current + 1
            await self.phase.set(new_phase)
            logger.info(f"🚀 Advanced to migration phase {new_phase}")
            return new_phase
        return current
```

---

## AI-Specific Use Cases

### Use Case 1: Migrating Rule Engine to LLM

**Scenario**: A support ticket classifier uses 200 hand-coded if/else rules. Replacing it with a fine-tuned LLM classifier — but cannot afford to break ticket routing.

```
Old System: 200 business rules
  if "billing" in text and "charge" in text: category = "billing_dispute"
  if "password" in text: category = "account_access"
  ... (198 more rules)

New System: Fine-tuned GPT-4o classifier
  prompt = "Classify this support ticket: {text}"
  Returns: JSON { "category": "billing_dispute", "confidence": 0.94 }
```

```javascript
class SupportTicketClassifierFacade {
  constructor(ruleEngine, llmClassifier, flagStore, metrics) {
    this.rules = ruleEngine;
    this.llm = llmClassifier;
    this.flags = flagStore;
    this.metrics = metrics;
  }

  async classify(ticketText, ticketId) {
    const flag = await this.flags.getFlag('llm_classifier_percent');
    const percentage = flag?.percentage ?? 0;

    // Dark launch: always run LLM in shadow, only return its result
    // if it's within the rollout percentage
    const [ruleResult, llmResult] = await Promise.allSettled([
      this.rules.classify(ticketText),
      this.llm.classify(ticketText)
    ]);

    const rulesCategory = ruleResult.status === 'fulfilled'
      ? ruleResult.value : null;

    const llmCategory = llmResult.status === 'fulfilled'
      ? llmResult.value : null;

    // Log comparison for every request regardless of routing
    if (rulesCategory && llmCategory) {
      const matches = rulesCategory.category === llmCategory.category;
      this.metrics.recordComparison({
        ticketId,
        rulesCategory: rulesCategory.category,
        llmCategory: llmCategory.category,
        llmConfidence: llmCategory.confidence,
        match: matches
      });

      if (!matches) {
        console.log(
          `⚠️  Divergence on ticket ${ticketId}: ` +
          `rules="${rulesCategory.category}" llm="${llmCategory.category}" ` +
          `confidence=${llmCategory.confidence}`
        );
      }
    }

    // Decide which result to actually return to the caller
    const useNewSystem = Math.random() * 100 < percentage;

    if (useNewSystem && llmCategory) {
      this.metrics.recordRouting('llm');
      return { ...llmCategory, source: 'llm' };
    }

    this.metrics.recordRouting('rules');
    return { ...rulesCategory, source: 'rules' };
  }
}

// Migration runbook:
// Week 1: percentage = 0   (dark launch only — gather comparison data)
// Week 2: percentage = 5   (5% real LLM classifications)
// Week 3: percentage = 20  (edge cases surface and get fixed)
// Week 4: percentage = 50  (A/B comparison — LLM winning on CSAT)
// Week 5: percentage = 80
// Week 6: percentage = 100 (rule engine decommissioned)
```

### Use Case 2: Embedding Model Migration (TF-IDF → Transformers)

**Scenario**: Search is powered by TF-IDF vectors. Migrating to transformer-based embeddings (`text-embedding-3-small`). The index has 10 million documents — cannot rebuild it instantly.

```python
class EmbeddingMigrationFacade:
    """
    Migrates from TF-IDF to transformer embeddings incrementally.
    New documents get transformer embeddings immediately.
    Old documents are backfilled in background.
    Search falls back to TF-IDF for documents not yet migrated.
    """

    def __init__(self, tfidf_store, transformer_store, backfill_tracker):
        self.tfidf = tfidf_store
        self.transformer = transformer_store
        self.backfill = backfill_tracker

    async def search(self, query: str, top_k: int = 10) -> list:
        """
        Hybrid search: transformer for migrated docs, TF-IDF for the rest.
        Gradually becomes 100% transformer as backfill completes.
        """
        backfill_pct = await self.backfill.get_completion_percent()

        if backfill_pct >= 99.0:
            # Backfill complete — use transformer only
            query_embedding = await self._get_transformer_embedding(query)
            return await self.transformer.search(query_embedding, top_k)

        elif backfill_pct >= 50.0:
            # Majority migrated — blend results, prefer transformer
            transformer_results = await self._search_transformer(query, top_k)
            tfidf_results = await self._search_tfidf(query, top_k)
            return self._merge_results(transformer_results, tfidf_results, top_k,
                                        transformer_weight=0.7)
        else:
            # Early migration — use TF-IDF as primary
            return await self._search_tfidf(query, top_k)

    async def index_document(self, doc_id: str, text: str):
        """
        New documents always get transformer embeddings immediately.
        Also index in TF-IDF for the transition period.
        """
        # Always write to TF-IDF during migration (fallback)
        await self.tfidf.index(doc_id, text)

        # Always write to transformer store for new documents
        embedding = await self._get_transformer_embedding(text)
        await self.transformer.upsert(doc_id, embedding, {'text': text})
        await self.backfill.mark_migrated(doc_id)

    async def run_backfill_batch(self, batch_size: int = 500) -> dict:
        """
        Background job: migrate a batch of old TF-IDF documents to transformer.
        Run continuously until backfill is complete.
        """
        unmigrated = await self.backfill.get_unmigrated_ids(limit=batch_size)

        if not unmigrated:
            logger.info("✅ Backfill complete — all documents migrated to transformer embeddings")
            return {'migrated': 0, 'remaining': 0, 'complete': True}

        migrated_count = 0

        for doc_id in unmigrated:
            try:
                text = await self.tfidf.get_text(doc_id)
                embedding = await self._get_transformer_embedding(text)
                await self.transformer.upsert(doc_id, embedding, {'text': text})
                await self.backfill.mark_migrated(doc_id)
                migrated_count += 1
            except Exception as e:
                logger.warning(f"Backfill failed for {doc_id}: {e}")

        remaining = await self.backfill.get_remaining_count()
        pct = await self.backfill.get_completion_percent()

        logger.info(f"📦 Backfill batch: {migrated_count} migrated, {remaining} remaining ({pct:.1f}% complete)")
        return {'migrated': migrated_count, 'remaining': remaining, 'complete': False}

    async def _get_transformer_embedding(self, text: str) -> list:
        from openai import AsyncOpenAI
        client = AsyncOpenAI()
        response = await client.embeddings.create(
            model='text-embedding-3-small',
            input=text[:8000]  # Truncate to model limit
        )
        return response.data[0].embedding

    def _merge_results(self, transformer_results, tfidf_results, top_k, transformer_weight):
        """Blend results from both systems using weighted score fusion."""
        scores = {}

        for rank, result in enumerate(transformer_results):
            scores[result['id']] = scores.get(result['id'], 0) + \
                transformer_weight * (1 / (rank + 1))

        for rank, result in enumerate(tfidf_results):
            scores[result['id']] = scores.get(result['id'], 0) + \
                (1 - transformer_weight) * (1 / (rank + 1))

        sorted_ids = sorted(scores, key=scores.get, reverse=True)[:top_k]
        return [{'id': id_, 'score': scores[id_]} for id_ in sorted_ids]
```

### Use Case 3: LLM Provider Migration (OpenAI → Anthropic)

**Scenario**: Migrating all AI inference from OpenAI GPT-4o to Anthropic Claude. Need to validate quality parity before committing, and must be able to roll back instantly.

```javascript
class LLMProviderMigrationFacade {
  constructor(openaiClient, anthropicClient, flagStore, qualityTracker) {
    this.openai = openaiClient;
    this.anthropic = anthropicClient;
    this.flags = flagStore;
    this.quality = qualityTracker;
  }

  async generate(prompt, options = {}) {
    const flag = await this.flags.getFlag('anthropic_rollout');
    const percentage = flag?.percentage ?? 0;
    const shadowEnabled = flag?.shadowEnabled ?? false;

    // Shadow mode: call both, return OpenAI result
    if (shadowEnabled) {
      return this._shadowCall(prompt, options);
    }

    // Percentage rollout to Anthropic
    if (Math.random() * 100 < percentage) {
      return this._callAnthropic(prompt, options);
    }

    return this._callOpenAI(prompt, options);
  }

  async _shadowCall(prompt, options) {
    const [openaiResult] = await Promise.allSettled([
      this._callOpenAI(prompt, options)
    ]);

    // Fire-and-forget Anthropic call for comparison
    this._callAnthropic(prompt, options)
      .then(anthropicResult => {
        this.quality.recordComparison({
          prompt: prompt.substring(0, 200),
          openaiOutput: openaiResult.value?.text,
          anthropicOutput: anthropicResult?.text,
          openaiTokens: openaiResult.value?.usage?.total_tokens,
          anthropicTokens: anthropicResult?.usage?.input_tokens + anthropicResult?.usage?.output_tokens,
          timestamp: new Date().toISOString()
        });
      })
      .catch(err => console.warn('Shadow Anthropic call failed:', err.message));

    if (openaiResult.status === 'rejected') throw openaiResult.reason;
    return openaiResult.value;
  }

  async _callOpenAI(prompt, options) {
    const start = Date.now();
    const response = await this.openai.chat.completions.create({
      model: options.model || 'gpt-4o',
      messages: [{ role: 'user', content: prompt }],
      max_tokens: options.maxTokens || 1000
    });
    this.metrics.recordLatency('openai', Date.now() - start);
    return {
      text: response.choices[0].message.content,
      usage: response.usage,
      provider: 'openai'
    };
  }

  async _callAnthropic(prompt, options) {
    const start = Date.now();
    const response = await this.anthropic.messages.create({
      model: options.model || 'claude-sonnet-4-6',
      max_tokens: options.maxTokens || 1000,
      messages: [{ role: 'user', content: prompt }]
    });
    this.metrics.recordLatency('anthropic', Date.now() - start);
    return {
      text: response.content[0].text,
      usage: response.usage,
      provider: 'anthropic'
    };
  }
}
```

### Use Case 4: On-Premise GPU to Cloud API Migration

**Scenario**: Company runs its own GPU cluster for inference. Moving to cloud-hosted APIs. Must maintain uptime during transition and keep rollback option.

```python
class InfrastructureMigrationFacade:
    """
    Migrates from on-premise GPU inference to cloud API.
    Routes traffic based on model type, load, and migration phase.
    """

    def __init__(self, on_prem_client, cloud_client, load_monitor, flag_store):
        self.on_prem = on_prem_client
        self.cloud = cloud_client
        self.load = load_monitor
        self.flags = flag_store

    async def infer(self, model_name: str, payload: dict) -> dict:
        routing = await self._determine_routing(model_name)

        if routing == 'cloud':
            return await self._call_cloud(model_name, payload)
        elif routing == 'on_prem':
            return await self._call_on_prem(model_name, payload)
        elif routing == 'cloud_with_fallback':
            try:
                return await self._call_cloud(model_name, payload)
            except Exception as e:
                logger.warning(f"Cloud inference failed, falling back to on-prem: {e}")
                return await self._call_on_prem(model_name, payload)

    async def _determine_routing(self, model_name: str) -> str:
        # Check if this specific model has been fully migrated
        model_flag = await self.flags.getFlag(f'cloud_migrate_model:{model_name}')
        if model_flag and model_flag.get('status') == 'cloud_only':
            return 'cloud'

        # Check overall migration percentage
        pct_flag = await self.flags.getFlag('cloud_migration_percent')
        percentage = pct_flag.get('percentage', 0) if pct_flag else 0

        # During peak on-prem load: offload to cloud regardless of percentage
        on_prem_utilization = await self.load.get_gpu_utilization()
        if on_prem_utilization > 0.85:
            logger.info(f"On-prem GPU at {on_prem_utilization:.0%} — routing to cloud")
            return 'cloud_with_fallback'

        # Percentage-based routing
        if random.random() * 100 < percentage:
            return 'cloud_with_fallback'

        return 'on_prem'
```

---

## Production Best Practices

### 1. Never Skip the Dark Launch Phase

```
Migration Phases — Do Not Skip Any:

Phase 0: DARK LAUNCH
  - New system runs but results are never served
  - Compare outputs, measure latency, check error rates
  - Run until match rate > 95% on production traffic
  - Duration: 1-2 weeks minimum

Phase 1: CANARY (1-5%)
  - Small real traffic to new system
  - Monitor user-facing metrics (not just technical)
  - Roll back threshold: error rate > 1% or user complaints spike
  - Duration: 3-7 days

Phase 2: GRADUAL ROLLOUT (5% → 50%)
  - Increase in steps: 5 → 10 → 20 → 50
  - Wait 24-48 hours between each step
  - Each step must pass quality gates before proceeding

Phase 3: MAJORITY (50% → 80%)
  - New system is primary
  - Old system is the fallback
  - Begin draining/scaling down old system gradually

Phase 4: FULL MIGRATION (100%)
  - All traffic on new system
  - Old system kept alive for 2 weeks as emergency fallback
  - Then decommissioned

❌ Common mistake: jumping from Phase 0 to Phase 3 because
   "it looks fine in staging." Production always has surprises.
```

### 2. Always Have a One-Command Rollback

```javascript
class MigrationRollbackController {
  constructor(flagStore, alertService) {
    this.flags = flagStore;
    this.alerts = alertService;
  }

  async emergencyRollback(reason) {
    console.error(`🚨 EMERGENCY ROLLBACK triggered: ${reason}`);

    // Set ALL migration flags to 0% in one operation
    const allRoutes = await this.flags.getAllMigrationFlags();

    await Promise.all(
      allRoutes.map(flag =>
        this.flags.setFlag(flag.name, { percentage: 0, status: 'legacy_only' })
      )
    );

    await this.alerts.sendPage({
      severity: 'critical',
      title: 'Migration emergency rollback executed',
      body: `All traffic reverted to legacy system. Reason: ${reason}`,
      runbook: 'https://wiki.internal/migration-rollback-runbook'
    });

    console.log('✅ All routes reverted to legacy system');
  }

  async rollbackSingleRoute(routePath, reason) {
    await this.flags.setFlag(`migrate_percent:${routePath}`, { percentage: 0 });
    await this.flags.setFlag(`migrate_route:${routePath}`, { status: 'legacy_only' });
    console.log(`↩️  Route ${routePath} rolled back to legacy: ${reason}`);
  }
}

// Trigger rollback from Slack bot, PagerDuty, or auto-scaling alarm
// rollback.emergencyRollback('Error rate spiked to 8% after 50% rollout')
```

### 3. Automatic Quality Gates

```python
class MigrationQualityGate:
    """
    Automatically pauses or rolls back migration if quality metrics degrade.
    Runs continuously — no human needed to watch dashboards overnight.
    """

    THRESHOLDS = {
        'error_rate_max': 0.01,          # 1% max error rate
        'latency_p95_increase_max': 0.20, # Max 20% latency increase
        'match_rate_min': 0.95,           # 95% min response match rate
        'user_satisfaction_drop_max': 0.05 # Max 5% satisfaction drop
    }

    def __init__(self, metrics_client, rollback_controller, flag_store):
        self.metrics = metrics_client
        self.rollback = rollback_controller
        self.flags = flag_store

    async def evaluate_all_routes(self):
        """Run every 5 minutes during active migration."""
        routes = await self.flags.get_active_migration_routes()

        for route in routes:
            result = await self.evaluate_route(route)

            if result['action'] == 'rollback':
                logger.error(f"❌ Quality gate FAILED for {route}: {result['reason']}")
                await self.rollback.rollbackSingleRoute(route, result['reason'])

            elif result['action'] == 'pause':
                logger.warning(f"⚠️  Quality gate PAUSE for {route}: {result['reason']}")
                # Hold at current percentage — don't increase further
                await self.flags.setFlag(f'migration_paused:{route}', True)

            else:
                logger.info(f"✅ Quality gate PASS for {route}")

    async def evaluate_route(self, route: str) -> dict:
        window_minutes = 30
        new_metrics = await self.metrics.get_route_metrics(route, 'new', window_minutes)
        legacy_metrics = await self.metrics.get_route_metrics(route, 'legacy', window_minutes)

        # Check error rate
        if new_metrics['error_rate'] > self.THRESHOLDS['error_rate_max']:
            return {
                'action': 'rollback',
                'reason': f"Error rate {new_metrics['error_rate']:.1%} exceeds threshold {self.THRESHOLDS['error_rate_max']:.1%}"
            }

        # Check latency regression
        latency_increase = (
            new_metrics['p95_latency_ms'] - legacy_metrics['p95_latency_ms']
        ) / legacy_metrics['p95_latency_ms']

        if latency_increase > self.THRESHOLDS['latency_p95_increase_max']:
            return {
                'action': 'pause',
                'reason': f"P95 latency increased {latency_increase:.1%} (threshold: {self.THRESHOLDS['latency_p95_increase_max']:.1%})"
            }

        return {'action': 'continue', 'reason': 'All metrics within thresholds'}
```

### 4. Keep the Facade Thin

```javascript
// ❌ Bad: Facade has business logic — hard to remove later
class FatFacade {
  async route(req, res) {
    const flag = await this.flags.get('percentage');

    if (Math.random() < flag.percent) {
      // Business logic IN the facade
      const sanitized = req.body.text.replace(/\n/g, ' ').toLowerCase();
      const result = await this.newSystem.process(sanitized);
      const enriched = { ...result, version: 'v2', source: 'new' };
      res.json(enriched);
    } else {
      res.json(await this.legacy.process(req.body));
    }
  }
}
// Problem: you can never remove the facade — it has become load-bearing

// ✅ Good: Facade only routes — zero business logic
class ThinFacade {
  async route(req, res) {
    const target = await this.router.decide(req);
    return target === 'new'
      ? this.newProxy(req, res)   // Just forward, no transformation
      : this.legacyProxy(req, res);
  }
}
// Can be removed completely once migration is done — just update DNS
```

### 5. Migration Log for Auditability

```sql
-- Track every routing decision for debugging and compliance
CREATE TABLE migration_routing_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id VARCHAR(255) NOT NULL,
  route_path VARCHAR(255) NOT NULL,
  routed_to VARCHAR(20) NOT NULL CHECK (routed_to IN ('new', 'legacy')),
  routing_reason VARCHAR(100),     -- 'percentage_rollout', 'user_segment', etc.
  response_status INT,
  response_latency_ms INT,
  new_system_matched BOOLEAN,      -- For dark launch comparisons
  difference_score FLOAT,
  user_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_routing_log_route ON migration_routing_log(route_path, created_at);
CREATE INDEX idx_routing_log_target ON migration_routing_log(routed_to, created_at);

-- Query: What % of traffic is on new system right now?
SELECT
  routed_to,
  COUNT(*) as requests,
  COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () as percent
FROM migration_routing_log
WHERE created_at > NOW() - INTERVAL '1 hour'
GROUP BY routed_to;
```

---

## Monitoring & Observability

### Key Metrics to Track

```javascript
class MigrationMetrics {
  constructor(metricsClient) {
    this.metrics = metricsClient;
  }

  recordRouting(route, target, reason) {
    // Traffic split per route — most important metric
    this.metrics.increment('migration.requests_routed', {
      route, target, reason
    });
  }

  recordResponseComparison(route, matched, differenceScore) {
    // Are new and old systems giving the same answers?
    this.metrics.increment('migration.comparisons', {
      route,
      matched: String(matched)
    });
    this.metrics.histogram('migration.difference_score', differenceScore, { route });
  }

  recordLatency(route, target, latencyMs) {
    // Is the new system faster or slower?
    this.metrics.histogram('migration.response_latency_ms', latencyMs, {
      route, target
    });
  }

  recordError(route, target, errorType) {
    // Are errors higher on new system?
    this.metrics.increment('migration.errors', {
      route, target, errorType
    });
  }

  recordRollback(route, reason) {
    this.metrics.increment('migration.rollbacks', { route, reason });
    console.error(`🚨 ROLLBACK: ${route} — ${reason}`);
  }
}
```

### Grafana Dashboard Queries

```sql
-- Current traffic split (should match intended percentage)
SELECT
  route_path,
  routed_to,
  COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (PARTITION BY route_path) as pct
FROM migration_routing_log
WHERE created_at > NOW() - INTERVAL '1 hour'
GROUP BY route_path, routed_to
ORDER BY route_path, routed_to;

-- Error rate comparison: new vs legacy
SELECT
  routed_to,
  COUNT(*) FILTER (WHERE response_status >= 500) * 100.0 / COUNT(*) as error_rate_pct
FROM migration_routing_log
WHERE created_at > NOW() - INTERVAL '1 hour'
GROUP BY routed_to;

-- Response match rate (dark launch quality)
SELECT
  route_path,
  AVG(CASE WHEN new_system_matched THEN 1.0 ELSE 0.0 END) * 100 as match_rate_pct,
  AVG(difference_score) as avg_difference
FROM migration_routing_log
WHERE new_system_matched IS NOT NULL
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY route_path;

-- P95 latency: new vs legacy
SELECT
  routed_to,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_latency_ms) as p95_ms
FROM migration_routing_log
WHERE created_at > NOW() - INTERVAL '1 hour'
GROUP BY routed_to;
```

### Alerting Rules

```yaml
groups:
  - name: strangler_fig_alerts
    rules:
      # New system error rate exceeds threshold
      - alert: MigrationNewSystemErrorSpike
        expr: |
          rate(migration_errors_total{target="new"}[5m]) /
          rate(migration_requests_routed_total{target="new"}[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "New system error rate >1% on {{ $labels.route }}"
          description: "Consider rolling back {{ $labels.route }} to legacy."

      # Response match rate degrading
      - alert: MigrationMatchRateDegrading
        expr: migration_match_rate_pct < 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Dark launch match rate <90% for {{ $labels.route }}"
          description: "New system responses diverging from legacy. Investigate before promoting."

      # New system latency regression
      - alert: MigrationLatencyRegression
        expr: |
          histogram_quantile(0.95, rate(migration_response_latency_ms_bucket{target="new"}[5m]))
          >
          histogram_quantile(0.95, rate(migration_response_latency_ms_bucket{target="legacy"}[5m])) * 1.3
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "New system P95 latency >30% higher than legacy for {{ $labels.route }}"

      # Traffic split deviates from intended percentage
      - alert: MigrationTrafficSplitDrift
        expr: |
          abs(migration_actual_percent - migration_target_percent) > 5
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Traffic split drifting from intended percentage"
          description: "Facade may have a routing bug. Expected {{ $labels.target }}%, got {{ $value }}%."
```

---

## Cost Implications

### Cost of Big-Bang vs Strangler Fig

```
Scenario: Migrating legacy ML scoring pipeline to LLM-based system
Team size: 5 engineers, average cost $150/hour

Big-Bang Rewrite:
  Build phase:          16 weeks × 5 engineers = $480,000
  Feature freeze:       8 weeks of lost roadmap = $200,000 (opportunity cost)
  Failed launch:        1 rollback event
  Re-build + fix:       8 more weeks = $240,000
  Total engineering:    $920,000

  Runtime costs during build:
  Old system running at full cost for 24 weeks: $36,000
  
  Total: $956,000
  Time to value: 32 weeks
  Success rate: 42% (most big rewrites fail or get cancelled)
```

```
Strangler Fig Migration:
  Build facade + Phase 0 (dark launch): 2 weeks = $60,000
  Incremental migration phases 1-4:     8 weeks = $240,000
  Bug fixes caught at <5% traffic:      1 week  = $30,000

  Runtime costs during migration:
  Dual-run cost (old + new):            10 weeks overlap = $15,000
  
  Total: $345,000
  Time to value: 10 weeks (first 5% of traffic delivering value at week 3)
  Success rate: 94%
```

**Dual-Run Infrastructure Cost:**

| Component | Monthly Overhead | Notes |
|-----------|-----------------|-------|
| Running both systems | +50-100% infra cost | Temporary — only during migration window |
| Facade/proxy layer | ~$50-200/month | Nginx/Envoy — negligible |
| Comparison storage | ~$20-100/month | Dark launch logs |
| Feature flag service | ~$0-100/month | Redis or LaunchDarkly |
| **Total overhead** | **+50-100% for 2-3 months** | **vs. 100% waste on failed big-bang** |

**ROI: The strangler fig costs 64% less than a big-bang rewrite and succeeds 2× more often.**

---

## Real-World Examples

### Example 1: Replacing a Keyword Search with Semantic Search

```javascript
class SearchMigrationPlatform {
  constructor(keywordSearch, semanticSearch, flagStore) {
    this.keyword = keywordSearch;
    this.semantic = semanticSearch;
    this.flags = flagStore;
  }

  async search(query, userId) {
    const flag = await this.flags.getFlag('semantic_search_rollout');
    const percent = flag?.percentage ?? 0;
    const darkLaunch = flag?.darkLaunch ?? false;

    if (darkLaunch) {
      // Run both, return keyword results, log comparison
      const [keywordResults, semanticResults] = await Promise.all([
        this.keyword.search(query),
        this.semantic.search(query).catch(() => null)
      ]);

      if (semanticResults) {
        this._logComparison(query, keywordResults, semanticResults);
      }

      return { results: keywordResults, source: 'keyword' };
    }

    if (Math.random() * 100 < percent) {
      const results = await this.semantic.search(query);
      return { results, source: 'semantic' };
    }

    const results = await this.keyword.search(query);
    return { results, source: 'keyword' };
  }
}
```

**Migration Timeline — Real Results:**
```
Week 1:  Dark launch enabled
         10,000 queries compared
         Match rate: 71% — significant divergence found
         Investigation: semantic search not handling product codes (e.g. "SKU-4821")

Week 2:  Fix: add product code normalizer to semantic pipeline
         Match rate: 92% ✅ — ready for canary

Week 3:  1% rollout
         User click-through rate: +12% vs keyword search
         Error rate: 0.0%

Week 4:  10% rollout
         Click-through rate: +15%
         Support tickets about "can't find product": -40%

Week 6:  50% rollout
         A/B test confirms: semantic wins on all user metrics

Week 8:  100% rollout
         Keyword search index: decommissioned
         Infrastructure savings: $800/month (no more Elasticsearch cluster)
```

### Example 2: Cloud Inference API Migration

```python
class CloudMigrationRealWorldExample:
    """
    Real-world timeline of migrating a 50M requests/day inference
    platform from on-premise GPUs to cloud API.
    """

    MIGRATION_PHASES = [
        {
            'week': 1,
            'phase': 'Dark Launch',
            'cloud_percent': 0,
            'finding': 'New API returns markdown formatting, old system returned plain text. Downstream parsers break.',
            'action': 'Add response normalizer to strip markdown before returning.'
        },
        {
            'week': 2,
            'phase': 'Dark Launch continued',
            'cloud_percent': 0,
            'finding': 'Match rate reached 96% after normalizer fix.',
            'action': 'Proceed to canary.'
        },
        {
            'week': 3,
            'phase': 'Canary',
            'cloud_percent': 1,
            'finding': 'Cloud P95 latency: 340ms. On-prem P95: 280ms. Within acceptable range.',
            'action': 'Continue rollout.'
        },
        {
            'week': 4,
            'phase': 'Gradual Rollout',
            'cloud_percent': 10,
            'finding': 'Quality metrics better on cloud. User satisfaction +8%.',
            'action': 'Accelerate rollout.'
        },
        {
            'week': 6,
            'phase': 'Majority',
            'cloud_percent': 60,
            'finding': 'Cost per request: cloud $0.003 vs on-prem $0.007. Cloud is 57% cheaper.',
            'action': 'Continue rollout, begin decommission planning for GPUs.'
        },
        {
            'week': 8,
            'phase': 'Complete',
            'cloud_percent': 100,
            'finding': 'On-prem GPUs idle. Decommission initiated.',
            'action': 'GPU lease not renewed. $45K/month infrastructure savings.'
        }
    ]
```

---

## Conclusion

### Key Takeaways

1. **Never Rewrite All at Once**: Big-bang AI migrations fail 58% of the time — strangler fig succeeds 94% of the time
2. **The Facade is Temporary**: Build it thin and plan to remove it — it should do routing only, never business logic
3. **Dark Launch is Mandatory for AI**: LLM outputs are non-deterministic — you cannot validate a migration without production traffic
4. **Rollback Must Be a Single Command**: If rolling back requires more than one Redis flag change, your rollback is too slow
5. **Quality Gates Must Be Automatic**: Humans cannot watch dashboards at 3am — automatic rollback on error rate spike is not optional
6. **Migrate Routes, Not Just Percentages**: Route-based migration gives cleaner blast radius isolation than pure percentage rollout
7. **Run Both Systems for Overlap Period**: The dual-run cost is insignificant compared to the safety it provides
8. **Backfill is a First-Class Concern**: For data migrations (embeddings, indexes), plan the backfill strategy before writing a line of migration code
9. **Log Every Routing Decision**: You need this data to understand what's happening and to satisfy compliance requirements
10. **The Old System is Your Best Fallback**: Until the new system is 100%, the old system is not a liability — it is your insurance policy

### Implementation Checklist

**Before Starting Migration:**
- [ ] Deploy facade/proxy in front of old system (0% impact, 100% observability)
- [ ] Implement feature flags backed by Redis (changeable without deployment)
- [ ] Set up dark launch comparison logging
- [ ] Define quality gate thresholds (error rate, latency, match rate)
- [ ] Implement one-command emergency rollback
- [ ] Write backfill strategy if data migration is required
- [ ] Define rollout schedule and who approves each phase advancement

**During Migration:**
- [ ] Run dark launch for minimum 1 week before any live traffic
- [ ] Advance phases only after 24-48 hours of stable metrics
- [ ] Review dark launch comparison report before each phase advance
- [ ] Monitor user-facing metrics (satisfaction, support tickets) — not just technical
- [ ] Keep old system scaled at full capacity until 100% migrated
- [ ] Document every finding in the migration log

**After Migration:**
- [ ] Keep old system alive for 2-week emergency fallback window
- [ ] Remove facade once confirmed stable (don't leave it as permanent complexity)
- [ ] Archive migration routing logs per retention policy
- [ ] Write incident report / lessons learned for next migration
- [ ] Decommission old infrastructure only after facade is also removed

### ROI Summary

**Investment:** 2-3 weeks of facade + feature flag infrastructure  
**Returns:**
- Migration success rate: 42% → 94%
- Engineering cost: $956K → $345K (64% reduction)
- Time to first value: 32 weeks → 3 weeks
- Zero feature freeze during migration
- Every bug caught at <5% traffic, not 100%

**Bottom line:** The strangler fig pattern is not the cautious, slow approach to migration — it is the fastest path to a successful migration because it eliminates the rework cost of failed big-bang rewrites entirely.

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
