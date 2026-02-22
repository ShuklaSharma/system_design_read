# Data Flow Patterns for Distributed Systems

## Table of Contents

1. [Saga Pattern](#1-saga-pattern)
2. [Two-Phase Commit (2PC)](#2-two-phase-commit-2pc)
3. [CQRS — Command Query Responsibility Segregation](#3-cqrs)
4. [Event Sourcing](#4-event-sourcing)
5. [Combining Patterns](#5-combining-patterns)
6. [Pattern Comparison & Decision Guide](#6-pattern-comparison--decision-guide)

---

## 1. Saga Pattern

### What Is It?

A **Saga** is a sequence of local transactions where each step publishes an event or message that triggers the next step. If any step fails, the saga executes **compensating transactions** to undo the work already done — giving you distributed transactions without distributed locks.

```
Traditional Distributed Transaction (ACID):
  ┌──────────────────────────────────────────────────────┐
  │  BEGIN GLOBAL TRANSACTION                            │
  │    Service A → Service B → Service C                 │
  │  COMMIT / ROLLBACK everything atomically             │
  └──────────────────────────────────────────────────────┘
  Problem: Locks held across services = tight coupling + failure cascades

Saga (Eventual Consistency):
  Step 1: Service A local tx  → success → emit event
  Step 2: Service B local tx  → success → emit event
  Step 3: Service C local tx  → FAIL    → compensate C → compensate B → compensate A
  Each service owns its own database. No global lock.
```

### The Two Saga Styles

#### Choreography-Based Saga
Services react to events from each other — no central coordinator.

```
[Order Service]──order.created──►[Payment Service]──payment.processed──►[Inventory Service]
      ▲                                   │                                      │
      │                           payment.failed                        stock.reserved
      └──────────────────────────── (compensate) ◄───── stock.failed ◄──────────┘
```

```javascript
// Order Service publishes event
class OrderService {
  async createOrder(orderData) {
    const order = await this.db.orders.create({
      ...orderData,
      status: 'PENDING'
    });

    // Publish event to trigger next saga step
    await this.eventBus.publish('order.created', {
      orderId: order.id,
      customerId: order.customerId,
      amount: order.totalAmount,
      items: order.items
    });

    return order;
  }

  // Compensation: called when a downstream step fails
  async cancelOrder(orderId, reason) {
    await this.db.orders.update(orderId, {
      status: 'CANCELLED',
      cancellationReason: reason
    });

    await this.eventBus.publish('order.cancelled', { orderId, reason });
    console.log(`Order ${orderId} compensated: ${reason}`);
  }
}

// Payment Service listens and reacts
class PaymentService {
  constructor() {
    this.eventBus.subscribe('order.created', this.handleOrderCreated.bind(this));
    this.eventBus.subscribe('inventory.failed', this.handleInventoryFailed.bind(this));
  }

  async handleOrderCreated(event) {
    try {
      const payment = await this.processPayment({
        orderId: event.orderId,
        amount: event.amount,
        customerId: event.customerId
      });

      await this.eventBus.publish('payment.processed', {
        orderId: event.orderId,
        paymentId: payment.id
      });

    } catch (err) {
      // Payment failed — trigger compensation upstream
      await this.eventBus.publish('payment.failed', {
        orderId: event.orderId,
        reason: err.message
      });
    }
  }

  // Compensation: refund when inventory step fails
  async handleInventoryFailed(event) {
    await this.refundPayment(event.orderId);
    await this.eventBus.publish('payment.refunded', { orderId: event.orderId });
  }
}
```

#### Orchestration-Based Saga
A central **Saga Orchestrator** drives the steps and handles failures.

```
                    ┌─────────────────────────┐
                    │    Order Saga Orchestrator│
                    └────────────┬────────────┘
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       [Payment Svc]    [Inventory Svc]    [Shipping Svc]
       step 1: charge    step 2: reserve   step 3: schedule
```

```javascript
class OrderSagaOrchestrator {
  constructor() {
    this.steps = [
      { name: 'Payment',   execute: this.chargePayment.bind(this),    compensate: this.refundPayment.bind(this) },
      { name: 'Inventory', execute: this.reserveStock.bind(this),     compensate: this.releaseStock.bind(this) },
      { name: 'Shipping',  execute: this.scheduleShipment.bind(this), compensate: this.cancelShipment.bind(this) }
    ];
  }

  async execute(order) {
    const executedSteps = [];

    for (const step of this.steps) {
      try {
        console.log(`▶ Executing step: ${step.name}`);
        const result = await step.execute(order);
        executedSteps.push({ step, result });
        console.log(`✅ Step ${step.name} succeeded`);
      } catch (err) {
        console.error(`❌ Step ${step.name} failed: ${err.message}`);
        await this.compensate(executedSteps, err);
        throw new Error(`Saga failed at step [${step.name}]: ${err.message}`);
      }
    }

    return { success: true, orderId: order.id };
  }

  async compensate(executedSteps, originalError) {
    console.log('🔄 Starting compensation...');
    // Compensate in reverse order
    for (const { step, result } of executedSteps.reverse()) {
      try {
        await step.compensate(result);
        console.log(`✅ Compensated: ${step.name}`);
      } catch (compErr) {
        // Log but continue compensating remaining steps
        console.error(`⚠️ Compensation failed for ${step.name}:`, compErr.message);
      }
    }
  }

  async chargePayment(order) {
    const response = await paymentService.charge(order.customerId, order.amount);
    return { paymentId: response.id };
  }

  async refundPayment({ paymentId }) {
    await paymentService.refund(paymentId);
  }

  async reserveStock(order) {
    const reservation = await inventoryService.reserve(order.items);
    return { reservationId: reservation.id };
  }

  async releaseStock({ reservationId }) {
    await inventoryService.release(reservationId);
  }

  async scheduleShipment(order) {
    const shipment = await shippingService.schedule(order);
    return { shipmentId: shipment.id };
  }

  async cancelShipment({ shipmentId }) {
    await shippingService.cancel(shipmentId);
  }
}

// Usage
const saga = new OrderSagaOrchestrator();
try {
  await saga.execute(order);
  console.log('Order placed successfully!');
} catch (err) {
  console.error('Order failed, all compensations applied:', err.message);
}
```

### Saga State Machine with Persistence

```javascript
// Persist saga state so it survives crashes
class PersistentSaga {
  async createSagaInstance(sagaType, payload) {
    return await this.db.sagas.create({
      type: sagaType,
      status: 'STARTED',
      currentStep: 0,
      payload,
      completedSteps: [],
      createdAt: new Date()
    });
  }

  async advanceStep(sagaId, stepResult) {
    await this.db.sagas.update(sagaId, {
      currentStep: this.db.sagas.currentStep + 1,
      completedSteps: [...this.db.sagas.completedSteps, stepResult],
      updatedAt: new Date()
    });
  }

  async markComplete(sagaId) {
    await this.db.sagas.update(sagaId, {
      status: 'COMPLETED',
      completedAt: new Date()
    });
  }

  async markFailed(sagaId, error) {
    await this.db.sagas.update(sagaId, {
      status: 'COMPENSATING',
      failureReason: error.message,
      updatedAt: new Date()
    });
  }

  // Resume a saga that was interrupted mid-flight
  async resumeSaga(sagaId) {
    const saga = await this.db.sagas.findById(sagaId);
    if (saga.status === 'COMPENSATING') {
      return await this.compensate(saga.completedSteps);
    }
    return await this.executeFrom(saga.currentStep, saga.payload);
  }
}
```

### When to Use Saga

| Scenario | Choreography | Orchestration |
|---|---|---|
| Simple, few services | ✅ Best fit | Overkill |
| Complex business logic | ❌ Hard to track | ✅ Best fit |
| Clear ownership per team | ✅ Decoupled | Needs central owner |
| Easy debugging/audit | ❌ Events spread everywhere | ✅ Central log |
| Avoiding single point of failure | ✅ No coordinator | ❌ Coordinator is SPOF |

---

## 2. Two-Phase Commit (2PC)

### What Is It?

**Two-Phase Commit** is a distributed protocol that ensures all participants in a transaction either all commit or all roll back — providing true atomicity across multiple databases or services.

```
Phase 1 — Prepare (Voting):
  Coordinator ──► Participant A: "Can you commit?"  → "YES" (locks resources)
  Coordinator ──► Participant B: "Can you commit?"  → "YES" (locks resources)
  Coordinator ──► Participant C: "Can you commit?"  → "NO"  (cannot proceed)

Phase 2 — Commit or Abort:
  If ALL said YES → Coordinator sends COMMIT to all
  If ANY said NO  → Coordinator sends ABORT to all (everyone rolls back)
```

### Flow Diagram

```
HAPPY PATH:
  Coordinator          DB_A              DB_B
      │──── PREPARE ───►│                 │
      │                  │◄── READY ───────│  (Phase 1: Both vote YES)
      │──── PREPARE ─────────────────────►│
      │◄──── READY ──────────────────────│
      │                  │                │
      │──── COMMIT ─────►│                │  (Phase 2: Commit to all)
      │──── COMMIT ──────────────────────►│
      │◄─── ACK ─────────│                │
      │◄─── ACK ──────────────────────────│
      │  ✅ Transaction Complete           │

FAILURE PATH (DB_B votes NO):
  Coordinator          DB_A              DB_B
      │──── PREPARE ───►│                 │
      │◄─── READY ──────│                 │
      │──── PREPARE ─────────────────────►│
      │◄─── NOT READY ────────────────────│  (Phase 1: DB_B votes NO)
      │                  │                │
      │──── ABORT ──────►│                │  (Phase 2: Abort all)
      │──── ABORT ───────────────────────►│
      │◄─── ACK ─────────│                │
      │◄─── ACK ──────────────────────────│
      │  ❌ All rolled back               │
```

### Implementation

```javascript
class TwoPhaseCommitCoordinator {
  constructor(participants) {
    this.participants = participants; // Array of participant service clients
    this.transactionLog = [];        // Durable log for crash recovery
  }

  async executeTransaction(transactionId, operations) {
    console.log(`\n🚀 Starting 2PC transaction: ${transactionId}`);

    // Log transaction start (for crash recovery)
    await this.logTransaction(transactionId, 'STARTED', operations);

    // ─── PHASE 1: PREPARE ───────────────────────────────────────────
    console.log('\n📋 PHASE 1: Sending PREPARE to all participants...');
    const prepareResults = await this.sendPrepare(transactionId, operations);

    const allVotedYes = prepareResults.every(r => r.vote === 'YES');

    // ─── PHASE 2: COMMIT or ABORT ───────────────────────────────────
    if (allVotedYes) {
      console.log('\n✅ All participants voted YES → Sending COMMIT');
      await this.logTransaction(transactionId, 'COMMITTING');
      await this.sendCommit(transactionId);
      await this.logTransaction(transactionId, 'COMMITTED');
      return { success: true, transactionId };
    } else {
      const failedParticipant = prepareResults.find(r => r.vote !== 'YES');
      console.log(`\n❌ Participant voted NO (${failedParticipant.participantId}) → Sending ABORT`);
      await this.logTransaction(transactionId, 'ABORTING');
      await this.sendAbort(transactionId);
      await this.logTransaction(transactionId, 'ABORTED');
      return { success: false, transactionId, reason: failedParticipant.reason };
    }
  }

  async sendPrepare(transactionId, operations) {
    const preparePromises = this.participants.map(async (participant, i) => {
      try {
        const operation = operations[i];
        const response = await participant.prepare(transactionId, operation);
        console.log(`  Participant ${participant.id}: ${response.vote}`);
        return { participantId: participant.id, vote: response.vote };
      } catch (err) {
        console.log(`  Participant ${participant.id}: TIMEOUT/ERROR → treating as NO`);
        return { participantId: participant.id, vote: 'NO', reason: err.message };
      }
    });

    return Promise.all(preparePromises);
  }

  async sendCommit(transactionId) {
    await Promise.all(
      this.participants.map(p => this.sendWithRetry(p, 'commit', transactionId))
    );
  }

  async sendAbort(transactionId) {
    await Promise.all(
      this.participants.map(p => this.sendWithRetry(p, 'abort', transactionId))
    );
  }

  // Commit/Abort must be retried until acknowledged — they must not fail
  async sendWithRetry(participant, action, transactionId, maxRetries = 10) {
    for (let i = 0; i < maxRetries; i++) {
      try {
        await participant[action](transactionId);
        return;
      } catch (err) {
        console.warn(`  Retry ${i + 1}/${maxRetries} for ${participant.id}.${action}...`);
        await this.sleep(1000 * Math.pow(2, i)); // Exponential backoff
      }
    }
    throw new Error(`Failed to ${action} participant ${participant.id} after ${maxRetries} retries`);
  }

  async logTransaction(transactionId, status, data = {}) {
    // Write to durable storage (crash-safe)
    await this.durableLog.write({ transactionId, status, data, timestamp: Date.now() });
  }

  sleep(ms) { return new Promise(resolve => setTimeout(resolve, ms)); }
}

// Participant (each database/service implements this)
class TwoPhaseCommitParticipant {
  constructor(db) {
    this.db = db;
    this.preparedTransactions = new Map(); // In-memory write-ahead log
  }

  async prepare(transactionId, operation) {
    try {
      // Validate and pre-execute the operation (but don't commit)
      await this.db.beginTransaction();
      await this.db.execute(operation.sql, operation.params);

      // Hold the lock — write to WAL but do NOT commit yet
      this.preparedTransactions.set(transactionId, {
        operation,
        dbTransaction: this.db.currentTransaction()
      });

      return { vote: 'YES' };
    } catch (err) {
      await this.db.rollback();
      return { vote: 'NO', reason: err.message };
    }
  }

  async commit(transactionId) {
    const prepared = this.preparedTransactions.get(transactionId);
    if (!prepared) throw new Error(`No prepared transaction: ${transactionId}`);

    await this.db.commit(prepared.dbTransaction);
    this.preparedTransactions.delete(transactionId);
    console.log(`✅ Committed transaction ${transactionId}`);
  }

  async abort(transactionId) {
    const prepared = this.preparedTransactions.get(transactionId);
    if (!prepared) return; // Already cleaned up — idempotent

    await this.db.rollback(prepared.dbTransaction);
    this.preparedTransactions.delete(transactionId);
    console.log(`🔄 Aborted transaction ${transactionId}`);
  }
}
```

### 2PC vs Saga Comparison

| Aspect | Two-Phase Commit | Saga |
|---|---|---|
| **Consistency** | Strong (ACID) | Eventual |
| **Locking** | Holds locks during protocol | No distributed locks |
| **Availability** | Lower (blocking) | Higher |
| **Performance** | Slower (synchronous) | Faster (async) |
| **Failure recovery** | Coordinator crash = blocked | Compensation logic needed |
| **Best for** | Financial transfers, critical data | Microservices, long-running workflows |

### When to Use 2PC

- You require true atomicity (no partial commits acceptable)
- Short-lived operations where lock duration is acceptable
- Working within a controlled environment (same data center, same vendor)
- Examples: bank ledger transfers, inventory deduction + payment deduction

---

## 3. CQRS

### What Is It?

**Command Query Responsibility Segregation** separates the model for writing data (Commands) from the model for reading data (Queries). You maintain different data stores optimised for each concern.

```
WITHOUT CQRS — One model does everything:
  Client ──► Single Service ──► Single DB ──► Same model for reads & writes
  Problem: Write optimisation (normalised) conflicts with read optimisation (denormalised)

WITH CQRS — Separate write and read paths:
  Client
    │──── Command (write) ──► Command Handler ──► Write DB (normalised, consistent)
    │                                │
    │                         Domain Event ──► Projector
    │                                              │
    │                                        Read DB (denormalised, fast)
    └──── Query (read) ──────────────────────────► Query Handler ──► Read DB
```

### Core Concepts

| Term | Meaning |
|---|---|
| **Command** | An intent to change state: `PlaceOrder`, `TransferFunds` |
| **Query** | A request for data that does NOT change state |
| **Command Handler** | Validates command, applies domain logic, persists to write store |
| **Query Handler** | Reads from the read-optimised store (no business logic) |
| **Projector / Read Model Builder** | Listens to domain events and updates the read store |

### Implementation

```javascript
// ─── COMMANDS ─────────────────────────────────────────────────────────────
class PlaceOrderCommand {
  constructor({ customerId, items, shippingAddress }) {
    this.type = 'PlaceOrder';
    this.customerId = customerId;
    this.items = items;
    this.shippingAddress = shippingAddress;
    this.timestamp = new Date();
  }
}

class CancelOrderCommand {
  constructor({ orderId, reason }) {
    this.type = 'CancelOrder';
    this.orderId = orderId;
    this.reason = reason;
    this.timestamp = new Date();
  }
}

// ─── COMMAND HANDLERS ─────────────────────────────────────────────────────
class OrderCommandHandler {
  constructor(writeDb, eventBus) {
    this.writeDb = writeDb;
    this.eventBus = eventBus;
  }

  async handle(command) {
    switch (command.type) {
      case 'PlaceOrder':    return this.handlePlaceOrder(command);
      case 'CancelOrder':   return this.handleCancelOrder(command);
      default: throw new Error(`Unknown command: ${command.type}`);
    }
  }

  async handlePlaceOrder(command) {
    // Validate
    if (!command.items?.length) throw new Error('Order must have at least one item');

    // Load aggregate from WRITE store (normalised, consistent)
    const customer = await this.writeDb.customers.findById(command.customerId);
    if (!customer) throw new Error('Customer not found');

    // Apply business rules
    const order = Order.create({
      customerId: command.customerId,
      items: command.items,
      shippingAddress: command.shippingAddress,
      status: 'PENDING',
      totalAmount: this.calculateTotal(command.items)
    });

    // Persist to write store
    await this.writeDb.orders.insert(order);

    // Publish domain event → will update the read model
    await this.eventBus.publish('OrderPlaced', {
      orderId: order.id,
      customerId: command.customerId,
      items: command.items,
      totalAmount: order.totalAmount,
      placedAt: new Date()
    });

    return { orderId: order.id };
  }

  async handleCancelOrder(command) {
    const order = await this.writeDb.orders.findById(command.orderId);
    if (!order) throw new Error('Order not found');
    if (order.status === 'SHIPPED') throw new Error('Cannot cancel a shipped order');

    await this.writeDb.orders.update(command.orderId, {
      status: 'CANCELLED',
      cancelledAt: new Date(),
      cancellationReason: command.reason
    });

    await this.eventBus.publish('OrderCancelled', {
      orderId: command.orderId,
      reason: command.reason,
      cancelledAt: new Date()
    });
  }

  calculateTotal(items) {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

// ─── QUERY HANDLERS ───────────────────────────────────────────────────────
class OrderQueryHandler {
  constructor(readDb) {
    this.readDb = readDb; // Separate, denormalised read store
  }

  // Optimised for fast list rendering — no joins, pre-computed fields
  async getOrdersByCustomer(customerId, { page = 1, limit = 20 } = {}) {
    return this.readDb.orderSummaries.find({
      filter: { customerId },
      sort: { placedAt: -1 },
      skip: (page - 1) * limit,
      limit
    });
  }

  // Optimised for order detail page — all data in one document
  async getOrderDetail(orderId) {
    return this.readDb.orderDetails.findOne({ orderId });
  }

  // Aggregate query — pre-computed in read model
  async getDashboardStats(customerId) {
    return this.readDb.customerStats.findOne({ customerId });
  }
}

// ─── READ MODEL PROJECTOR ─────────────────────────────────────────────────
// Listens to domain events and keeps the read store up to date
class OrderProjector {
  constructor(readDb, eventBus) {
    this.readDb = readDb;
    eventBus.subscribe('OrderPlaced',    this.onOrderPlaced.bind(this));
    eventBus.subscribe('OrderShipped',   this.onOrderShipped.bind(this));
    eventBus.subscribe('OrderCancelled', this.onOrderCancelled.bind(this));
  }

  async onOrderPlaced(event) {
    // Upsert the denormalised summary (optimised for list view)
    await this.readDb.orderSummaries.upsert({
      orderId: event.orderId,
      customerId: event.customerId,
      itemCount: event.items.length,
      totalAmount: event.totalAmount,
      status: 'PENDING',
      placedAt: event.placedAt
    });

    // Upsert the detail document (optimised for detail view)
    await this.readDb.orderDetails.upsert({
      orderId: event.orderId,
      customerId: event.customerId,
      items: event.items,
      totalAmount: event.totalAmount,
      status: 'PENDING',
      placedAt: event.placedAt,
      timeline: [{ status: 'PENDING', timestamp: event.placedAt }]
    });

    // Update pre-computed stats
    await this.readDb.customerStats.increment({
      customerId: event.customerId,
      totalOrders: 1,
      totalSpend: event.totalAmount
    });
  }

  async onOrderShipped(event) {
    await this.readDb.orderSummaries.update(event.orderId, { status: 'SHIPPED', shippedAt: event.shippedAt });
    await this.readDb.orderDetails.push(event.orderId, 'timeline', {
      status: 'SHIPPED',
      timestamp: event.shippedAt,
      trackingNumber: event.trackingNumber
    });
  }

  async onOrderCancelled(event) {
    await this.readDb.orderSummaries.update(event.orderId, { status: 'CANCELLED' });
    await this.readDb.orderDetails.push(event.orderId, 'timeline', {
      status: 'CANCELLED',
      reason: event.reason,
      timestamp: event.cancelledAt
    });
    await this.readDb.customerStats.increment({
      customerId: event.customerId,
      cancelledOrders: 1
    });
  }
}

// ─── API LAYER (wiring it together) ──────────────────────────────────────
app.post('/orders', async (req, res) => {
  // Commands go to the command handler
  const command = new PlaceOrderCommand(req.body);
  const result = await orderCommandHandler.handle(command);
  res.status(201).json(result);
});

app.get('/orders', async (req, res) => {
  // Queries go to the read model — no write DB involved
  const orders = await orderQueryHandler.getOrdersByCustomer(
    req.user.id,
    { page: req.query.page, limit: req.query.limit }
  );
  res.json(orders);
});
```

### Read Model Rebuild

One of CQRS's superpowers — you can rebuild the entire read model by replaying events:

```javascript
class ReadModelRebuilder {
  async rebuild(projector, eventStore) {
    console.log('🔄 Rebuilding read model...');
    await this.readDb.orderSummaries.deleteAll();
    await this.readDb.orderDetails.deleteAll();
    await this.readDb.customerStats.deleteAll();

    const allEvents = await eventStore.getAllEvents({ orderedByTime: true });
    let count = 0;

    for (const event of allEvents) {
      await projector.apply(event);   // Re-apply every historical event
      count++;
      if (count % 1000 === 0) console.log(`  Processed ${count} events...`);
    }

    console.log(`✅ Read model rebuilt from ${count} events`);
  }
}
```

### When to Use CQRS

- Read and write workloads have very different scaling needs
- Complex domain logic on writes, simple fast queries on reads
- Multiple read representations of the same data needed
- You need to add new query models without changing write logic

---

## 4. Event Sourcing

### What Is It?

Instead of storing the *current state* of an entity, **Event Sourcing** stores the full sequence of events that led to that state. The current state is derived by replaying all events.

```
Traditional (State-based):
  orders table:
  ┌─────────┬──────────┬────────────┐
  │ order_id│  status  │   amount   │
  ├─────────┼──────────┼────────────┤
  │   001   │ SHIPPED  │  $150.00   │  ← Only current state; history lost
  └─────────┴──────────┴────────────┘

Event Sourced:
  event_store table:
  ┌─────────┬─────────────────┬──────────────────────────────────┬───────────┐
  │event_id │     type        │           payload                │ timestamp │
  ├─────────┼─────────────────┼──────────────────────────────────┼───────────┤
  │    1    │ OrderPlaced     │ {items: [...], amount: 150}      │ 09:00     │
  │    2    │ PaymentReceived │ {paymentId: "px_123"}            │ 09:01     │
  │    3    │ ItemShipped     │ {trackingNo: "TRK_456"}          │ 10:30     │  ← Full audit trail
  └─────────┴─────────────────┴──────────────────────────────────┴───────────┘
  Current state = replay all 3 events
```

### Core Implementation

```javascript
// ─── EVENT STORE ──────────────────────────────────────────────────────────
class EventStore {
  constructor(db) {
    this.db = db;
  }

  // Append events with optimistic concurrency control
  async appendEvents(streamId, events, expectedVersion) {
    const currentVersion = await this.getCurrentVersion(streamId);

    // Optimistic concurrency — detect concurrent modifications
    if (currentVersion !== expectedVersion) {
      throw new ConcurrencyError(
        `Stream ${streamId} expected version ${expectedVersion}, got ${currentVersion}`
      );
    }

    const eventsToStore = events.map((event, i) => ({
      streamId,
      version: expectedVersion + i + 1,
      type: event.type,
      payload: JSON.stringify(event.payload),
      metadata: JSON.stringify(event.metadata || {}),
      timestamp: new Date()
    }));

    await this.db.events.insertMany(eventsToStore);
    return { newVersion: expectedVersion + events.length };
  }

  // Retrieve all events for an aggregate (for replaying)
  async getEvents(streamId, fromVersion = 0) {
    const rows = await this.db.events.find({
      streamId,
      version: { $gte: fromVersion }
    }).sort({ version: 1 });

    return rows.map(row => ({
      ...row,
      payload: JSON.parse(row.payload),
      metadata: JSON.parse(row.metadata)
    }));
  }

  async getCurrentVersion(streamId) {
    const last = await this.db.events.findOne({ streamId }, { sort: { version: -1 } });
    return last ? last.version : 0;
  }
}

// ─── AGGREGATE ────────────────────────────────────────────────────────────
class Order {
  constructor() {
    this.id = null;
    this.status = null;
    this.items = [];
    this.totalAmount = 0;
    this.version = 0;               // Tracks which events have been applied
    this.pendingEvents = [];         // Events raised but not yet persisted
  }

  // Reconstitute state from event history
  static fromHistory(events) {
    const order = new Order();
    for (const event of events) {
      order.apply(event, false); // false = don't add to pendingEvents
    }
    return order;
  }

  // ── Commands (raise events, do NOT mutate state directly) ──
  place({ items, customerId, shippingAddress }) {
    if (this.status) throw new Error('Order already placed');
    this.raise({
      type: 'OrderPlaced',
      payload: { orderId: this.id, customerId, items, shippingAddress, totalAmount: this.calcTotal(items) }
    });
  }

  confirmPayment({ paymentId }) {
    if (this.status !== 'PENDING') throw new Error(`Cannot confirm payment for status: ${this.status}`);
    this.raise({ type: 'PaymentConfirmed', payload: { paymentId } });
  }

  ship({ trackingNumber, carrier }) {
    if (this.status !== 'PAID') throw new Error('Order must be paid before shipping');
    this.raise({ type: 'OrderShipped', payload: { trackingNumber, carrier, shippedAt: new Date() } });
  }

  cancel({ reason }) {
    if (['SHIPPED', 'DELIVERED', 'CANCELLED'].includes(this.status)) {
      throw new Error(`Cannot cancel order with status: ${this.status}`);
    }
    this.raise({ type: 'OrderCancelled', payload: { reason, cancelledAt: new Date() } });
  }

  // ── Event application (state mutations ONLY happen here) ──
  apply(event, isNew = true) {
    switch (event.type) {
      case 'OrderPlaced':
        this.id = event.payload.orderId;
        this.customerId = event.payload.customerId;
        this.items = event.payload.items;
        this.totalAmount = event.payload.totalAmount;
        this.status = 'PENDING';
        break;

      case 'PaymentConfirmed':
        this.status = 'PAID';
        this.paymentId = event.payload.paymentId;
        break;

      case 'OrderShipped':
        this.status = 'SHIPPED';
        this.trackingNumber = event.payload.trackingNumber;
        this.shippedAt = event.payload.shippedAt;
        break;

      case 'OrderCancelled':
        this.status = 'CANCELLED';
        this.cancellationReason = event.payload.reason;
        break;
    }

    this.version++;
    if (isNew) this.pendingEvents.push(event);
  }

  raise(event) {
    this.apply(event, true);
  }

  clearPendingEvents() {
    this.pendingEvents = [];
  }

  calcTotal(items) {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

// ─── REPOSITORY ───────────────────────────────────────────────────────────
class OrderRepository {
  constructor(eventStore) {
    this.eventStore = eventStore;
  }

  async findById(orderId) {
    const events = await this.eventStore.getEvents(orderId);
    if (!events.length) return null;
    return Order.fromHistory(events);
  }

  async save(order) {
    if (!order.pendingEvents.length) return;

    await this.eventStore.appendEvents(
      order.id,
      order.pendingEvents,
      order.version - order.pendingEvents.length  // Expected version before new events
    );

    order.clearPendingEvents();
  }
}
```

### Snapshots (Performance Optimization)

For aggregates with thousands of events, replaying all events on every load is expensive. Snapshots cache state at a point in time:

```javascript
class SnapshotRepository {
  constructor(eventStore, snapshotStore) {
    this.eventStore = eventStore;
    this.snapshotStore = snapshotStore;
    this.snapshotThreshold = 50; // Take snapshot every 50 events
  }

  async findById(orderId) {
    // 1. Load most recent snapshot (if any)
    const snapshot = await this.snapshotStore.findLatest(orderId);

    let order;
    let fromVersion = 0;

    if (snapshot) {
      // Restore from snapshot — skip all events before it
      order = Order.fromSnapshot(snapshot.state);
      fromVersion = snapshot.version;
      console.log(`📸 Restored from snapshot at version ${fromVersion}`);
    } else {
      order = new Order();
    }

    // 2. Apply only events AFTER the snapshot
    const recentEvents = await this.eventStore.getEvents(orderId, fromVersion + 1);
    for (const event of recentEvents) {
      order.apply(event, false);
    }

    return order;
  }

  async save(order) {
    await this.eventStore.appendEvents(order.id, order.pendingEvents, order.version - order.pendingEvents.length);
    order.clearPendingEvents();

    // Take a snapshot if threshold reached
    if (order.version % this.snapshotThreshold === 0) {
      await this.snapshotStore.save({
        streamId: order.id,
        version: order.version,
        state: order.toSnapshot(),   // Serialise current state
        takenAt: new Date()
      });
      console.log(`📸 Snapshot saved at version ${order.version}`);
    }
  }
}
```

### Event Sourcing Projections

Events drive multiple read models (works naturally with CQRS):

```javascript
class OrderProjection {
  constructor(readDb, eventStore) {
    this.readDb = readDb;
    this.eventStore = eventStore;
  }

  async rebuildAll() {
    console.log('🔄 Rebuilding all projections from event store...');
    const allEvents = await this.eventStore.getAllEvents();

    for (const event of allEvents) {
      await this.handleEvent(event);
    }
  }

  async handleEvent(event) {
    switch (event.type) {
      case 'OrderPlaced':
        await this.readDb.orders.upsert({
          id: event.payload.orderId,
          status: 'PENDING',
          customerId: event.payload.customerId,
          totalAmount: event.payload.totalAmount,
          itemCount: event.payload.items.length,
          createdAt: event.timestamp
        });
        break;

      case 'PaymentConfirmed':
        await this.readDb.orders.update(event.streamId, { status: 'PAID' });
        await this.readDb.revenue.add({ amount: event.payload.amount, date: event.timestamp });
        break;

      case 'OrderShipped':
        await this.readDb.orders.update(event.streamId, {
          status: 'SHIPPED',
          trackingNumber: event.payload.trackingNumber,
          shippedAt: event.payload.shippedAt
        });
        break;
    }
  }
}

// Time-travel debugging — reconstruct state at any point in time
async function getOrderStateAt(orderId, targetDate, eventStore) {
  const allEvents = await eventStore.getEvents(orderId);
  const eventsUpToDate = allEvents.filter(e => new Date(e.timestamp) <= targetDate);
  return Order.fromHistory(eventsUpToDate);
}

// Usage
const pastState = await getOrderStateAt('order_001', new Date('2024-06-01'), eventStore);
console.log(`Order status on June 1st: ${pastState.status}`);
```

### Benefits of Event Sourcing

```
✅ Complete audit trail — every change is recorded, no data loss
✅ Time travel — reconstruct any past state
✅ Event replay — fix bugs by correcting events and replaying
✅ Multiple projections — same events, different read models
✅ Natural integration with CQRS and Saga patterns
✅ Debugging — see exactly what happened and when

⚠️ Trade-offs:
   - Eventual consistency for reads (projections lag behind)
   - More complex than simple CRUD
   - Schema evolution requires careful event versioning
   - Querying current state requires a projection/snapshot
```

---

## 5. Combining Patterns

These patterns are designed to work together. Here's a real-world e-commerce order flow:

```
Customer places order
        │
        ▼
[CQRS — Command Handler]
  Validates PlaceOrderCommand
  Creates Order aggregate
  Appends OrderPlaced event to Event Store ────────────────────────────────────────┐
        │                                                                           │
        ▼                                                                           │
[Saga Orchestrator triggered by OrderPlaced event]                                 │
  Step 1: PaymentService.charge()                                                  │
  Step 2: InventoryService.reserve()                                               │
  Step 3: ShippingService.schedule()                                               │
  (Compensations if any step fails)                                                │
        │                                                                           │
        ▼                                                                           ▼
  Each step emits events                                           [Event Sourcing Projector]
  (PaymentProcessed, StockReserved, etc.)                         Listens to all events
        │                                                          Updates Read DB:
        │                                                          - orderSummaries
        │                                                          - customerStats
        ▼                                                          - inventoryLevels
[CQRS — Query Handler]
  Reads from denormalised Read DB
  Returns fast response to client
```

```javascript
// The complete wiring
class OrderFlowController {
  constructor({ commandHandler, queryHandler, sagaOrchestrator, eventStore }) {
    this.commandHandler = commandHandler;
    this.queryHandler = queryHandler;
    this.sagaOrchestrator = sagaOrchestrator;
    this.eventStore = eventStore;

    // Saga listens to events from event store
    this.eventStore.subscribe('OrderPlaced', (event) => {
      this.sagaOrchestrator.start('OrderFulfillmentSaga', event.payload);
    });
  }

  // Write path: Command → Event Store → Saga
  async placeOrder(orderData) {
    // CQRS Command
    const { orderId } = await this.commandHandler.handle(
      new PlaceOrderCommand(orderData)
    );
    // Saga kicks off asynchronously via event subscription above
    return { orderId, status: 'PROCESSING' };
  }

  // Read path: Query → Read Model (fast, no joins)
  async getOrder(orderId) {
    return this.queryHandler.getOrderDetail(orderId);
  }
}
```

---

## 6. Pattern Comparison & Decision Guide

| Pattern | Consistency | Complexity | Best Use Case |
|---|---|---|---|
| **Saga** | Eventual | Medium | Long-running distributed workflows |
| **2PC** | Strong (ACID) | High | Financial transfers, critical atomicity |
| **CQRS** | Eventual (reads) | Medium | High read/write scale disparity |
| **Event Sourcing** | Eventual | High | Audit trails, time travel, complex domains |

### Decision Flowchart

```
Do you need strong atomicity across multiple services?
├── YES → Use Two-Phase Commit (2PC)
└── NO  → Do you have a long-running multi-step workflow?
          ├── YES → Use Saga Pattern
          │         (Choreography if simple, Orchestration if complex)
          └── NO  → Do reads and writes have very different requirements?
                    ├── YES → Use CQRS
                    └── NO  → Do you need full audit trail / time travel?
                              ├── YES → Use Event Sourcing (+ CQRS)
                              └── NO  → Standard CRUD is fine
```

### Combining Patterns — Common Combos

```
Event Sourcing + CQRS       → Most natural pairing; events drive read model projections
Saga + CQRS                 → Commands trigger saga; saga events update read models
Event Sourcing + Saga        → Saga state persisted as events; full auditability
All four together            → Enterprise-grade, highest complexity, maximum flexibility
```

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
