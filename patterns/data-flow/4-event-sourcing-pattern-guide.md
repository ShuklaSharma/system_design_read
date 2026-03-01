# Event Sourcing Pattern for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why Event Sourcing Matters for AI Systems](#why-event-sourcing-matters)
3. [Core Concepts](#core-concepts)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Monitoring & Observability](#monitoring-observability)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Real-World Case Studies](#real-world-case-studies)
10. [Production Best Practices](#production-best-practices)

---

## Introduction

### What is Event Sourcing?

**Event Sourcing** is a persistence pattern where the **state of a system is derived from a sequence of immutable events**, rather than storing only the current state. Instead of updating a row in a database ("current balance: $500"), you append events ("deposited $200", "withdrew $50", "deposited $350"). The current state is always computed by replaying the event history.

**Analogy:** Like a bank ledger — your bank doesn't just store "balance: $5,000". It stores every deposit, withdrawal, and transfer. Your balance is the sum of all those transactions. At any moment, you can rewind to see your balance on any past date, audit any transaction, or replay history to correct an error.

### The Problem Without Event Sourcing

```
Scenario: AI model training pipeline — debugging a bad model

Without Event Sourcing (store only current state):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
models table:
  id | name | status | accuracy | created_at | updated_at
  1  | gpt-ft-v3 | deployed | 0.71 | 2024-01-10 | 2024-03-15

Questions you CANNOT answer:
  ❌ "Why did accuracy drop from 0.88 to 0.71?"
  ❌ "When exactly was the training dataset changed?"
  ❌ "What hyperparameters were used in the good version?"
  ❌ "Which team member made the configuration change that caused the drop?"
  ❌ "Can I reproduce the state from 3 weeks ago to compare?"
  ❌ "What was the exact sequence of events that led to the bad model?"

The database shows ONLY the current state.
History is gone. Debugging is guesswork.
```

### The Solution With Event Sourcing

```
With Event Sourcing (store events):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
events table:
  id | aggregate_id | type                 | data                        | timestamp | user
  1  | model-1      | ModelCreated         | {name, dataset_v1, lr=0.01} | 2024-01-10 | alice
  2  | model-1      | TrainingStarted      | {gpu, epochs=10}            | 2024-01-10 | system
  3  | model-1      | TrainingCompleted    | {accuracy=0.88, loss=0.12}  | 2024-01-11 | system
  4  | model-1      | ModelDeployed        | {env=production}            | 2024-01-12 | bob
  5  | model-1      | DatasetChanged       | {old=v1, new=v2 (buggy!)}   | 2024-02-01 | charlie
  6  | model-1      | RetrainingStarted    | {same hyperparams}          | 2024-02-02 | system
  7  | model-1      | TrainingCompleted    | {accuracy=0.71, loss=0.31}  | 2024-02-03 | system

Now you CAN answer everything:
  ✅ "Why did accuracy drop?" → Dataset changed (event 5: charlie, dataset v2 "buggy")
  ✅ "What were the good hyperparameters?" → Event 1: lr=0.01, event 2: epochs=10
  ✅ "Who changed the dataset?" → charlie, Feb 1
  ✅ "Reproduce state from Jan 12?" → Replay events 1–4
  ✅ "Compare good vs bad training run?" → Events 1–3 vs 5–7
  ✅ Full audit trail, zero guesswork
```

---

## Why Event Sourcing Matters for AI Systems

### Unique AI System Challenges

1. **Reproducibility Is Critical**
   - "Why did this model make this prediction?" requires knowing the exact state at inference time
   - Regulatory AI (finance, healthcare) must prove reproducibility
   - Research reproducibility is a core scientific requirement
   - Debugging model regressions requires comparing past and present states

2. **Complex State with Many Contributors**
   - Data scientists, engineers, automated pipelines all modify model state
   - Without event sourcing: impossible to know who changed what when
   - With event sourcing: every state change is attributed and timestamped

3. **Long-Running Processes with Partial Failures**
   - Training runs that fail mid-way: which steps completed?
   - Batch ingestion that crashes: what was processed?
   - Multi-day fine-tuning: can you resume or must you restart?
   - Event sourcing provides the exact checkpoint to resume from

4. **Evolving Requirements for Historical Analysis**
   - "We need to add a new metric to the experiment tracker" — without ES, historical data is gone
   - With Event Sourcing: replay all historical events through the new metric computation
   - Analytics requirements change — events never change, read models can be rebuilt

### Real Incident: No Event Sourcing

**AI Drug Discovery Startup — Series C**

```
Timeline:
2023 - 18 months of AI model development
       Models stored as current state in PostgreSQL

Dec 2023 - FDA pre-submission review meeting
           Reviewer: "Can you reproduce Model v4.2's predictions from July?"
           Team: "...we can try"

Dec 10 - Engineers search for model artifacts
          → S3 bucket: model weights ✅
          → Training code: git history ✅
          → Training data: MISSING (dataset was "cleaned up" in Oct)
          → Hyperparameter config: overwritten by later runs ❌
          → Exact library versions: unknown ❌
          → Data preprocessing pipeline: refactored twice since July ❌

Dec 15 - Cannot reproduce July predictions
Dec 20 - FDA pre-submission fails reproducibility requirement

Impact:
- 6-month FDA timeline delay
- $2M additional regulatory work
- Additional data retention systems required
- Near-miss: could have voided 18 months of regulatory progress
- Hired 2 MLOps engineers specifically for reproducibility
- Total impact: $8M+ (delayed product launch)
```

**Same Situation WITH Event Sourcing:**

```
Dec 10 - FDA reviewer asks for Model v4.2 from July
          Team: "One moment"

Dec 10 - Engineer runs:
          eventStore.replayTo('model-v4.2', timestamp='2023-07-15')
          → Training data version: event #234 (DatasetVersionLocked: dataset_v8.2)
          → Hyperparameters: event #231 (HyperparamsSet: lr=0.0003, batch=32...)
          → Library versions: event #230 (EnvironmentSnapshotted: requirements.txt hash)
          → Preprocessing: event #229 (PipelineVersionPinned: pipeline_v3.1)

Dec 10 - Exact environment reconstructed in 2 hours
Dec 11 - Predictions reproduced to 4 decimal places ✅
Dec 12 - FDA review proceeds on schedule

Impact: Zero delay, zero additional cost, passed review
```

---

## Core Concepts

### Events

Events are immutable facts about things that happened. They are named in the past tense.

```javascript
// ✅ Good event names (past tense, specific):
'ModelTrainingStarted'
'HyperparametersUpdated'
'DatasetVersionLocked'
'InferenceFailed'
'ModelDeployedToProduction'

// ❌ Bad event names (present tense, generic):
'ModelUpdate'
'DataChange'
'StatusSet'
```

### Aggregates

An aggregate is the domain object whose state is built from events. Each aggregate has a unique ID and a sequence of events.

```javascript
class MLModel {
  constructor(id) {
    this.id = id;
    this.events = [];
    // Derived state:
    this.status = 'DRAFT';
    this.accuracy = null;
    this.hyperparameters = {};
    this.datasetVersion = null;
    this.version = 0;
  }

  // Apply an event to update state
  apply(event) {
    switch (event.type) {
      case 'ModelCreated':
        this.status = 'DRAFT';
        this.hyperparameters = event.data.hyperparameters;
        this.datasetVersion = event.data.datasetVersion;
        break;
      case 'TrainingCompleted':
        this.status = 'TRAINED';
        this.accuracy = event.data.accuracy;
        break;
      case 'ModelDeployed':
        this.status = 'DEPLOYED';
        this.deployedAt = event.data.deployedAt;
        break;
      case 'ModelRetired':
        this.status = 'RETIRED';
        break;
    }
    this.version++;
  }

  // Rebuild state by replaying all events
  static fromEvents(id, events) {
    const model = new MLModel(id);
    for (const event of events) {
      model.apply(event);
    }
    return model;
  }
}
```

### Event Store

The append-only log that persists all events. Never updated, only appended.

```
Event Store Structure:
  id (UUID) | aggregate_id | aggregate_type | sequence | type | data | metadata | timestamp

Properties:
  ✅ Append-only (never update or delete)
  ✅ Ordered by sequence per aggregate
  ✅ Optimistic concurrency: reject if sequence doesn't match expected
```

---

## Implementation Patterns

### Pattern 1: Event Store Implementation

```javascript
class EventStore {
  constructor({ db }) {
    this.db = db;
  }

  // Append events for an aggregate — with optimistic concurrency
  async append(aggregateId, aggregateType, events, expectedVersion) {
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');

      // Check current version (optimistic concurrency control)
      const currentVersionResult = await client.query(
        `SELECT MAX(sequence) as version FROM events WHERE aggregate_id = $1`,
        [aggregateId]
      );
      const currentVersion = currentVersionResult.rows[0].version ?? -1;

      if (currentVersion !== expectedVersion) {
        await client.query('ROLLBACK');
        throw new Error(
          `Concurrency conflict: expected version ${expectedVersion}, got ${currentVersion}`
        );
      }

      // Append all events
      for (let i = 0; i < events.length; i++) {
        const event = events[i];
        const sequence = currentVersion + 1 + i;

        await client.query(
          `INSERT INTO events
           (id, aggregate_id, aggregate_type, sequence, type, data, metadata, timestamp)
           VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())`,
          [
            event.id || require('crypto').randomUUID(),
            aggregateId,
            aggregateType,
            sequence,
            event.type,
            JSON.stringify(event.data),
            JSON.stringify(event.metadata || {})
          ]
        );
      }

      await client.query('COMMIT');
      console.log(`Appended ${events.length} events to aggregate ${aggregateId}`);

    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  // Retrieve all events for an aggregate
  async getEvents(aggregateId, { fromSequence = 0, toSequence = null } = {}) {
    let sql = `SELECT * FROM events
               WHERE aggregate_id = $1 AND sequence >= $2`;
    const params = [aggregateId, fromSequence];

    if (toSequence !== null) {
      sql += ` AND sequence <= $3`;
      params.push(toSequence);
    }
    sql += ` ORDER BY sequence ASC`;

    const result = await this.db.query(sql, params);
    return result.rows.map(row => ({
      ...row,
      data: JSON.parse(row.data),
      metadata: JSON.parse(row.metadata)
    }));
  }

  // Get all events across all aggregates (for projections)
  async getEventStream({ fromTimestamp, eventTypes, limit = 1000 } = {}) {
    let sql = `SELECT * FROM events WHERE 1=1`;
    const params = [];

    if (fromTimestamp) {
      params.push(fromTimestamp);
      sql += ` AND timestamp >= $${params.length}`;
    }
    if (eventTypes && eventTypes.length > 0) {
      params.push(eventTypes);
      sql += ` AND type = ANY($${params.length})`;
    }

    sql += ` ORDER BY timestamp ASC LIMIT $${params.length + 1}`;
    params.push(limit);

    const result = await this.db.query(sql, params);
    return result.rows.map(r => ({ ...r, data: JSON.parse(r.data) }));
  }

  // Snapshot: store current state to avoid replaying all events on next load
  async saveSnapshot(aggregateId, state, version) {
    await this.db.query(
      `INSERT INTO snapshots (aggregate_id, state, version, created_at)
       VALUES ($1, $2, $3, NOW())
       ON CONFLICT (aggregate_id) DO UPDATE SET state = $2, version = $3, created_at = NOW()`,
      [aggregateId, JSON.stringify(state), version]
    );
  }

  async getSnapshot(aggregateId) {
    const result = await this.db.query(
      `SELECT * FROM snapshots WHERE aggregate_id = $1`,
      [aggregateId]
    );
    if (result.rows.length === 0) return null;
    const snap = result.rows[0];
    return { state: JSON.parse(snap.state), version: snap.version };
  }
}
```

### Pattern 2: AI Model Aggregate with Event Sourcing

```javascript
class MLModelAggregate {
  constructor() {
    this.id = null;
    this.status = null;
    this.accuracy = null;
    this.hyperparameters = {};
    this.datasetVersion = null;
    this.trainingRuns = [];
    this.deployments = [];
    this._version = -1;
    this._pendingEvents = [];
  }

  // ─── Command methods (raise events, don't mutate state directly) ───

  static create(id, { name, hyperparameters, datasetVersion, createdBy }) {
    const model = new MLModelAggregate();
    model._raise({
      type: 'ModelCreated',
      data: { id, name, hyperparameters, datasetVersion, createdBy }
    });
    return model;
  }

  startTraining({ gpuSpec, epochs, createdBy }) {
    if (this.status !== 'DRAFT' && this.status !== 'TRAINED') {
      throw new Error(`Cannot start training from status: ${this.status}`);
    }
    this._raise({
      type: 'TrainingStarted',
      data: { gpuSpec, epochs, startedBy: createdBy, startedAt: new Date() }
    });
  }

  completeTraining({ accuracy, loss, epochs, duration, artifactPath }) {
    if (this.status !== 'TRAINING') {
      throw new Error('Cannot complete training — model is not training');
    }
    this._raise({
      type: 'TrainingCompleted',
      data: { accuracy, loss, epochs, duration, artifactPath, completedAt: new Date() }
    });
  }

  deploy({ environment, deployedBy }) {
    if (this.status !== 'TRAINED') {
      throw new Error(`Cannot deploy model with status: ${this.status}`);
    }
    if (this.accuracy < 0.80) {
      throw new Error(`Accuracy ${this.accuracy} below deployment threshold of 0.80`);
    }
    this._raise({
      type: 'ModelDeployed',
      data: { environment, deployedBy, deployedAt: new Date() }
    });
  }

  retire({ reason, retiredBy }) {
    this._raise({
      type: 'ModelRetired',
      data: { reason, retiredBy, retiredAt: new Date() }
    });
  }

  // ─── Apply events to update state ───

  apply(event) {
    switch (event.type) {
      case 'ModelCreated':
        this.id = event.data.id;
        this.status = 'DRAFT';
        this.hyperparameters = event.data.hyperparameters;
        this.datasetVersion = event.data.datasetVersion;
        this.name = event.data.name;
        break;

      case 'TrainingStarted':
        this.status = 'TRAINING';
        this.trainingRuns.push({ startedAt: event.data.startedAt });
        break;

      case 'TrainingCompleted':
        this.status = 'TRAINED';
        this.accuracy = event.data.accuracy;
        this.loss = event.data.loss;
        this.artifactPath = event.data.artifactPath;
        this.trainingRuns[this.trainingRuns.length - 1].completedAt = event.data.completedAt;
        break;

      case 'ModelDeployed':
        this.status = 'DEPLOYED';
        this.deployments.push({ environment: event.data.environment, deployedAt: event.data.deployedAt });
        break;

      case 'ModelRetired':
        this.status = 'RETIRED';
        break;
    }
    this._version++;
  }

  _raise(eventData) {
    const event = {
      id: require('crypto').randomUUID(),
      ...eventData,
      metadata: { raisedAt: new Date() }
    };
    this._pendingEvents.push(event);
    this.apply(event); // Apply immediately to update in-memory state
  }

  // Get pending events to save to event store
  flushPendingEvents() {
    const events = [...this._pendingEvents];
    this._pendingEvents = [];
    return events;
  }

  // Load from event store
  static fromHistory(events) {
    const model = new MLModelAggregate();
    for (const event of events) {
      model.apply(event);
    }
    return model;
  }
}
```

### Pattern 3: Repository with Event Store + Snapshots

```javascript
class MLModelRepository {
  constructor({ eventStore, snapshotInterval = 50 }) {
    this.eventStore = eventStore;
    this.snapshotInterval = snapshotInterval; // Snapshot every 50 events
  }

  async save(model) {
    const events = model.flushPendingEvents();
    if (events.length === 0) return;

    // Append events with optimistic concurrency
    await this.eventStore.append(
      model.id,
      'MLModel',
      events,
      model._version - events.length // Expected version before these events
    );

    // Create snapshot periodically to speed up future loads
    if (model._version % this.snapshotInterval === 0) {
      await this.eventStore.saveSnapshot(model.id, {
        status: model.status,
        accuracy: model.accuracy,
        hyperparameters: model.hyperparameters,
        datasetVersion: model.datasetVersion,
        trainingRuns: model.trainingRuns,
        deployments: model.deployments,
        name: model.name
      }, model._version);
      console.log(`Snapshot saved for model ${model.id} at version ${model._version}`);
    }
  }

  async load(modelId) {
    // Try to load from snapshot first (avoid replaying all events)
    const snapshot = await this.eventStore.getSnapshot(modelId);

    let model;
    let fromSequence = 0;

    if (snapshot) {
      model = new MLModelAggregate();
      Object.assign(model, snapshot.state);
      model._version = snapshot.version;
      fromSequence = snapshot.version + 1;
      console.log(`Loaded model ${modelId} from snapshot at version ${snapshot.version}`);
    } else {
      model = new MLModelAggregate();
    }

    // Replay any events since the snapshot
    const events = await this.eventStore.getEvents(modelId, { fromSequence });
    for (const event of events) {
      model.apply(event);
    }

    if (!model.id) throw new Error(`Model ${modelId} not found`);
    return model;
  }

  // Load state at a specific point in time (time travel!)
  async loadAt(modelId, timestamp) {
    const allEvents = await this.eventStore.getEvents(modelId);
    const eventsUpTo = allEvents.filter(e => new Date(e.timestamp) <= new Date(timestamp));
    return MLModelAggregate.fromHistory(eventsUpTo);
  }
}

// Usage
const repo = new MLModelRepository({ eventStore });

// Create and train a model
const model = MLModelAggregate.create('model-001', {
  name: 'sentiment-classifier-v1',
  hyperparameters: { lr: 0.0003, batch: 32, epochs: 10 },
  datasetVersion: 'dataset-v8',
  createdBy: 'alice'
});

model.startTraining({ gpuSpec: 'A100', epochs: 10, createdBy: 'alice' });
await repo.save(model);

// ... training runs async ...

const loadedModel = await repo.load('model-001');
loadedModel.completeTraining({ accuracy: 0.923, loss: 0.08, artifactPath: 's3://...' });
loadedModel.deploy({ environment: 'production', deployedBy: 'bob' });
await repo.save(loadedModel);

// Time travel: what was the model state on Jan 15?
const historicalState = await repo.loadAt('model-001', '2024-01-15T00:00:00Z');
console.log(historicalState.status); // 'DRAFT' (before training completed)
```

---

## AI-Specific Use Cases

### Use Case 1: ML Experiment Audit Trail

```javascript
class MLExperimentEventSourcing {
  constructor({ eventStore, projector }) {
    this.store = eventStore;
    this.projector = projector;
  }

  async logEpochMetrics(experimentId, { epoch, loss, valLoss, accuracy, valAccuracy }) {
    await this.store.append(experimentId, 'MLExperiment', [{
      type: 'EpochCompleted',
      data: { epoch, loss, valLoss, accuracy, valAccuracy, timestamp: new Date() }
    }], await this._currentVersion(experimentId));
  }

  async compareRuns(experimentId, runA, runB) {
    // Replay events for each run to get full metric history
    const eventsA = await this.store.getEvents(`${experimentId}:${runA}`);
    const eventsB = await this.store.getEvents(`${experimentId}:${runB}`);

    const metricsA = eventsA
      .filter(e => e.type === 'EpochCompleted')
      .map(e => e.data);
    const metricsB = eventsB
      .filter(e => e.type === 'EpochCompleted')
      .map(e => e.data);

    return {
      runA: { id: runA, metrics: metricsA, bestAccuracy: Math.max(...metricsA.map(m => m.valAccuracy)) },
      runB: { id: runB, metrics: metricsB, bestAccuracy: Math.max(...metricsB.map(m => m.valAccuracy)) }
    };
  }
}
```

### Use Case 2: RAG Document Lifecycle Tracking

```javascript
// Every document in a RAG system has a full event history
const documentEvents = [
  { type: 'DocumentUploaded',    data: { filename, size, uploadedBy } },
  { type: 'DocumentChunked',     data: { chunkCount, chunkSize, strategy } },
  { type: 'DocumentEmbedded',    data: { model, vectorDimensions, tokensUsed } },
  { type: 'DocumentIndexed',     data: { vectorDBCollection, insertedAt } },
  { type: 'DocumentQueried',     data: { query, chunksReturned, relevanceScore } },
  { type: 'DocumentOutdated',    data: { reason, detectedBy } },
  { type: 'DocumentReembedded',  data: { newModel, reason } },
  { type: 'DocumentDeleted',     data: { reason, deletedBy, gdprRequest: true } }
];

// Now you can answer:
// "Why did search quality drop?" → DocumentReembedded with different model
// "Which documents are GDPR-deleted but still in vector DB?" → find gaps
// "What was the embedding cost for tenant X in Q3?" → sum EmbeddedEvent tokens
```

### Use Case 3: AI Inference Audit for Compliance

```javascript
class ComplianceAuditableInference {
  constructor({ eventStore, llm }) {
    this.store = eventStore;
    this.llm = llm;
  }

  async infer(requestId, { tenantId, userId, prompt, modelId }) {
    const sessionId = `inference:${requestId}`;

    // Record request received
    await this.store.append(sessionId, 'InferenceSession', [{
      type: 'InferenceRequested',
      data: { tenantId, userId, prompt, modelId, timestamp: new Date() },
      metadata: { ipAddress: '...', userAgent: '...' }
    }], -1);

    let response;
    try {
      // Call the model
      response = await this.llm.complete(modelId, prompt);

      // Record successful inference
      await this.store.append(sessionId, 'InferenceSession', [{
        type: 'InferenceCompleted',
        data: {
          output: response.text,
          tokensIn: response.usage.prompt_tokens,
          tokensOut: response.usage.completion_tokens,
          latencyMs: response.latencyMs,
          modelVersion: response.model
        }
      }], 0);

    } catch (err) {
      await this.store.append(sessionId, 'InferenceSession', [{
        type: 'InferenceFailed',
        data: { error: err.message, errorCode: err.code }
      }], 0);
      throw err;
    }

    return response;
  }

  // Reproduce any historical inference for audit
  async auditRequest(requestId) {
    const events = await this.store.getEvents(`inference:${requestId}`);
    return {
      request: events.find(e => e.type === 'InferenceRequested')?.data,
      response: events.find(e => e.type === 'InferenceCompleted')?.data,
      failure: events.find(e => e.type === 'InferenceFailed')?.data,
      fullHistory: events
    };
  }
}
```

---

## Advanced Patterns

### Pattern: Snapshots for Performance

```javascript
class SnapshotOptimizedRepository {
  // Without snapshots: loading a model with 10,000 training events = slow
  // With snapshots: load snapshot at event 9,950 + replay only 50 events = fast

  async load(aggregateId) {
    const snapshot = await this.eventStore.getSnapshot(aggregateId);

    if (snapshot && snapshot.version > 1000) {
      // High event count — snapshot is critical for performance
      const recentEvents = await this.eventStore.getEvents(aggregateId, {
        fromSequence: snapshot.version + 1
      });
      const aggregate = this._restoreFromSnapshot(snapshot.state);
      recentEvents.forEach(e => aggregate.apply(e));
      return aggregate;
    }

    // Low event count — just replay from the beginning
    const allEvents = await this.eventStore.getEvents(aggregateId);
    return MLModelAggregate.fromHistory(allEvents);
  }
}
```

### Pattern: Projections — Building Read Models from Events

```javascript
class AIMetricsDashboardProjection {
  constructor({ eventStore, clickhouse }) {
    this.store = eventStore;
    this.clickhouse = clickhouse;
    this.lastProcessedTimestamp = null;
  }

  // Run continuously, processing new events as they arrive
  async startProjecting() {
    while (true) {
      const events = await this.store.getEventStream({
        fromTimestamp: this.lastProcessedTimestamp,
        eventTypes: ['TrainingCompleted', 'ModelDeployed', 'InferenceCompleted'],
        limit: 500
      });

      for (const event of events) {
        await this.project(event);
        this.lastProcessedTimestamp = event.timestamp;
      }

      await new Promise(r => setTimeout(r, 1000)); // Poll every second
    }
  }

  async project(event) {
    switch (event.type) {
      case 'TrainingCompleted':
        await this.clickhouse.insert('model_training_history', [{
          model_id: event.aggregate_id,
          accuracy: event.data.accuracy,
          loss: event.data.loss,
          duration_minutes: event.data.duration / 60,
          completed_at: event.timestamp
        }]);
        break;

      case 'InferenceCompleted':
        await this.clickhouse.insert('inference_metrics', [{
          model_id: event.data.modelId,
          tenant_id: event.data.tenantId,
          tokens_in: event.data.tokensIn,
          tokens_out: event.data.tokensOut,
          latency_ms: event.data.latencyMs,
          cost_usd: (event.data.tokensIn + event.data.tokensOut) * 0.00003,
          timestamp: event.timestamp
        }]);
        break;
    }
  }

  // Rebuild entire projection from scratch
  async rebuild() {
    console.log('Rebuilding AI metrics dashboard projection...');
    await this.clickhouse.query('TRUNCATE TABLE model_training_history');
    await this.clickhouse.query('TRUNCATE TABLE inference_metrics');
    this.lastProcessedTimestamp = null;
    await this.startProjecting(); // Will process all events from beginning
  }
}
```

### Pattern: Event Upcasting (Schema Evolution)

```javascript
class EventUpcaster {
  // Events are immutable — but their interpretation can evolve
  // Upcasting transforms old event formats to new ones at read time

  upcast(event) {
    switch (event.type) {
      case 'TrainingCompleted':
        // v1 events didn't have 'artifactPath' — add a default
        if (!event.data.artifactPath) {
          return {
            ...event,
            data: {
              ...event.data,
              artifactPath: `s3://legacy-models/${event.aggregate_id}/model.pt`
            }
          };
        }
        return event;

      case 'ModelDeployed':
        // Old events used 'env', new ones use 'environment'
        if (event.data.env && !event.data.environment) {
          return {
            ...event,
            data: { ...event.data, environment: event.data.env }
          };
        }
        return event;

      default:
        return event;
    }
  }

  upcastAll(events) {
    return events.map(e => this.upcast(e));
  }
}
```

---

## Monitoring & Observability

```javascript
class EventSourcingMetrics {
  getStats() {
    return {
      eventStore: {
        totalEventsStored: this.totalEvents,
        eventsAppendedPerSecond: this.appendRate,
        avgAppendLatencyMs: this.avgAppendLatency,
        storeSizeGB: this.storeSizeBytes / 1e9
      },
      aggregates: {
        totalAggregatesLoaded: this.loadsTotal,
        avgLoadLatencyMs: this.avgLoadLatency,
        snapshotHitRate: `${(this.snapshotHits / this.loadsTotal * 100).toFixed(1)}%`,
        avgEventsReplayedPerLoad: this.avgEventsReplayed
      },
      projections: {
        activeProjections: this.projectionCount,
        avgProjectionLagMs: this.projectionLag,
        projectionErrors: this.projectionErrors,
        eventsInProjectionQueue: this.projectionQueueSize
      }
    };
  }
}

// Example output:
// {
//   "eventStore": {
//     "totalEventsStored": 45820000,
//     "eventsAppendedPerSecond": 1200,
//     "storeSizeGB": 18.4
//   },
//   "aggregates": {
//     "snapshotHitRate": "94.2%",
//     "avgEventsReplayedPerLoad": 8.3
//   },
//   "projections": {
//     "avgProjectionLagMs": 420,
//     "projectionErrors": 0
//   }
// }
```

---

## Integration with Other Patterns

### Event Sourcing + CQRS (Natural Pair)

```
Events → Event Store (write side)
              ↓ projections
         Read Model A (dashboard)  ← CQRS query side
         Read Model B (search)     ← CQRS query side
         Read Model C (billing)    ← CQRS query side
```

### Event Sourcing + Saga

Sagas listen to events from the event store and orchestrate compensations. The event store provides the exact sequence needed for reliable saga coordination.

```javascript
// Saga listens for events from the event store
eventStore.subscribe('ModelTrainingCompleted', async (event) => {
  await sagaOrchestrator.handleEvent(event);
});

eventStore.subscribe('ModelTrainingFailed', async (event) => {
  await sagaOrchestrator.compensate(event.data.sagaId);
});
```

---

## Real-World Case Studies

### Case Study 1: AI Drug Discovery Platform

**Problem:** FDA required reproducibility of model predictions from any date. Without Event Sourcing, this was impossible — training configs were overwritten.

**Solution:** Full Event Sourcing for all ML model aggregates. Every hyperparameter change, dataset version, and training run recorded as immutable events.

**Results:**
- ✅ Full reproducibility to any timestamp — predictions matched to 4 decimal places
- ✅ FDA pre-submission passed on first attempt
- ✅ 18 months of historical reproducibility recovered from legacy system migration
- ✅ Audit prep time: 3 weeks → 2 hours (automated from event store)
- ✅ Prevented $8M delay in product launch

### Case Study 2: AI Trading System

**Problem:** "Why did the model make this trade?" — no history of model state at trade time. Post-trade analysis required but impossible.

**Solution:** Event Sourcing for model state + inference events. Every trade references the exact model event sequence that produced the decision.

**Results:**
- ✅ Can reproduce any trade decision from any date
- ✅ SEC audit: passed with complete trade decision traceability
- ✅ Bug detection time: weeks → hours (replay events to reproduce issue)
- ✅ Regulatory compliance cost: reduced 60% (audit is automated)

### Case Study 3: Enterprise RAG Platform

**Problem:** Document pipeline had no history — "why is search quality dropping?" impossible to answer. No way to know when a document was re-embedded, which model was used, or why quality changed.

**Solution:** Event Sourcing for document lifecycle. Every upload, chunk, embed, index, query, and delete recorded as events.

**Results:**
- ✅ Root cause analysis time: days → 20 minutes (replay document events)
- ✅ Added new search quality metric retrospectively (rebuilt projection from events)
- ✅ GDPR deletion audit: automated — event store shows exact deletion sequence
- ✅ Search quality issues traced to specific embedding model version change
- ✅ Rebuilt read model with new schema in 4 hours — no data loss

---

## Production Best Practices

### 1. Events Are Immutable — Never Update Them

```javascript
// ❌ NEVER do this
await db.query(`UPDATE events SET data = $1 WHERE id = $2`, [newData, eventId]);

// ✅ If you made an error, append a corrective event
await eventStore.append(aggregateId, 'MLModel', [{
  type: 'HyperparameterCorrected',
  data: { correction: 'lr was 0.001, should be 0.0001', correctedBy: 'alice' }
}], currentVersion);
```

### 2. Design Events Around Business Facts, Not Technical Operations

```javascript
// ❌ Technical (hard to understand, brittle)
{ type: 'RowUpdated', data: { table: 'models', column: 'status', value: 'deployed' } }

// ✅ Business fact (meaningful, stable)
{ type: 'ModelDeployedToProduction', data: { modelVersion, environment, approvedBy } }
```

### 3. Implement Snapshots Before You Need Them

```javascript
// Rule of thumb: add snapshots when aggregates regularly exceed 100 events
const SNAPSHOT_THRESHOLD = 100;

if (aggregate._version % SNAPSHOT_THRESHOLD === 0) {
  await repo.saveSnapshot(aggregate);
}
```

### 4. Plan Event Store Growth

```
Estimate storage needs:
  Events/day: 1,000,000
  Avg event size: 2KB
  Daily storage: 2GB
  Annual storage: 730GB

Plan for:
  - Partitioning by date (fast range queries)
  - Archiving old events to cold storage
  - Compression (events compress 5-10x)
```

### When to Use Event Sourcing

✅ **Use Event Sourcing For:**
- ML experiment tracking requiring reproducibility
- AI systems with regulatory audit requirements
- Complex domains with rich history (model lifecycle, document lifecycle)
- Systems where "what happened?" is as important as "what is the current state?"
- Any system where you need time-travel debugging

❌ **Don't Use Event Sourcing For:**
- Simple CRUD applications (massive overkill)
- Systems where exact historical state is never needed
- Small teams without operational maturity for event stores
- High-frequency simple counters (use simpler aggregation)
- External systems you don't control

### ROI Summary

**Investment:**
- Architecture & implementation: 4–6 weeks = $30K
- Event store setup & operations: 1 week = $5K
- Projection framework: 1 week = $5K
- Training team on patterns: 3 days = $3K
- **Total: ~$43K**

**Returns (Annual, AI platform with compliance needs):**
- Regulatory audit cost reduction: $200K
- Prevented compliance violation (1 event): $2–8M potential
- Debugging time reduction (80% faster root cause): $150K
- Feature velocity from replayable history: $100K
- **Total: $450K+/year (excluding major violation prevention)**

**ROI: 1,047%+ first year**

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
