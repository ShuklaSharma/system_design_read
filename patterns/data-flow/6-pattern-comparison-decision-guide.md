# Pattern Comparison & Decision Guide for AI Applications

## Table of Contents
1. [Overview](#overview)
2. [Pattern Summary Cards](#pattern-summary-cards)
3. [Comparison Tables](#comparison-tables)
4. [Decision Flowchart](#decision-flowchart)
5. [Problem-to-Pattern Mapping](#problem-to-pattern-mapping)
6. [AI System Archetypes](#ai-system-archetypes)
7. [Cost & Complexity Tradeoffs](#cost--complexity-tradeoffs)
8. [Pattern Maturity Model](#pattern-maturity-model)
9. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)

---

## Overview

This guide is the companion to all pattern tutorials in this series. Use it to:
- Quickly identify which pattern(s) address your current problem
- Compare patterns on key dimensions (complexity, consistency, performance impact)
- Choose the right pattern for your system's maturity level
- Plan a realistic pattern adoption roadmap

**Patterns covered in this series:**
1. Bulkhead
2. Retry + Exponential Backoff
3. Circuit Breaker
4. Backpressure
5. Cache-Aside
6. Sharding
7. Saga
8. Two-Phase Commit (2PC)
9. CQRS
10. Event Sourcing

---

## Pattern Summary Cards

### Bulkhead
```
Problem solved:  One service consuming all resources → others fail
Mechanism:       Isolated resource pools per service/tenant
Consistency:     N/A (not a data pattern)
Complexity:      Low
Implementation:  Hours to days
Cost impact:     Prevents emergency GPU/API over-spend
AI use case:     Isolate LLM, image gen, embedding pools
Watch out for:   Pools too small (starvation) or too large (no isolation)
```

### Retry + Exponential Backoff
```
Problem solved:  Transient failures (429, 503, timeouts)
Mechanism:       Re-attempt with increasing delays between tries
Consistency:     N/A (operations must be idempotent)
Complexity:      Low
Implementation:  Hours
Cost impact:     Slightly increases API calls (~5%) — saves 87% support tickets
AI use case:     OpenAI/Anthropic rate limit handling, GPU cold starts
Watch out for:   Retry storms, non-idempotent operations
```

### Circuit Breaker
```
Problem solved:  Slow provider causing cascading timeouts
Mechanism:       Open (fail fast) → Half-open (probe) → Closed (normal)
Consistency:     N/A
Complexity:      Low-Medium
Implementation:  Days
Cost impact:     Prevents runaway retry costs during provider outages
AI use case:     OpenAI/Stability/Anthropic outages, GPU service down
Watch out for:   Threshold tuning (too sensitive = flapping, too lenient = cascades)
```

### Backpressure
```
Problem solved:  Overload causing system crash or OOM
Mechanism:       Reject or queue excess requests at defined limits
Consistency:     N/A
Complexity:      Low-Medium
Implementation:  Days
Cost impact:     Prevents runaway API bills during traffic spikes
AI use case:     Viral traffic spikes, Black Friday, product launches
Watch out for:   Rejection UX (must communicate clearly to users)
```

### Cache-Aside
```
Problem solved:  Repeated expensive API calls, high latency
Mechanism:       Check cache → miss → call API → store result
Consistency:     Eventual (reads may be stale by TTL duration)
Complexity:      Low
Implementation:  Days
Cost impact:     70–90% reduction in LLM API costs
AI use case:     FAQ bots, RAG embeddings, semantic search
Watch out for:   Caching personalized data, cache stampede on miss
```

### Sharding
```
Problem solved:  Single-node data capacity limits
Mechanism:       Partition data across multiple independent nodes
Consistency:     Depends on strategy (strong within shard, eventual across)
Complexity:      High
Implementation:  Weeks
Cost impact:     Linear cost scaling instead of exponential emergency upgrades
AI use case:     200M+ vector embeddings, multi-tenant isolation, large datasets
Watch out for:   Hot spots, cross-shard queries, resharding complexity
```

### Saga
```
Problem solved:  Multi-step pipeline failures leaving inconsistent state
Mechanism:       Compensating transactions to undo completed steps
Consistency:     Eventual (compensations take time)
Complexity:      Medium-High
Implementation:  Weeks
Cost impact:     Eliminates charge-without-delivery, reduces support costs
AI use case:     Payment + AI generation + delivery pipelines
Watch out for:   Non-idempotent compensations, compensation failures (need DLQ)
```

### Two-Phase Commit (2PC)
```
Problem solved:  Atomic updates across multiple data stores
Mechanism:       Prepare (vote) → Commit or Abort (all or nothing)
Consistency:     Strong (atomic, all-or-nothing)
Complexity:      High
Implementation:  Weeks
Cost impact:     Prevents costly reconciliation, regulatory fines
AI use case:     Model deployment atomicity, experiment result commits
Watch out for:   Not for long transactions (> 30s), coordinator is SPOF, blocking
```

### CQRS
```
Problem solved:  Read/write contention, different optimal schemas for reads/writes
Mechanism:       Separate models/databases for commands and queries
Consistency:     Eventual (read model lags write by seconds)
Complexity:      Medium-High
Implementation:  Weeks
Cost impact:     70-80% DB cost reduction, unlocks enterprise sales
AI use case:     AI analytics dashboards, inference logging, experiment tracking
Watch out for:   Eventual consistency surprises, projection maintenance
```

### Event Sourcing
```
Problem solved:  Lost history, inability to reproduce past state, lack of audit trail
Mechanism:       Store events (not current state); derive state by replaying events
Consistency:     Eventual (read models are projections)
Complexity:      Very High
Implementation:  Months
Cost impact:     Reduces audit cost, prevents regulatory fines
AI use case:     ML experiment tracking, FDA reproducibility, compliance AI systems
Watch out for:   Never delete events, schema evolution complexity, snapshot management
```

---

## Comparison Tables

### By Problem Category

| Problem | Primary Pattern | Supporting Pattern |
|---|---|---|
| Transient API failures | Retry + Backoff | Circuit Breaker |
| Provider outage / slow | Circuit Breaker | Cache-Aside (fallback) |
| Traffic spike / overload | Backpressure | Bulkhead |
| Noisy neighbor / tenant isolation | Bulkhead | Sharding |
| Repeated API costs | Cache-Aside | Semantic Cache |
| Data scale (100M+ records) | Sharding | Consistent Hashing |
| Multi-step pipeline failure | Saga | Retry within steps |
| Cross-store atomic update | 2PC | Saga (if long-running) |
| Read/write DB contention | CQRS | Cache-Aside |
| Lost history / audit trail | Event Sourcing | CQRS |
| Model reproducibility | Event Sourcing | 2PC |
| High dashboard query latency | CQRS | Cache-Aside |

### By System Property

| Property | Best Pattern(s) |
|---|---|
| **Availability** | Circuit Breaker, Cache-Aside, Retry |
| **Consistency (strong)** | 2PC |
| **Consistency (eventual)** | Saga, CQRS, Event Sourcing |
| **Scalability (load)** | Bulkhead, Backpressure |
| **Scalability (data)** | Sharding |
| **Observability / Audit** | Event Sourcing |
| **Cost optimization** | Cache-Aside, Sharding |
| **Resilience** | Circuit Breaker + Retry + Bulkhead |
| **Data isolation** | Bulkhead, Sharding |

### By Complexity & Time-to-Value

| Pattern | Complexity | Time to Implement | Time to Value |
|---|---|---|---|
| Retry + Backoff | ⬛ Low | Hours | Immediate |
| Cache-Aside | ⬛ Low | Days | Days |
| Backpressure | ⬛⬛ Low-Med | Days | Days |
| Circuit Breaker | ⬛⬛ Low-Med | Days | Days |
| Bulkhead | ⬛⬛ Low-Med | Days–1 week | Week |
| CQRS (simple) | ⬛⬛⬛ Medium | 2–4 weeks | 1 month |
| Saga | ⬛⬛⬛ Medium | 2–4 weeks | 1 month |
| Sharding | ⬛⬛⬛⬛ High | 3–6 weeks | 1–2 months |
| 2PC | ⬛⬛⬛⬛ High | 2–4 weeks | 1 month |
| Event Sourcing | ⬛⬛⬛⬛⬛ Very High | 2–4 months | 3–6 months |

### By Consistency Guarantee

| Pattern | Consistency Model | Acceptable When... |
|---|---|---|
| 2PC | Strong (ACID) | "Half committed = catastrophic" (financial, medical) |
| Bulkhead | N/A (isolation) | Any scenario |
| Retry | N/A (single op) | Operation is idempotent |
| Circuit Breaker | N/A (availability) | Any scenario |
| Cache-Aside | Eventual (TTL-bounded) | Stale data is acceptable for TTL duration |
| Sharding | Strong within shard | Single-shard queries; eventual across shards |
| Saga | Eventual | "Partial state" can be compensated |
| CQRS | Eventual (seconds) | Dashboard/analytics can be seconds behind |
| Event Sourcing | Eventual (for reads) | History is primary; reads can lag |

---

## Decision Flowchart

```
START: What is your primary pain point?
│
├─► "AI API calls failing" ────────────────────────────────────────────┐
│     │                                                                 │
│     ├─ Failing occasionally (< 5% error rate)?                       │
│     │    └─► Retry + Backoff                                         │
│     │                                                                 │
│     └─ Failing sustained (> 10% error rate, provider down)?          │
│          └─► Circuit Breaker  (+Retry for when it closes)            │
│                                                                       │
├─► "Too expensive / high API costs" ─────────────────────────────────┐│
│     │                                                                ││
│     ├─ Same queries being made repeatedly?                           ││
│     │    └─► Cache-Aside (semantic cache for AI queries)             ││
│     │                                                                ││
│     └─ Too many concurrent requests hitting provider?                ││
│          └─► Bulkhead (limit concurrency) + Backpressure             ││
│                                                                       │
├─► "System crashes under load" ──────────────────────────────────────┤
│     │                                                                 │
│     ├─ All services crash together?                                   │
│     │    └─► Bulkhead (isolate service pools)                        │
│     │                                                                 │
│     └─ System overwhelmed, OOM errors?                               │
│          └─► Backpressure (reject at limits)                         │
│                                                                       │
├─► "One tenant slowing down others" ─────────────────────────────────┤
│     │                                                                 │
│     ├─ Noisy requests (concurrency)?                                  │
│     │    └─► Bulkhead (per-tenant pools)                             │
│     │                                                                 │
│     └─ Noisy data (huge dataset)?                                    │
│          └─► Sharding (tenant-isolated shards)                       │
│                                                                       │
├─► "Database too slow" ──────────────────────────────────────────────┤
│     │                                                                 │
│     ├─ Dashboard queries blocking writes?                             │
│     │    └─► CQRS (separate read/write models)                       │
│     │                                                                 │
│     └─ Data volume exceeding single node?                            │
│          └─► Sharding                                                │
│                                                                       │
├─► "Multi-step pipeline leaving inconsistent state" ─────────────────┤
│     │                                                                 │
│     ├─ Steps span multiple services, long-running?                   │
│     │    └─► Saga (compensating transactions)                        │
│     │                                                                 │
│     └─ Steps span 2–3 data stores you OWN, short-lived?             │
│          └─► 2PC (atomic commit)                                     │
│                                                                       │
├─► "Can't reproduce past model behavior" ────────────────────────────┤
│     │                                                                 │
│     └─► Event Sourcing (+ CQRS for queryable history)               │
│                                                                       │
└─► "Need audit trail / compliance" ──────────────────────────────────┘
      └─► Event Sourcing (immutable event log)
```

---

## Problem-to-Pattern Mapping

### Specific AI Problems

| Specific Symptom | Root Cause | Pattern |
|---|---|---|
| OpenAI 429 errors at peak | Rate limit hit | Retry + Backoff + Bulkhead |
| Image generation slowing chat | Shared resource pool | Bulkhead |
| Same question costs $0.03 every time | No caching | Cache-Aside |
| Dashboard takes 40 seconds to load | Read/write contention | CQRS |
| Customer charged but content not delivered | No saga | Saga |
| Re-deploying model corrupts predictions | Partial deployment | 2PC |
| Can't find why model degraded | No history | Event Sourcing |
| Vector search takes 8 seconds | Too many vectors per node | Sharding |
| Free tier users slow down paying users | No isolation | Bulkhead + Sharding |
| Service dies when OpenAI has an outage | No circuit breaker | Circuit Breaker + Cache |
| 100 concurrent cache misses hit API together | Cache stampede | Lock-based cache |
| Training data is inconsistent after pipeline crash | No saga | Saga |
| "Why did the model make this prediction?" | No audit trail | Event Sourcing |
| Experiment results can't be reproduced | State overwritten | Event Sourcing |

---

## AI System Archetypes

### Archetype 1: AI Chatbot / FAQ Bot

```
Scale: 10K–100K users/day
Key problems: API cost, latency, provider outages

Recommended patterns:
  Priority 1: Cache-Aside (semantic cache) — 70% cost reduction
  Priority 2: Retry + Circuit Breaker — handle OpenAI downtime gracefully
  Priority 3: Bulkhead — isolate different chat features (search vs generation)

Skip for now: Event Sourcing, 2PC, Sharding (overkill at this scale)
```

### Archetype 2: Multi-Tenant AI SaaS Platform

```
Scale: 100–10,000 business customers, varied usage
Key problems: Noisy neighbor, cost control, dashboard performance

Recommended patterns:
  Priority 1: Bulkhead — tenant isolation (enterprise vs. free)
  Priority 2: Backpressure — protect under load
  Priority 3: Cache-Aside — reduce per-tenant API costs
  Priority 4: CQRS — dashboards for tenants, analytics for team
  Priority 5: Sharding — large enterprise tenant data isolation

Add when ready: Saga (payment pipeline), Event Sourcing (audit features)
```

### Archetype 3: AI Model Training Platform / MLOps

```
Scale: 10–500 ML researchers, large datasets
Key problems: Reproducibility, experiment tracking, long job reliability

Recommended patterns:
  Priority 1: Event Sourcing — full experiment audit trail
  Priority 2: CQRS — fast experiment comparison queries
  Priority 3: Saga — complex training pipelines (data prep → train → eval → deploy)
  Priority 4: 2PC — atomic model deployment across registry + router + feature store

Nice to have: Sharding (for very large experiment histories)
```

### Archetype 4: High-Throughput AI Inference API

```
Scale: 1M+ requests/day, SLA-bound
Key problems: Latency, availability, cost at scale

Recommended patterns:
  Priority 1: Circuit Breaker + Retry — provider reliability
  Priority 2: Backpressure — protect under load spikes
  Priority 3: Cache-Aside — reduce repeated inference costs
  Priority 4: Bulkhead — separate premium/standard tiers
  Priority 5: Sharding — distribute vector store

Add later: CQRS (analytics), Event Sourcing (if compliance required)
```

### Archetype 5: Regulated AI System (Finance, Healthcare, Legal)

```
Scale: Moderate (100K–10M requests/day), compliance-heavy
Key problems: Audit trail, reproducibility, consistency, data residency

Recommended patterns:
  Priority 1: Event Sourcing — immutable audit trail (non-negotiable)
  Priority 2: 2PC — atomic updates across compliant stores
  Priority 3: CQRS — compliance reports + operational dashboards
  Priority 4: Circuit Breaker + Retry — availability during audits
  Priority 5: Sharding (geographic) — data residency (GDPR, HIPAA)
  Priority 6: Saga — complex workflows with compensations

Do not skip: Event Sourcing and 2PC are likely regulatory requirements
```

---

## Cost & Complexity Tradeoffs

### ROI by Pattern (from individual guides)

| Pattern | Implementation Cost | Annual Return | ROI |
|---|---|---|---|
| Retry + Backoff | $1K | $600K | 60,000% |
| Cache-Aside | $4.5K | $458K | 10,000% |
| Circuit Breaker | $3K | $400K | 13,000% |
| Bulkhead | $7K | $3M | 42,857% |
| Backpressure | $5K | $200K | 4,000% |
| CQRS | $30K | $855K | 2,850% |
| Saga | $23K | $590K | 2,565% |
| 2PC | $13K | $2.54M | 19,538% |
| Sharding | $45K | $2.4M | 5,233% |
| Event Sourcing | $43K | $450K+ | 1,047%+ |

> Note: ROI figures are illustrative based on mid-size AI SaaS scenarios in individual guides. Your numbers will vary based on scale, incident frequency, and team size.

### Complexity vs. Availability Impact

```
High Impact,                          High Impact,
Low Complexity                        High Complexity
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│                                │                  │
│  Retry + Backoff               │  Bulkhead         │
│  Cache-Aside                   │  Sharding         │
│  Circuit Breaker               │  Saga             │
│                                │  2PC              │
│                                │  CQRS             │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│                                │                  │
│  Backpressure                  │  Event Sourcing   │
│  (simple rejection)            │                  │
│                                │                  │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Low Impact,                           Low Impact,
Low Complexity                         High Complexity
```

---

## Pattern Maturity Model

### Stage 1: Survivable (Month 1–3)

Your system can handle basic production load without catastrophic failures.

```
✅ Must have:
  - Retry + Exponential Backoff (on ALL external AI API calls)
  - Basic error handling (don't crash on 429/503)
  - Health checks and basic alerting

📈 Outcome: Error rate < 1%, service survives provider blips
```

### Stage 2: Reliable (Month 3–6)

Your system degrades gracefully under load and provider issues.

```
✅ Must have:
  - Circuit Breaker (per AI provider)
  - Backpressure (basic rejection under overload)
  - Cache-Aside (for repeated/expensive queries)

📈 Outcome: 99.5% uptime, API costs reduced 50–70%
```

### Stage 3: Scalable (Month 6–12)

Your system serves multiple tenants with different requirements without interference.

```
✅ Must have:
  - Bulkhead (per tenant tier or service type)
  - CQRS (if analytics or dashboards are slow)
  - Saga (if you have multi-step pipelines involving payment)

📈 Outcome: Enterprise customers can be onboarded, SLA met
```

### Stage 4: Compliant & Auditable (Month 12–24)

Your system can pass regulatory audits and reproduce any historical state.

```
✅ Must have:
  - Event Sourcing (full audit trail)
  - 2PC (atomic model deployments)
  - Geographic Sharding (data residency for GDPR/HIPAA)

📈 Outcome: Enterprise and regulated industry deals unlocked
```

### Stage 5: Hyper-Scale (Month 18–36)

Your system scales linearly with data and load growth.

```
✅ Must have:
  - Sharding (vector DB, document store, tenant data)
  - Consistent Hashing (elastic scaling)
  - Full observability across all patterns

📈 Outcome: Serve 10x more customers without re-architecting
```

---

## Quick Reference Cheatsheet

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PATTERN CHEATSHEET — AI APPLICATIONS                                   │
├─────────────────────────┬───────────────────────────────────────────────┤
│  PATTERN                │  USE WHEN                                     │
├─────────────────────────┼───────────────────────────────────────────────┤
│  Retry + Backoff        │  Occasional 429/503 from AI APIs              │
│  Circuit Breaker        │  Provider down, cascading timeouts            │
│  Backpressure           │  Traffic spikes, OOM risk, cost explosions    │
│  Bulkhead               │  Tenants interfering with each other          │
│  Cache-Aside            │  Repeated queries, high API costs, latency    │
│  Sharding               │  Data too large for one node (100M+ records)  │
│  Saga                   │  Multi-step pipeline leaves inconsistent state│
│  2PC                    │  Atomic update across 2–4 owned data stores   │
│  CQRS                   │  Read/write DB contention, slow dashboards    │
│  Event Sourcing         │  Audit trail, reproducibility, compliance     │
├─────────────────────────┼───────────────────────────────────────────────┤
│  NEVER DO               │  WHY                                          │
├─────────────────────────┼───────────────────────────────────────────────┤
│  Retry without idempotency│  Double charges, duplicate records          │
│  2PC for > 30s ops      │  Blocks everything, defeats the purpose       │
│  Cache personalized data│  User A gets User B's response               │
│  ES without CQRS        │  Replaying 100K events per read = unusable   │
│  Saga without DLQ       │  Silent compensation failures = money lost    │
│  Sharding by timestamp  │  All writes hit one "latest" shard (hot spot)│
│  All patterns on day 1  │  Over-engineering kills startups              │
├─────────────────────────┼───────────────────────────────────────────────┤
│  ADOPTION ORDER         │  WHY THIS ORDER                               │
├─────────────────────────┼───────────────────────────────────────────────┤
│  1. Retry + CB          │  Biggest safety ROI, cheapest to add         │
│  2. Cache-Aside         │  Biggest cost ROI, easy to add               │
│  3. Backpressure        │  Protect from overload as you grow           │
│  4. Bulkhead            │  Add when tenants start interfering           │
│  5. CQRS                │  Add when DB contention appears              │
│  6. Saga                │  Add when payment + generation fails together │
│  7. Event Sourcing      │  Add for compliance or reproducibility needs  │
│  8. 2PC                 │  Add for atomic cross-store deployments       │
│  9. Sharding            │  Add when data volume forces it              │
└─────────────────────────┴───────────────────────────────────────────────┘
```

---

## Series Index

| Guide | Focus | Key Patterns |
|---|---|---|
| [Bulkhead Pattern Guide](bulkhead-pattern-guide.md) | Resource isolation | Thread pools, GPU pools, API quota pools |
| [Retry + Backoff Guide](retry-backoff-ai-guide.md) | Transient failure recovery | Exponential, jitter, linear backoff |
| [Circuit Breaker Guide](circuit-breaker-pattern-guide.md) | Cascading failure prevention | States, thresholds, recovery |
| [Backpressure Guide](backpressure-pattern-guide.md) | Overload protection | Queue limits, rejection, adaptive |
| [Cache-Aside Guide](cache-aside-pattern-guide.md) | Cost & latency reduction | Semantic cache, TTL, invalidation |
| [Sharding Guide](sharding-pattern-guide.md) | Data scale | Hash, range, consistent hashing |
| [Saga Guide](1-saga-pattern-guide.md) | Distributed transaction recovery | Choreography, orchestration |
| [2PC Guide](2-two-phase-commit-guide.md) | Atomic consistency | Prepare, commit, coordinator recovery |
| [CQRS Guide](3-cqrs-pattern-guide.md) | Read/write separation | Commands, queries, projections |
| [Event Sourcing Guide](4-event-sourcing-pattern-guide.md) | Audit & reproducibility | Events, aggregates, snapshots |
| [Combining Patterns Guide](5-combining-patterns-guide.md) | Full architecture | Reference stack, anti-patterns |
| **This Guide** | Decision support | Comparison tables, flowcharts |

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
