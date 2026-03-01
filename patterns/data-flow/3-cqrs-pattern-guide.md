# CQRS — Command Query Responsibility Segregation for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why CQRS Matters for AI Systems](#why-cqrs-matters)
3. [CQRS Architecture Variants](#cqrs-architecture-variants)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Monitoring & Observability](#monitoring-observability)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Real-World Case Studies](#real-world-case-studies)
10. [Production Best Practices](#production-best-practices)

---

## Introduction

### What is CQRS?

**CQRS (Command Query Responsibility Segregation)** is an architectural pattern that separates the **write side** (Commands — things that change state) from the **read side** (Queries — things that return data). Instead of one model serving both reads and writes, you have two specialized models: one optimized for writes, one optimized for reads.

**Analogy:** Like a restaurant — the kitchen (write side) receives orders and prepares food, optimized for cooking. The dining room menu (read side) is designed for customers to browse and decide, optimized for clarity and display. The kitchen and the menu are different — the kitchen doesn't list every dish as a full recipe, and the menu doesn't include cooking temperatures. Each is optimized for its audience.

### The Problem Without CQRS

```
Scenario: AI analytics platform — 50M events/day

Without CQRS (single shared model):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
One PostgreSQL table: ai_events (50M rows)
  - Columns for writes: raw_payload, model_id, user_id, timestamp
  - Columns for reads: aggregates, computed scores, joined user data

Write operation: INSERT raw event
  → Must update indexes on ALL read-optimized columns → slow
  → Write throughput: 500 events/sec (bottleneck: index updates)

Read operation: "Show AI usage by department for last 30 days"
  → Full table scan + GROUP BY + JOIN users + JOIN models
  → Query time: 42 seconds (unacceptable for dashboard)
  → During query: table locks block writes
  → During write burst: queries time out

Result:
- Engineers forced to choose: fast writes OR fast reads (never both)
- Read queries and write jobs compete for same DB resources
- Dashboard refresh causes write slowdowns
- Write spikes cause dashboard timeouts
- Schema changes for analytics break the write path
- Zero ability to scale reads independently from writes
```

### The Solution With CQRS

```
With CQRS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Write Side (Command Model):
  → PostgreSQL, schema optimized for INSERT speed
  → Minimal indexes, append-only
  → Write throughput: 12,000 events/sec
  → No joins, no aggregation columns

Read Side (Query Model):
  → ClickHouse (columnar DB), pre-aggregated
  → Pre-joined tables for dashboard queries
  → Materialized views for common queries
  → Query time: 0.3 seconds (42s → 0.3s)

Sync: Events published to Kafka → read model updated asynchronously
Lag: Typically 500ms–2 seconds (acceptable for analytics)

Result:
- Writes and reads never compete for resources
- Read models can be rebuilt without touching the write side
- Add a new analytics dimension: just build a new read model
- Scale read replicas independently as dashboard traffic grows
- Schema change on read side: transparent to write side
```

---

## Why CQRS Matters for AI Systems

### Unique AI System Challenges

1. **Extreme Read/Write Asymmetry**
   - AI training: write-heavy (millions of samples written once)
   - AI inference serving: read-heavy (billions of reads, rarely written)
   - AI analytics dashboards: complex aggregation reads, continuous writes
   - One model cannot be optimal for all three

2. **Complex Query Requirements**
   - "Show P95 latency by model, region, and user tier for the last 7 days"
   - "Which prompts produce the highest hallucination rate?"
   - "Show token usage vs cost vs accuracy across all model versions"
   - These queries require pre-aggregation — impossible with a raw event store

3. **Different Scaling Requirements**
   - Model training writes: burst, then quiet
   - Inference logging writes: steady high volume
   - Dashboard reads: spiky (9–5 business hours)
   - Experiment reads: high during research sprints
   - Cannot share resources between these profiles

4. **Read Model Flexibility**
   - Teams need different views of the same data
   - Data scientists want raw events
   - Product managers want aggregated dashboards
   - Engineers want latency histograms
   - CQRS lets you build a dedicated read model per audience

### Real Incident: No CQRS

**AI Observability Platform — Enterprise SaaS**

```
Timeline:
Q1 2024 - Platform growing: 200 enterprise customers
          All AI events written to single PostgreSQL DB
          Dashboard queries run against same DB

March:
09:00 - Monday morning: 150 engineers open dashboards simultaneously
09:02 - Dashboard queries start: GROUP BY + JOIN on 80M rows
09:05 - Dashboard queries acquiring shared locks
09:06 - AI inference events arriving: INSERT blocked by lock contention
09:08 - AI inference logging starts failing (write timeout)
09:10 - Customers see: inference logging disabled, data gap appearing
09:15 - Dashboard queries complete → writes unblock
09:15 - 9 minutes of missing inference data for all 200 customers
09:20 - Support tickets: "Why is there a gap in our AI usage data?"

April: Same issue, 3 times.
May: Engineers add read replicas.
May 15: Read replica 6 hours behind → dashboards show stale data.
May 20: Emergency: scale DB to $8,000/month instance.

Impact:
- Monthly data gaps: 4 incidents × 9 minutes each
- Customer trust issues: data gaps in compliance reports
- DB cost: $800/month → $8,000/month (10x emergency scaling)
- Engineering time: 40 hours/month firefighting DB performance
- Q2 churn: 12 customers cited "unreliable data" as exit reason
- Estimated lost ARR from churn: $480K
```

**Same Situation WITH CQRS:**

```
Architecture: Writes → Kafka → ClickHouse read model
              Dashboards query ClickHouse ONLY

Monday 09:00 - 150 engineers open dashboards
09:02 - 150 dashboard queries hit ClickHouse
        (completely separate from the write DB)
09:02 - AI inference events continue writing to PostgreSQL
        (unaffected by dashboard queries — different system)
09:05 - All dashboards load in 0.4 seconds
09:05 - Inference logging: uninterrupted, 0 gaps

Impact:
- Zero data gaps
- Zero lock contention
- PostgreSQL cost: $800/month (unchanged — write-optimized)
- ClickHouse cost: $600/month (read-optimized, efficient)
- Total: $1,400/month vs $8,000/month emergency cost
- Zero churn from data reliability issues
```

---

## CQRS Architecture Variants

### 1. Simple CQRS (Same Database, Split Models)

Use the same database but separate read/write models in code. Minimal overhead, good starting point.

```
Write Model: User.create(), User.update() → normalized tables
Read Model:  UserView.findForDashboard() → denormalized view/materialized view
```

**Use For:** Small teams, getting started, low-scale systems

### 2. Separate Databases (Read/Write Split)

Write to a primary DB; sync to a separate read-optimized DB.

```
Commands → Primary DB (PostgreSQL)
                ↓ replication
Queries  ← Read DB (ClickHouse, Elasticsearch, Redis)
```

**Use For:** Different query patterns, high read traffic, AI analytics

### 3. Event-Driven CQRS

Commands produce events; events build read models. Enables multiple independent read models.

```
Commands → Event Store → Events → Read Model A (dashboard)
                                → Read Model B (ML features)
                                → Read Model C (billing)
```

**Use For:** Complex domains, multiple stakeholder views, audit requirements

### 4. CQRS + Event Sourcing (Full Pattern)

The event store IS the write model. Read models are projections from events. This is the most powerful — and most complex — variant. (See the Event Sourcing guide for details.)

---

## Implementation Patterns

### Pattern 1: Simple CQRS with Separated Handlers

```javascript
// ─────────────────────────────────────────
// COMMAND SIDE — write operations
// ─────────────────────────────────────────

class CommandBus {
  constructor() {
    this.handlers = new Map();
  }

  register(commandName, handler) {
    this.handlers.set(commandName, handler);
  }

  async dispatch(command) {
    const handler = this.handlers.get(command.type);
    if (!handler) throw new Error(`No handler for command: ${command.type}`);

    // Commands return void (or a minimal ID) — never return full query data
    const result = await handler.handle(command);
    return result;
  }
}

// Command handlers — focused on business logic + persistence
class CreateAIJobCommandHandler {
  constructor({ db, eventBus }) {
    this.db = db;
    this.eventBus = eventBus;
  }

  async handle({ jobId, tenantId, modelId, prompt, userId }) {
    // Validate
    if (!prompt || prompt.length < 1) throw new Error('Prompt is required');

    // Write to normalized write model
    await this.db.query(
      `INSERT INTO ai_jobs (id, tenant_id, model_id, prompt, user_id, status, created_at)
       VALUES ($1, $2, $3, $4, $5, 'PENDING', NOW())`,
      [jobId, tenantId, modelId, prompt, userId]
    );

    // Publish event for read model update
    await this.eventBus.publish('AIJobCreated', {
      jobId, tenantId, modelId, prompt, userId,
      timestamp: new Date().toISOString()
    });

    return { jobId }; // Return only the ID — not the full object
  }
}

class CompleteAIJobCommandHandler {
  constructor({ db, eventBus }) {
    this.db = db;
    this.eventBus = eventBus;
  }

  async handle({ jobId, output, tokensUsed, latencyMs, modelVersion }) {
    await this.db.query(
      `UPDATE ai_jobs
       SET status = 'COMPLETED', output = $2, tokens_used = $3,
           latency_ms = $4, model_version = $5, completed_at = NOW()
       WHERE id = $1`,
      [jobId, output, tokensUsed, latencyMs, modelVersion]
    );

    await this.eventBus.publish('AIJobCompleted', {
      jobId, tokensUsed, latencyMs, modelVersion,
      timestamp: new Date().toISOString()
    });

    return { jobId };
  }
}

// ─────────────────────────────────────────
// QUERY SIDE — read operations
// ─────────────────────────────────────────

class QueryBus {
  constructor() {
    this.handlers = new Map();
  }

  register(queryName, handler) {
    this.handlers.set(queryName, handler);
  }

  async execute(query) {
    const handler = this.handlers.get(query.type);
    if (!handler) throw new Error(`No handler for query: ${query.type}`);
    return handler.handle(query);
  }
}

class AIUsageDashboardQueryHandler {
  constructor({ readDB }) {
    this.readDB = readDB; // Separate ClickHouse / read-optimized DB
  }

  async handle({ tenantId, startDate, endDate, groupBy = 'model' }) {
    // Queries read from the optimized read model — not the write DB
    const result = await this.readDB.query(
      `SELECT
         ${groupBy},
         COUNT(*) AS total_jobs,
         SUM(tokens_used) AS total_tokens,
         AVG(latency_ms) AS avg_latency_ms,
         QUANTILE(0.95)(latency_ms) AS p95_latency_ms,
         SUM(estimated_cost_usd) AS total_cost_usd
       FROM ai_jobs_read_model
       WHERE tenant_id = {tenantId:String}
         AND completed_at BETWEEN {startDate:DateTime} AND {endDate:DateTime}
       GROUP BY ${groupBy}
       ORDER BY total_jobs DESC`,
      { tenantId, startDate, endDate }
    );
    return result.rows;
  }
}

class RecentJobsQueryHandler {
  constructor({ readDB }) {
    this.readDB = readDB;
  }

  async handle({ tenantId, userId, limit = 20 }) {
    const result = await this.readDB.query(
      `SELECT job_id, model_name, prompt_preview, status,
              tokens_used, latency_ms, created_at
       FROM ai_jobs_user_view
       WHERE tenant_id = {tenantId:String}
         AND user_id = {userId:String}
       ORDER BY created_at DESC
       LIMIT {limit:UInt32}`,
      { tenantId, userId, limit }
    );
    return result.rows;
  }
}

// Wire it up
const commandBus = new CommandBus();
commandBus.register('CreateAIJob', new CreateAIJobCommandHandler({ db, eventBus }));
commandBus.register('CompleteAIJob', new CompleteAIJobCommandHandler({ db, eventBus }));

const queryBus = new QueryBus();
queryBus.register('AIUsageDashboard', new AIUsageDashboardQueryHandler({ readDB }));
queryBus.register('RecentJobs', new RecentJobsQueryHandler({ readDB }));
```

### Pattern 2: Read Model Projector (Event → Read Model)

```javascript
class AIJobReadModelProjector {
  constructor({ clickhouse }) {
    this.clickhouse = clickhouse;
  }

  // Listen for domain events and update read models
  async onAIJobCreated(event) {
    await this.clickhouse.insert('ai_jobs_read_model', [{
      job_id: event.jobId,
      tenant_id: event.tenantId,
      model_id: event.modelId,
      model_name: await this._resolveModelName(event.modelId),
      user_id: event.userId,
      prompt_preview: event.prompt.substring(0, 100),
      status: 'PENDING',
      tokens_used: 0,
      latency_ms: 0,
      estimated_cost_usd: 0,
      created_at: event.timestamp,
      completed_at: null
    }]);
  }

  async onAIJobCompleted(event) {
    const costPerToken = 0.00003; // GPT-4 pricing
    await this.clickhouse.query(
      `ALTER TABLE ai_jobs_read_model UPDATE
         status = 'COMPLETED',
         tokens_used = {tokensUsed:UInt32},
         latency_ms = {latencyMs:UInt32},
         estimated_cost_usd = {cost:Float64},
         model_version = {modelVersion:String},
         completed_at = {timestamp:DateTime}
       WHERE job_id = {jobId:String}`,
      {
        jobId: event.jobId,
        tokensUsed: event.tokensUsed,
        latencyMs: event.latencyMs,
        cost: event.tokensUsed * costPerToken,
        modelVersion: event.modelVersion,
        timestamp: event.timestamp
      }
    );
  }

  async onAIJobFailed(event) {
    await this.clickhouse.query(
      `ALTER TABLE ai_jobs_read_model UPDATE
         status = 'FAILED',
         error_message = {errorMsg:String},
         failed_at = {timestamp:DateTime}
       WHERE job_id = {jobId:String}`,
      { jobId: event.jobId, errorMsg: event.error, timestamp: event.timestamp }
    );
  }

  // Rebuild entire read model from scratch (useful when schema changes)
  async rebuildFromEvents(eventStore) {
    console.log('Rebuilding read model from event store...');
    await this.clickhouse.query('TRUNCATE TABLE ai_jobs_read_model');

    let offset = 0;
    const batchSize = 1000;

    while (true) {
      const events = await eventStore.getEvents({ offset, limit: batchSize });
      if (events.length === 0) break;

      for (const event of events) {
        if (event.type === 'AIJobCreated') await this.onAIJobCreated(event);
        if (event.type === 'AIJobCompleted') await this.onAIJobCompleted(event);
        if (event.type === 'AIJobFailed') await this.onAIJobFailed(event);
      }

      offset += batchSize;
      console.log(`Rebuilt ${offset} events...`);
    }

    console.log('Read model rebuild complete');
  }
}
```

### Pattern 3: Express API with CQRS

```javascript
const express = require('express');
const app = express();

app.use(express.json());

// ─── COMMAND endpoints (write) ───────────────────
// These return 202 Accepted (not the full resource)

app.post('/api/jobs', async (req, res) => {
  const jobId = require('crypto').randomUUID();
  try {
    await commandBus.dispatch({
      type: 'CreateAIJob',
      jobId,
      tenantId: req.headers['x-tenant-id'],
      userId: req.user.id,
      modelId: req.body.modelId,
      prompt: req.body.prompt
    });
    // Return 202 + job ID — not the full job object (read side handles that)
    res.status(202).json({ jobId, status: 'ACCEPTED' });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

app.post('/api/jobs/:jobId/complete', async (req, res) => {
  try {
    await commandBus.dispatch({
      type: 'CompleteAIJob',
      jobId: req.params.jobId,
      output: req.body.output,
      tokensUsed: req.body.tokensUsed,
      latencyMs: req.body.latencyMs,
      modelVersion: req.body.modelVersion
    });
    res.status(202).json({ status: 'ACCEPTED' });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// ─── QUERY endpoints (read) ───────────────────────
// These query the optimized read model

app.get('/api/dashboard/usage', async (req, res) => {
  const data = await queryBus.execute({
    type: 'AIUsageDashboard',
    tenantId: req.headers['x-tenant-id'],
    startDate: req.query.start,
    endDate: req.query.end,
    groupBy: req.query.groupBy || 'model'
  });
  res.json(data);
});

app.get('/api/jobs', async (req, res) => {
  const jobs = await queryBus.execute({
    type: 'RecentJobs',
    tenantId: req.headers['x-tenant-id'],
    userId: req.user.id,
    limit: parseInt(req.query.limit) || 20
  });
  res.json(jobs);
});
```

---

## AI-Specific Use Cases

### Use Case 1: AI Inference Logging + Analytics Dashboard

```
Write Side: Inference events (model, tokens, latency, cost)
  → PostgreSQL, append-only, write-optimized
  → Throughput: 10,000+ events/sec

Read Side A: Real-time dashboard
  → ClickHouse, pre-aggregated materialized views
  → Query: 0.2 seconds for "last 30 days by model"

Read Side B: ML Feature Store (for model retraining)
  → Parquet on S3, partitioned by date/model
  → Read by training pipeline, not by users

Read Side C: Billing
  → PostgreSQL aggregates, updated hourly
  → Used for invoice generation

Same events → 3 purpose-built read models → zero interference
```

### Use Case 2: AI Experiment Tracking

```javascript
// Commands: create experiment, log metric, complete run
// Queries: compare runs, find best hyperparameters, plot learning curves

class ExperimentTrackingCQRS {
  // Write side: append metric events
  async logMetric(command) {
    await commandBus.dispatch({
      type: 'LogExperimentMetric',
      experimentId: command.experimentId,
      runId: command.runId,
      step: command.step,
      metricName: command.metricName,
      value: command.value,
      timestamp: new Date()
    });
  }

  // Read side: compare best runs (complex query impossible on raw write model)
  async getBestRuns(query) {
    return queryBus.execute({
      type: 'BestRunsByMetric',
      experimentId: query.experimentId,
      metric: query.metric,       // 'val_accuracy'
      topK: query.topK || 10,
      hyperparameterKeys: query.hyperparameterKeys  // For comparison table
    });
    // Returns pre-aggregated: run_id, final_metric, hyperparams, duration
    // Impossible to compute efficiently from raw metric events at query time
  }
}
```

### Use Case 3: Multi-Model AI Observability

```javascript
// Different teams need different views of the same AI operations data

// Data Science team read model: raw metrics, per-step breakdowns
class MLTeamQueryHandler {
  async handle({ modelId, dateRange }) {
    return this.mlDB.query(
      `SELECT step, loss, accuracy, gradient_norm, learning_rate
       FROM training_metrics_raw
       WHERE model_id = ? AND timestamp BETWEEN ? AND ?
       ORDER BY step`,
      [modelId, dateRange.start, dateRange.end]
    );
  }
}

// Product team read model: high-level usage, by feature
class ProductTeamQueryHandler {
  async handle({ productArea, dateRange }) {
    return this.analyticsDB.query(
      `SELECT feature_name, active_users, avg_latency_ms, success_rate
       FROM ai_feature_summary
       WHERE product_area = ? AND week BETWEEN ? AND ?`,
      [productArea, dateRange.startWeek, dateRange.endWeek]
    );
  }
}

// Finance team read model: costs and billing
class FinanceTeamQueryHandler {
  async handle({ tenantId, billingPeriod }) {
    return this.billingDB.query(
      `SELECT model_name, total_tokens, total_cost_usd, cost_by_feature
       FROM monthly_ai_billing
       WHERE tenant_id = ? AND billing_period = ?`,
      [tenantId, billingPeriod]
    );
  }
}
```

---

## Advanced Patterns

### Pattern: Eventual Consistency Management

CQRS read models are eventually consistent. Handle this transparently.

```javascript
class EventuallyConsistentCQRS {
  constructor({ commandBus, queryBus, eventBus, consistencyTracker }) {
    this.commands = commandBus;
    this.queries = queryBus;
    this.events = eventBus;
    this.tracker = consistencyTracker;
  }

  async dispatchAndWait(command, { maxWaitMs = 2000 } = {}) {
    const result = await this.commands.dispatch(command);

    // Wait for read model to catch up (optional — only when needed)
    if (maxWaitMs > 0) {
      await this.tracker.waitForConsistency(result.eventId, maxWaitMs);
    }

    return result;
  }

  // After a command, tell the client the read model may lag
  commandResponse(jobId, eventId) {
    return {
      jobId,
      status: 'ACCEPTED',
      // Client can poll this endpoint to know when read model is updated
      consistencyUrl: `/api/consistency/${eventId}`,
      note: 'Dashboard reflects changes within ~2 seconds'
    };
  }
}

// Client-side: check if read model is caught up
app.get('/api/consistency/:eventId', async (req, res) => {
  const isCaughtUp = await consistencyTracker.isProcessed(req.params.eventId);
  res.json({ consistent: isCaughtUp });
});
```

### Pattern: Multiple Read Model Rebuild Strategies

```javascript
class ReadModelManager {
  // Full rebuild: drop and recreate from all events
  // Use: schema change, bug fix in projector, new read model needed
  async fullRebuild(projector, eventStore) {
    await projector.truncateReadModel();
    await projector.rebuildFromEvents(eventStore);
  }

  // Partial rebuild: replay only recent events
  // Use: temporary desync, deployment gap
  async partialRebuild(projector, eventStore, fromTimestamp) {
    const events = await eventStore.getEventsSince(fromTimestamp);
    for (const event of events) {
      await projector.project(event);
    }
  }

  // Shadow rebuild: rebuild new read model alongside old one
  // Use: migrating to new schema without downtime
  async shadowRebuild(oldProjector, newProjector, eventStore) {
    // 1. Rebuild new model from history
    await newProjector.rebuildFromEvents(eventStore);

    // 2. Start dual-writing to both
    eventStore.onNewEvent(event => {
      oldProjector.project(event);
      newProjector.project(event);
    });

    // 3. Validate new model matches old
    const isValid = await this._validateModels(oldProjector, newProjector);

    // 4. Switch traffic to new model
    if (isValid) await this.switchToNewModel(newProjector);
  }
}
```

---

## Monitoring & Observability

```javascript
class CQRSMetrics {
  getStats() {
    return {
      commandSide: {
        totalCommandsDispatched: this.commandsDispatched,
        commandSuccessRate: `${(this.commandsSucceeded / this.commandsDispatched * 100).toFixed(1)}%`,
        avgCommandLatencyMs: this._avg(this.commandLatencies),
        commandsByType: this.commandCounts
      },
      querySide: {
        totalQueriesExecuted: this.queriesExecuted,
        avgQueryLatencyMs: this._avg(this.queryLatencies),
        p99QueryLatencyMs: this._p99(this.queryLatencies),
        queriesByType: this.queryCounts
      },
      consistency: {
        avgReadModelLagMs: this.readModelLag,
        maxReadModelLagMs: this.maxReadModelLag,
        eventsInProjectionQueue: this.projectionQueueDepth,
        projectionErrors: this.projectionErrors
      }
    };
  }
}

// Example output:
// {
//   "commandSide": {
//     "totalCommandsDispatched": 50000,
//     "commandSuccessRate": "99.8%",
//     "avgCommandLatencyMs": 12
//   },
//   "querySide": {
//     "avgQueryLatencyMs": 38,
//     "p99QueryLatencyMs": 120
//   },
//   "consistency": {
//     "avgReadModelLagMs": 380,
//     "eventsInProjectionQueue": 42
//   }
// }
```

---

## Integration with Other Patterns

### CQRS + Event Sourcing

The most natural pairing. Event Sourcing provides the event stream that CQRS read models consume.

```
Events → Event Store (write side)
              ↓ event stream
         Projector → Read Model A (dashboard)
         Projector → Read Model B (search index)
         Projector → Read Model C (ML features)
```

### CQRS + Cache-Aside

Cache query results from the read model for frequently requested data.

```javascript
class CachedQueryBus extends QueryBus {
  constructor({ queryBus, cache, ttl = 60 }) {
    super();
    this.underlying = queryBus;
    this.cache = cache;
    this.ttl = ttl;
  }

  async execute(query) {
    const cacheKey = `query:${JSON.stringify(query)}`;

    const cached = await this.cache.get(cacheKey);
    if (cached) return cached;

    const result = await this.underlying.execute(query);
    await this.cache.set(cacheKey, result, this.ttl);
    return result;
  }
}
```

### CQRS + Bulkhead

Separate resource pools for commands and queries — a query storm cannot slow down writes.

```javascript
const pLimit = require('p-limit');

const commandLimiter = pLimit(100);  // Max 100 concurrent write ops
const queryLimiter = pLimit(500);    // Max 500 concurrent read ops

// Commands use their own pool
app.post('/api/jobs', (req, res) =>
  commandLimiter(() => handleCreateJob(req, res))
);

// Queries use a separate pool — never starve writes
app.get('/api/dashboard', (req, res) =>
  queryLimiter(() => handleDashboard(req, res))
);
```

---

## Real-World Case Studies

### Case Study 1: AI Model Observability Platform

**Problem:** Single PostgreSQL table serving both inference logging (writes) and analytics dashboard (reads). Read queries causing write timeouts. DB cost $8,000/month and growing.

**Solution:** CQRS with PostgreSQL for writes, ClickHouse for read models. Kafka connecting them.

**Results:**
- ✅ Write throughput: 500 → 12,000 events/sec
- ✅ Dashboard query latency: 42s → 0.3s
- ✅ DB cost: $8,000 → $1,400/month (saves $79K/year)
- ✅ Zero data gaps (no more lock contention)
- ✅ 3 new read models added without touching write path

### Case Study 2: Enterprise AI Experiment Tracker

**Problem:** ML teams needed complex cross-run comparisons. Same DB used for write (logging metrics) and read (generating comparison tables). Large experiments locked the DB.

**Solution:** CQRS separating metric writes from experiment comparison reads. Materialized views for leaderboard queries.

**Results:**
- ✅ Experiment comparison time: 4.5 minutes → 2.3 seconds
- ✅ ML team productivity: +35% (less waiting for queries)
- ✅ Write-heavy training jobs no longer affect dashboard users
- ✅ Added new read model (hyperparameter heatmaps) in 1 day — no write-side changes

### Case Study 3: AI Content Platform — Multi-Tenant

**Problem:** 300 enterprise tenants sharing one DB. Tenant A's heavy analytics query slowed Tenant B's content generation. Noisy neighbor problem.

**Solution:** CQRS with tenant-isolated read model shards. Commands go to shared write DB; read models are per-tenant ClickHouse schemas.

**Results:**
- ✅ Tenant isolation: guaranteed — read models physically separated
- ✅ Premium tenant P99 query latency: 8.4s → 0.18s
- ✅ Content generation throughput: +300% (not competing with analytics)
- ✅ Enterprise customers renewed: 100% of at-risk accounts retained

---

## Production Best Practices

### 1. Start Simple, Evolve to Full CQRS

```
Stage 1: Single DB, split handlers in code (no infrastructure change)
Stage 2: Add read replica (separate read traffic physically)
Stage 3: Add dedicated read DB (ClickHouse, Elasticsearch)
Stage 4: Event-driven projections (full CQRS + Event Sourcing)

Don't jump to Stage 4 on day one.
```

### 2. Handle Eventual Consistency Explicitly

```javascript
// ❌ BAD: Client expects immediate consistency
const job = await createJob(data);
const details = await getJob(job.id); // May not be in read model yet!

// ✅ GOOD: Acknowledge the lag
const { jobId } = await createJob(data);
res.json({
  jobId,
  message: 'Job created. Dashboard updates within 2 seconds.',
  pollUrl: `/api/jobs/${jobId}/status`
});
```

### 3. Make Projectors Idempotent

```javascript
// Events may be replayed — projectors must handle duplicates
async function onAIJobCompleted(event) {
  await clickhouse.query(
    `INSERT INTO ai_jobs_read_model (job_id, ...) VALUES (...)
     ON CONFLICT (job_id) DO UPDATE SET ...`  // Upsert, not insert
  );
}
```

### When to Use CQRS

✅ **Use CQRS For:**
- AI platforms with heavy analytics dashboards AND high write throughput
- Multi-tenant systems with different read requirements per tenant
- Systems where write schema and read schema are fundamentally different
- Experiment tracking, inference logging, AI observability

❌ **Don't Use CQRS For:**
- Simple CRUD apps where reads and writes share the same schema
- Small teams who lack bandwidth to maintain two models
- Systems where read data must be immediately consistent after writes
- Early-stage prototypes (add CQRS when scaling pain appears)

### ROI Summary

**Investment:**
- Architecture + implementation: 3–4 weeks = $20K
- Read model design + ClickHouse setup: 1 week = $5K
- Kafka / event bus setup: 3 days = $3K
- Monitoring: 2 days = $2K
- **Total: ~$30K**

**Returns (Annual, AI SaaS platform):**
- DB cost reduction ($8K → $1.4K/month): $79K
- Engineering time freed from DB firefighting: $96K
- Customer churn prevented (reliability improvement): $480K
- Feature velocity increase (independent models): $200K
- **Total: $855K/year**

**ROI: 2,850% first year**

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
