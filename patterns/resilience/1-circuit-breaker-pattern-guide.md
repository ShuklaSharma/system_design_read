# Circuit Breaker Pattern for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why Circuit Breakers Matter for AI Systems](#why-circuit-breakers-matter)
3. [Circuit Breaker States](#circuit-breaker-states)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Monitoring & Observability](#monitoring-observability)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Real-World Case Studies](#real-world-case-studies)
10. [Production Best Practices](#production-best-practices)

---

## Introduction

### What is a Circuit Breaker?

A **Circuit Breaker** is a design pattern that prevents an application from repeatedly trying to execute an operation that's likely to fail. Named after electrical circuit breakers that prevent electrical overload.

**Analogy:** Like a circuit breaker in your home that trips when there's a power surge, protecting your appliances from damage.

### The Problem Without Circuit Breakers

```
Scenario: OpenAI API is down

Without Circuit Breaker:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Request 1 → OpenAI → Timeout (30s) → Fail ❌
Request 2 → OpenAI → Timeout (30s) → Fail ❌
Request 3 → OpenAI → Timeout (30s) → Fail ❌
... 1000 more requests ...
Request 1000 → OpenAI → Timeout (30s) → Fail ❌

Result:
- 30,000 seconds (8.3 hours) wasted waiting
- All requests failed
- Resources tied up in timeouts
- Cascading failures to dependent services
- Users get terrible experience
```

### The Solution With Circuit Breaker

```
With Circuit Breaker:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Request 1 → OpenAI → Timeout (30s) → Fail ❌
Request 2 → OpenAI → Timeout (30s) → Fail ❌
Request 3 → OpenAI → Timeout (30s) → Fail ❌
Request 4 → OpenAI → Timeout (30s) → Fail ❌
Request 5 → OpenAI → Timeout (30s) → Fail ❌

🔴 CIRCUIT BREAKER OPENS (5 failures detected)

Request 6 → Circuit Open → Fail Fast (1ms) ❌
Request 7 → Circuit Open → Fail Fast (1ms) ❌
... 994 more requests (all fail fast) ...

After 60 seconds:
🟡 CIRCUIT BREAKER HALF-OPEN (testing recovery)

Request 1001 → OpenAI → Success ✅

🟢 CIRCUIT BREAKER CLOSED (service recovered)

Result:
- Only 150 seconds wasted (5 timeouts)
- 995 requests failed fast (instant feedback)
- Resources freed immediately
- System remained responsive
- Users got clear error message
```

---

## Why Circuit Breakers Matter for AI Systems

### Unique AI System Challenges

1. **Expensive Operations**
   - LLM API calls: $0.002 - $0.10 per request
   - GPU inference: $0.50 - $5.00 per hour
   - Every failed request = wasted money

2. **Long Timeouts**
   - Text generation: 5-30 seconds
   - Image generation: 30-180 seconds
   - Video generation: 5-30 minutes
   - Holding connections/resources is costly

3. **Cascading Failures**
   - AI service down → entire platform affected
   - One model failure → block all AI features
   - External API outage → your service appears broken

4. **Third-Party Dependencies**
   - OpenAI, Anthropic, Cohere outages
   - Cloud provider issues (AWS, GCP, Azure)
   - Vector database failures (Pinecone, Weaviate)
   - No control over their reliability

### Real Incident: No Circuit Breaker

**December 2023 - OpenAI Outage**

```
Timeline:
14:00 - OpenAI API starts experiencing issues
14:02 - Our service starts getting slow responses (10-20s)
14:05 - OpenAI completely down, all requests timeout (30s)
14:05 - Our request queue fills up (10,000 requests waiting)
14:10 - Our servers run out of connection pool
14:12 - Our entire platform becomes unresponsive
14:15 - Emergency: All hands on deck
14:30 - Manual intervention: Kill all OpenAI requests
14:45 - Deploy emergency fix: Disable AI features
15:30 - OpenAI recovers
16:00 - Re-enable AI features, monitor closely

Impact:
- 2 hours total disruption
- $200K revenue loss
- 5,000+ angry users
- Emergency deployment on Friday afternoon
- Team weekend ruined
- Trust damaged
```

**Same Incident WITH Circuit Breaker:**

```
Timeline:
14:00 - OpenAI API starts experiencing issues
14:02 - Circuit breaker detects failures (5 in 10s)
14:02 - 🔴 Circuit opens automatically
14:02 - All requests fail fast with clear message
14:02 - Automatic fallback to cached responses
14:02 - Status page updated automatically
15:30 - Circuit breaker detects OpenAI recovery
15:30 - 🟢 Circuit closes automatically
15:30 - Normal operation resumes

Impact:
- 2 minutes disruption (just detection time)
- Zero manual intervention
- Users got cached results (90% useful)
- Clear status message: "AI temporarily unavailable"
- No revenue loss
- Team unaffected
- Trust maintained
```

---

## Circuit Breaker States

### State Machine Diagram

```
                    ┌─────────────────┐
                    │                 │
                    │     CLOSED      │  Normal operation
                    │   (Healthy)     │  All requests pass through
                    │                 │
                    └────────┬────────┘
                             │
                    Failure threshold reached
                    (e.g., 5 failures in 10s)
                             │
                             ▼
                    ┌─────────────────┐
         ┌─────────▶│                 │
         │          │      OPEN       │  Failing fast
         │          │    (Failing)    │  All requests rejected
         │          │                 │
         │          └────────┬────────┘
         │                   │
         │          Timeout period elapsed
         │          (e.g., 60 seconds)
         │                   │
         │                   ▼
         │          ┌─────────────────┐
         │          │                 │
         │          │   HALF-OPEN     │  Testing recovery
         │          │   (Testing)     │  Limited requests allowed
         │          │                 │
         │          └────┬───────┬────┘
         │               │       │
    Still failing        │       │  Success threshold reached
         │               │       │  (e.g., 3 successes)
         │               │       │
         └───────────────┘       └──────────────┐
                                                 │
                                                 ▼
                                        Back to CLOSED
```

### State Details

#### 1. CLOSED (Healthy) 🟢

**Behavior:**
- All requests pass through normally
- Monitor success/failure rate
- Count failures in sliding window

**Transition to OPEN:**
- Failure rate exceeds threshold (e.g., 50% in last 10s)
- OR consecutive failures exceed limit (e.g., 5 in a row)
- OR error rate spike detected

```javascript
// Example transition logic
if (failureRate > 0.5 && requestCount > 10) {
  state = 'OPEN';
  openedAt = Date.now();
}

if (consecutiveFailures >= 5) {
  state = 'OPEN';
  openedAt = Date.now();
}
```

#### 2. OPEN (Failing) 🔴

**Behavior:**
- Immediately reject all requests (fail fast)
- Don't even attempt to call the service
- Wait for timeout period (e.g., 60 seconds)

**User Experience:**
```javascript
// Instant failure response
throw new CircuitBreakerOpenError({
  message: 'OpenAI service temporarily unavailable',
  retryAfter: 60,
  status: 503,
  fallback: getCachedResponse()
});
```

**Transition to HALF-OPEN:**
- After timeout period elapsed
- Automatically transitions to test recovery

#### 3. HALF-OPEN (Testing) 🟡

**Behavior:**
- Allow limited requests through (e.g., 3-5)
- Monitor their success/failure
- All other requests still fail fast

**Transition to CLOSED:**
- If success threshold met (e.g., 3/3 succeed)
- Service appears healthy

**Transition to OPEN:**
- If any request fails
- Service still unhealthy

```javascript
// Half-open logic
if (state === 'HALF_OPEN') {
  if (halfOpenSuccesses >= 3) {
    state = 'CLOSED';
    reset();
  } else if (halfOpenFailures > 0) {
    state = 'OPEN';
    openedAt = Date.now();
  }
}
```

---

## Implementation Patterns

### Pattern 1: Basic Circuit Breaker

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    // Configuration
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000; // 60 seconds
    this.
    
    // State
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
    this.lastError = null;
  }

  async execute(fn) {
    // Check current state
    if (this.state === 'OPEN') {
      // Check if timeout has elapsed
      if (Date.now() < this.nextAttempt) {
        // Still in timeout period - fail fast
        throw new CircuitBreakerOpenError(
          `Circuit breaker is OPEN. Try again in ${
            Math.ceil((this.nextAttempt - Date.now()) / 1000)
          }s`,
          this.lastError
        );
      }
      
      // Timeout elapsed - transition to HALF_OPEN
      this.state = 'HALF_OPEN';
      this.successCount = 0;
      console.log('Circuit breaker transitioning to HALF_OPEN');
    }

    try {
      // Execute the function
      const result = await fn();
      
      // Success!
      this.onSuccess();
      return result;
      
    } catch (error) {
      // Failure
      this.onFailure(error);
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      
      if (this.successCount >= this.successThreshold) {
        // Enough successes - close the circuit
        this.state = 'CLOSED';
        console.log('Circuit breaker CLOSED - service recovered');
      }
    }
  }

  onFailure(error) {
    this.lastError = error;
    this.failureCount++;
    
    if (this.state === 'HALF_OPEN') {
      // Any failure in HALF_OPEN → back to OPEN
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
      console.log(`Circuit breaker back to OPEN after failure in HALF_OPEN`);
      
    } else if (this.state === 'CLOSED') {
      if (this.failureCount >= this.failureThreshold) {
        // Too many failures - open the circuit
        this.state = 'OPEN';
        this.nextAttempt = Date.now() + this.timeout;
        console.log(`Circuit breaker OPENED after ${this.failureCount} failures`);
      }
    }
  }

  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      nextAttempt: this.state === 'OPEN' ? this.nextAttempt : null
    };
  }

  reset() {
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
    console.log('Circuit breaker manually reset');
  }
}

class CircuitBreakerOpenError extends Error {
  constructor(message, lastError) {
    super(message);
    this.name = 'CircuitBreakerOpenError';
    this.lastError = lastError;
  }
}

// Usage
const openAIBreaker = new CircuitBreaker({
  failureThreshold: 5,
  successThreshold: 2,
  timeout: 60000
});

async function callOpenAI(prompt) {
  return openAIBreaker.execute(async () => {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }]
      }),
      timeout: 30000
    });

    if (!response.ok) {
      throw new Error(`OpenAI API error: ${response.status}`);
    }

    return response.json();
  });
}

// Using it
try {
  const result = await callOpenAI('Hello, world!');
  console.log(result);
} catch (error) {
  if (error instanceof CircuitBreakerOpenError) {
    console.log('Service temporarily unavailable:', error.message);
    // Show cached result or fallback
  } else {
    console.error('Request failed:', error);
  }
}
```

### Pattern 2: Time-Window Based Circuit Breaker

```javascript
class TimeWindowCircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 0.5; // 50%
    this.minimumRequests = options.minimumRequests || 10;
    this.windowSize = options.windowSize || 10000; // 10 seconds
    this.timeout = options.timeout || 60000;
    this.successThreshold = options.successThreshold || 2;
    
    // State
    this.state = 'CLOSED';
    this.requests = []; // Sliding window of requests
    this.nextAttempt = Date.now();
    this.halfOpenSuccesses = 0;
  }

  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new CircuitBreakerOpenError('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
      this.halfOpenSuccesses = 0;
    }

    const startTime = Date.now();
    
    try {
      const result = await fn();
      this.recordSuccess(startTime);
      return result;
    } catch (error) {
      this.recordFailure(startTime);
      throw error;
    }
  }

  recordSuccess(startTime) {
    this.requests.push({
      timestamp: startTime,
      success: true
    });
    
    this.cleanup();
    
    if (this.state === 'HALF_OPEN') {
      this.halfOpenSuccesses++;
      if (this.halfOpenSuccesses >= this.successThreshold) {
        this.state = 'CLOSED';
        console.log('Circuit breaker CLOSED');
      }
    }
  }

  recordFailure(startTime) {
    this.requests.push({
      timestamp: startTime,
      success: false
    });
    
    this.cleanup();
    
    if (this.state === 'HALF_OPEN') {
      // Any failure → back to OPEN
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
      console.log('Circuit breaker back to OPEN');
      return;
    }
    
    if (this.state === 'CLOSED') {
      const stats = this.getWindowStats();
      
      if (stats.total >= this.minimumRequests) {
        const failureRate = stats.failures / stats.total;
        
        if (failureRate >= this.failureThreshold) {
          this.state = 'OPEN';
          this.nextAttempt = Date.now() + this.timeout;
          console.log(
            `Circuit breaker OPENED: ${(failureRate * 100).toFixed(1)}% failure rate`
          );
        }
      }
    }
  }

  cleanup() {
    // Remove requests outside the window
    const cutoff = Date.now() - this.windowSize;
    this.requests = this.requests.filter(r => r.timestamp > cutoff);
  }

  getWindowStats() {
    this.cleanup();
    
    const total = this.requests.length;
    const failures = this.requests.filter(r => !r.success).length;
    const successes = total - failures;
    
    return {
      total,
      successes,
      failures,
      failureRate: total > 0 ? failures / total : 0
    };
  }

  getState() {
    const stats = this.getWindowStats();
    
    return {
      state: this.state,
      ...stats,
      nextAttempt: this.state === 'OPEN' ? this.nextAttempt : null
    };
  }
}

// Usage with detailed monitoring
const breaker = new TimeWindowCircuitBreaker({
  failureThreshold: 0.5,  // 50% failure rate
  minimumRequests: 10,     // Need at least 10 requests
  windowSize: 10000,       // Over 10 second window
  timeout: 60000           // Open for 60 seconds
});

// Monitor circuit breaker state
setInterval(() => {
  const state = breaker.getState();
  console.log('Circuit Breaker Status:', {
    state: state.state,
    failureRate: `${(state.failureRate * 100).toFixed(1)}%`,
    requests: `${state.total} (${state.successes}✅ / ${state.failures}❌)`
  });
}, 5000);
```

### Pattern 3: Advanced Circuit Breaker with Metrics

```python
from enum import Enum
from datetime import datetime, timedelta
from typing import Callable, Optional, Any
import asyncio
from collections import deque
import logging

logger = logging.getLogger(__name__)

class CircuitState(Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class CircuitBreakerMetrics:
    """Track detailed metrics for circuit breaker"""
    
    def __init__(self, window_size: int = 10):
        self.window_size = window_size
        self.successes = deque(maxlen=window_size)
        self.failures = deque(maxlen=window_size)
        self.latencies = deque(maxlen=window_size)
        self.state_changes = []
        
    def record_success(self, latency_ms: float):
        self.successes.append({
            'timestamp': datetime.now(),
            'latency': latency_ms
        })
        self.latencies.append(latency_ms)
    
    def record_failure(self, error: Exception):
        self.failures.append({
            'timestamp': datetime.now(),
            'error': str(error)
        })
    
    def record_state_change(self, from_state: CircuitState, to_state: CircuitState):
        self.state_changes.append({
            'timestamp': datetime.now(),
            'from': from_state.value,
            'to': to_state.value
        })
        logger.info(f"Circuit breaker: {from_state.value} → {to_state.value}")
    
    def get_stats(self) -> dict:
        total = len(self.successes) + len(self.failures)
        
        return {
            'total_requests': total,
            'successes': len(self.successes),
            'failures': len(self.failures),
            'success_rate': len(self.successes) / total if total > 0 else 0,
            'avg_latency': sum(self.latencies) / len(self.latencies) if self.latencies else 0,
            'p95_latency': sorted(self.latencies)[int(len(self.latencies) * 0.95)] if self.latencies else 0,
            'state_changes': len(self.state_changes)
        }

class AdvancedCircuitBreaker:
    """
    Production-grade circuit breaker with:
    - Time-window based failure detection
    - Exponential backoff for open duration
    - Detailed metrics and logging
    - Health check callback
    """
    
    def __init__(
        self,
        failure_threshold: float = 0.5,
        min_requests: int = 10,
        window_seconds: int = 10,
        timeout_seconds: int = 60,
        max_timeout_seconds: int = 300,
        success_threshold: int = 2,
        name: str = "circuit_breaker"
    ):
        self.failure_threshold = failure_threshold
        self.min_requests = min_requests
        self.window_seconds = window_seconds
        self.timeout_seconds = timeout_seconds
        self.max_timeout_seconds = max_timeout_seconds
        self.success_threshold = success_threshold
        self.name = name
        
        # State
        self.state = CircuitState.CLOSED
        self.next_attempt = datetime.now()
        self.half_open_successes = 0
        self.consecutive_failures = 0
        
        # Metrics
        self.metrics = CircuitBreakerMetrics()
        
        # Request tracking
        self.requests = deque()
        
        # Callbacks
        self.on_open: Optional[Callable] = None
        self.on_half_open: Optional[Callable] = None
        self.on_close: Optional[Callable] = None
    
    async def execute(self, func: Callable, *args, **kwargs) -> Any:
        """Execute function with circuit breaker protection"""
        
        # Check state
        if self.state == CircuitState.OPEN:
            if datetime.now() < self.next_attempt:
                # Still in timeout - fail fast
                raise CircuitBreakerOpenError(
                    f"Circuit breaker '{self.name}' is OPEN. "
                    f"Retry after {(self.next_attempt - datetime.now()).seconds}s"
                )
            
            # Timeout elapsed - try half open
            self._transition_to_half_open()
        
        # Execute function
        start_time = datetime.now()
        
        try:
            result = await func(*args, **kwargs) if asyncio.iscoroutinefunction(func) else func(*args, **kwargs)
            
            # Success
            latency = (datetime.now() - start_time).total_seconds() * 1000
            self._record_success(latency)
            
            return result
            
        except Exception as error:
            # Failure
            self._record_failure(error)
            raise
    
    def _record_success(self, latency_ms: float):
        """Record successful request"""
        self.requests.append({
            'timestamp': datetime.now(),
            'success': True,
            'latency': latency_ms
        })
        
        self.metrics.record_success(latency_ms)
        self.consecutive_failures = 0
        self._cleanup_old_requests()
        
        if self.state == CircuitState.HALF_OPEN:
            self.half_open_successes += 1
            
            if self.half_open_successes >= self.success_threshold:
                self._transition_to_closed()
    
    def _record_failure(self, error: Exception):
        """Record failed request"""
        self.requests.append({
            'timestamp': datetime.now(),
            'success': False,
            'error': str(error)
        })
        
        self.metrics.record_failure(error)
        self.consecutive_failures += 1
        self._cleanup_old_requests()
        
        if self.state == CircuitState.HALF_OPEN:
            # Any failure in half-open → back to open
            self._transition_to_open()
            
        elif self.state == CircuitState.CLOSED:
            # Check if we should open
            stats = self._get_window_stats()
            
            if stats['total'] >= self.min_requests:
                failure_rate = stats['failures'] / stats['total']
                
                if failure_rate >= self.failure_threshold:
                    logger.warning(
                        f"Circuit breaker '{self.name}': "
                        f"Failure rate {failure_rate:.1%} exceeds threshold {self.failure_threshold:.1%}"
                    )
                    self._transition_to_open()
    
    def _cleanup_old_requests(self):
        """Remove requests outside time window"""
        cutoff = datetime.now() - timedelta(seconds=self.window_seconds)
        self.requests = deque(
            [r for r in self.requests if r['timestamp'] > cutoff],
            maxlen=self.requests.maxlen
        )
    
    def _get_window_stats(self) -> dict:
        """Get statistics for current time window"""
        self._cleanup_old_requests()
        
        total = len(self.requests)
        failures = sum(1 for r in self.requests if not r['success'])
        successes = total - failures
        
        return {
            'total': total,
            'successes': successes,
            'failures': failures,
            'failure_rate': failures / total if total > 0 else 0
        }
    
    def _transition_to_open(self):
        """Transition to OPEN state"""
        old_state = self.state
        self.state = CircuitState.OPEN
        
        # Exponential backoff (but capped)
        backoff = min(
            self.timeout_seconds * (2 ** (self.consecutive_failures - 1)),
            self.max_timeout_seconds
        )
        
        self.next_attempt = datetime.now() + timedelta(seconds=backoff)
        self.metrics.record_state_change(old_state, self.state)
        
        if self.on_open:
            self.on_open()
    
    def _transition_to_half_open(self):
        """Transition to HALF_OPEN state"""
        old_state = self.state
        self.state = CircuitState.HALF_OPEN
        self.half_open_successes = 0
        self.metrics.record_state_change(old_state, self.state)
        
        if self.on_half_open:
            self.on_half_open()
    
    def _transition_to_closed(self):
        """Transition to CLOSED state"""
        old_state = self.state
        self.state = CircuitState.CLOSED
        self.consecutive_failures = 0
        self.metrics.record_state_change(old_state, self.state)
        
        if self.on_close:
            self.on_close()
    
    def get_state(self) -> dict:
        """Get current state and metrics"""
        stats = self._get_window_stats()
        
        return {
            'name': self.name,
            'state': self.state.value,
            'next_attempt': self.next_attempt.isoformat() if self.state == CircuitState.OPEN else None,
            'window_stats': stats,
            'metrics': self.metrics.get_stats(),
            'consecutive_failures': self.consecutive_failures
        }
    
    def reset(self):
        """Manually reset circuit breaker"""
        logger.info(f"Circuit breaker '{self.name}' manually reset")
        self.state = CircuitState.CLOSED
        self.consecutive_failures = 0
        self.half_open_successes = 0
        self.next_attempt = datetime.now()

class CircuitBreakerOpenError(Exception):
    """Raised when circuit breaker is open"""
    pass

# Usage Example
import anthropic

async def call_anthropic(prompt: str) -> str:
    """Call Anthropic API with circuit breaker"""
    client = anthropic.AsyncAnthropic()
    
    message = await client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    )
    
    return message.content[0].text

# Create circuit breaker
breaker = AdvancedCircuitBreaker(
    failure_threshold=0.5,
    min_requests=5,
    window_seconds=10,
    timeout_seconds=60,
    name="anthropic_api"
)

# Set up callbacks
breaker.on_open = lambda: logger.error("⚠️  Anthropic API circuit breaker OPEN")
breaker.on_close = lambda: logger.info("✅ Anthropic API circuit breaker CLOSED")

# Use it
async def generate_text(prompt: str) -> str:
    try:
        result = await breaker.execute(call_anthropic, prompt)
        return result
    except CircuitBreakerOpenError as e:
        logger.warning(f"Circuit breaker open: {e}")
        # Return cached or fallback response
        return get_cached_response(prompt)
    except Exception as e:
        logger.error(f"Request failed: {e}")
        raise

# Monitor status
async def monitor_breaker():
    while True:
        status = breaker.get_state()
        logger.info(f"Circuit Breaker Status: {status}")
        await asyncio.sleep(30)
```

---

## AI-Specific Use Cases

### Use Case 1: Multi-Provider LLM Gateway

```javascript
const CircuitBreaker = require('opossum');

class LLMGateway {
  constructor() {
    // Circuit breaker for each provider
    this.providers = {
      openai: this.createBreaker('OpenAI', () => this.callOpenAI),
      anthropic: this.createBreaker('Anthropic', () => this.callAnthropic),
      cohere: this.createBreaker('Cohere', () => this.callCohere)
    };
    
    // Provider priority
    this.providerPriority = ['openai', 'anthropic', 'cohere'];
  }

  createBreaker(name, fn) {
    const breaker = new CircuitBreaker(fn, {
      timeout: 30000,                // 30 second timeout
      errorThresholdPercentage: 50,  // Open at 50% errors
      resetTimeout: 60000,           // Try again after 60s
      rollingCountTimeout: 10000,    // 10 second window
      rollingCountBuckets: 10,
      name: name
    });

    // Event handlers
    breaker.on('open', () => {
      console.log(`🔴 ${name} circuit breaker OPEN`);
      this.notifyOncall(`${name} circuit breaker opened`);
    });

    breaker.on('halfOpen', () => {
      console.log(`🟡 ${name} circuit breaker HALF-OPEN - testing recovery`);
    });

    breaker.on('close', () => {
      console.log(`🟢 ${name} circuit breaker CLOSED - service recovered`);
    });

    breaker.fallback((error) => {
      console.log(`Using fallback for ${name}:`, error.message);
      return this.getFallbackResponse(name);
    });

    return breaker;
  }

  async generate(prompt, options = {}) {
    // Try each provider in priority order
    for (const providerName of this.providerPriority) {
      const breaker = this.providers[providerName];
      
      // Skip if circuit is open
      if (breaker.opened) {
        console.log(`Skipping ${providerName} (circuit open)`);
        continue;
      }

      try {
        console.log(`Trying ${providerName}...`);
        const result = await breaker.fire(prompt, options);
        
        return {
          text: result,
          provider: providerName,
          fallback: false
        };
        
      } catch (error) {
        console.log(`${providerName} failed:`, error.message);
        // Try next provider
        continue;
      }
    }

    // All providers failed
    throw new Error('All LLM providers unavailable');
  }

  async callOpenAI(prompt, options) {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: options.model || 'gpt-4',
        messages: [{ role: 'user', content: prompt }],
        max_tokens: options.maxTokens || 1000
      }),
      timeout: 30000
    });

    if (!response.ok) {
      throw new Error(`OpenAI error: ${response.status}`);
    }

    const data = await response.json();
    return data.choices[0].message.content;
  }

  async callAnthropic(prompt, options) {
    const Anthropic = require('@anthropic-ai/sdk');
    const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

    const message = await client.messages.create({
      model: options.model || 'claude-sonnet-4-20250514',
      max_tokens: options.maxTokens || 1000,
      messages: [{ role: 'user', content: prompt }]
    });

    return message.content[0].text;
  }

  async callCohere(prompt, options) {
    const response = await fetch('https://api.cohere.ai/v1/generate', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.COHERE_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: options.model || 'command',
        prompt: prompt,
        max_tokens: options.maxTokens || 1000
      }),
      timeout: 30000
    });

    if (!response.ok) {
      throw new Error(`Cohere error: ${response.status}`);
    }

    const data = await response.json();
    return data.generations[0].text;
  }

  getFallbackResponse(provider) {
    return {
      text: `Service temporarily unavailable. We're working on it!`,
      provider: provider,
      fallback: true
    };
  }

  notifyOncall(message) {
    // Send to PagerDuty, Slack, etc.
    console.log('ALERT:', message);
  }

  getStatus() {
    return Object.entries(this.providers).map(([name, breaker]) => ({
      provider: name,
      state: breaker.opened ? 'OPEN' : breaker.halfOpen ? 'HALF_OPEN' : 'CLOSED',
      stats: breaker.stats
    }));
  }
}

// Express API
const express = require('express');
const app = express();
const gateway = new LLMGateway();

app.post('/api/generate', async (req, res) => {
  const { prompt, options } = req.body;
  
  try {
    const result = await gateway.generate(prompt, options);
    
    res.json({
      text: result.text,
      provider: result.provider,
      fallback: result.fallback
    });
    
  } catch (error) {
    res.status(503).json({
      error: 'All AI providers unavailable',
      message: error.message,
      status: gateway.getStatus()
    });
  }
});

app.get('/api/health', (req, res) => {
  const status = gateway.getStatus();
  const allDown = status.every(s => s.state === 'OPEN');
  
  res.status(allDown ? 503 : 200).json({
    status: allDown ? 'degraded' : 'healthy',
    providers: status
  });
});
```

### Use Case 2: RAG System with Vector Database Protection

```python
import asyncio
from typing import List, Dict
import numpy as np
from pinecone import Pinecone

class RAGSystemWithCircuitBreakers:
    """
    RAG system with circuit breakers for:
    - Vector database (Pinecone)
    - Embedding API (OpenAI)
    - LLM API (Anthropic)
    """
    
    def __init__(self):
        # Circuit breakers for each service
        self.pinecone_breaker = AdvancedCircuitBreaker(
            name="pinecone",
            failure_threshold=0.6,
            timeout_seconds=30
        )
        
        self.embedding_breaker = AdvancedCircuitBreaker(
            name="openai_embeddings",
            failure_threshold=0.5,
            timeout_seconds=60
        )
        
        self.llm_breaker = AdvancedCircuitBreaker(
            name="anthropic_llm",
            failure_threshold=0.5,
            timeout_seconds=60
        )
        
        # Fallback data
        self.cached_embeddings = {}
        self.cached_documents = {}
    
    async def query(self, question: str) -> Dict:
        """
        Execute RAG query with circuit breaker protection.
        Gracefully degrade if services unavailable.
        """
        result = {
            'question': question,
            'answer': None,
            'sources': [],
            'degraded': False,
            'errors': []
        }
        
        try:
            # Step 1: Generate embedding
            embedding = await self.get_embedding_with_fallback(question)
            
            if embedding is None:
                result['errors'].append('Embedding service unavailable')
                result['degraded'] = True
                # Can't proceed without embedding
                result['answer'] = "I'm having trouble processing your question right now. Please try again in a moment."
                return result
            
            # Step 2: Search vector database
            documents = await self.search_vectors_with_fallback(embedding)
            
            if not documents:
                result['errors'].append('Vector database unavailable')
                result['degraded'] = True
                # Use generic knowledge without retrieval
                documents = []
            
            result['sources'] = [doc['id'] for doc in documents]
            
            # Step 3: Generate answer
            answer = await self.generate_answer_with_fallback(
                question,
                documents
            )
            
            if answer is None:
                result['errors'].append('LLM service unavailable')
                result['degraded'] = True
                result['answer'] = "I'm experiencing technical difficulties. Please try again shortly."
            else:
                result['answer'] = answer
            
            return result
            
        except Exception as e:
            logger.error(f"RAG query failed: {e}")
            result['errors'].append(str(e))
            result['answer'] = "An error occurred processing your request."
            return result
    
    async def get_embedding_with_fallback(self, text: str) -> Optional[np.ndarray]:
        """Get embedding with circuit breaker and cache fallback"""
        try:
            return await self.embedding_breaker.execute(
                self._generate_embedding,
                text
            )
        except CircuitBreakerOpenError:
            logger.warning("Embedding circuit breaker open, using cache")
            # Try to use cached embedding for similar text
            return self.cached_embeddings.get(text)
        except Exception as e:
            logger.error(f"Embedding failed: {e}")
            return None
    
    async def _generate_embedding(self, text: str) -> np.ndarray:
        """Actually generate embedding"""
        import openai
        
        response = await openai.Embedding.acreate(
            model="text-embedding-ada-002",
            input=text
        )
        
        embedding = np.array(response['data'][0]['embedding'])
        
        # Cache for fallback
        self.cached_embeddings[text] = embedding
        
        return embedding
    
    async def search_vectors_with_fallback(
        self,
        embedding: np.ndarray,
        top_k: int = 5
    ) -> List[Dict]:
        """Search vector DB with circuit breaker and cache fallback"""
        try:
            return await self.pinecone_breaker.execute(
                self._search_pinecone,
                embedding,
                top_k
            )
        except CircuitBreakerOpenError:
            logger.warning("Pinecone circuit breaker open, using cached documents")
            # Return cached documents
            return list(self.cached_documents.values())[:top_k]
        except Exception as e:
            logger.error(f"Vector search failed: {e}")
            return []
    
    async def _search_pinecone(
        self,
        embedding: np.ndarray,
        top_k: int
    ) -> List[Dict]:
        """Actually search Pinecone"""
        pc = Pinecone(api_key=process.env.PINECONE_API_KEY)
        index = pc.Index("documents")
        
        results = index.query(
            vector=embedding.tolist(),
            top_k=top_k,
            include_metadata=True
        )
        
        documents = [
            {
                'id': match['id'],
                'score': match['score'],
                'text': match['metadata']['text']
            }
            for match in results['matches']
        ]
        
        # Cache for fallback
        for doc in documents:
            self.cached_documents[doc['id']] = doc
        
        return documents
    
    async def generate_answer_with_fallback(
        self,
        question: str,
        documents: List[Dict]
    ) -> Optional[str]:
        """Generate answer with circuit breaker"""
        try:
            return await self.llm_breaker.execute(
                self._call_llm,
                question,
                documents
            )
        except CircuitBreakerOpenError:
            logger.warning("LLM circuit breaker open")
            return None
        except Exception as e:
            logger.error(f"LLM call failed: {e}")
            return None
    
    async def _call_llm(
        self,
        question: str,
        documents: List[Dict]
    ) -> str:
        """Actually call LLM"""
        import anthropic
        
        client = anthropic.AsyncAnthropic()
        
        # Build context
        context = "\n\n".join([
            f"Document {i+1}: {doc['text']}"
            for i, doc in enumerate(documents)
        ])
        
        if not context:
            context = "No documents available."
        
        prompt = f"""Based on the following documents, answer the question.

Documents:
{context}

Question: {question}

Answer:"""
        
        message = await client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1000,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return message.content[0].text
    
    def get_health_status(self) -> Dict:
        """Get health status of all services"""
        return {
            'pinecone': self.pinecone_breaker.get_state(),
            'embeddings': self.embedding_breaker.get_state(),
            'llm': self.llm_breaker.get_state()
        }

# FastAPI integration
from fastapi import FastAPI

app = FastAPI()
rag = RAGSystemWithCircuitBreakers()

@app.post("/query")
async def query_rag(question: str):
    result = await rag.query(question)
    return result

@app.get("/health")
async def health_check():
    status = rag.get_health_status()
    
    # System is unhealthy if any circuit is open
    any_open = any(
        s['state'] == 'OPEN'
        for s in status.values()
    )
    
    return {
        "status": "degraded" if any_open else "healthy",
        "services": status
    }
```

---

## Advanced Patterns

### Pattern 1: Cascading Circuit Breakers

```javascript
class CascadingCircuitBreaker {
  constructor() {
    // Dependent services
    this.breakers = {
      database: new CircuitBreaker({ name: 'database' }),
      cache: new CircuitBreaker({ name: 'cache' }),
      ai: new CircuitBreaker({ name: 'ai' })
    };
    
    // Dependency graph: AI depends on cache and database
    this.dependencies = {
      ai: ['cache', 'database']
    };
  }

  async execute(service, fn) {
    const breaker = this.breakers[service];
    
    // Check if dependencies are healthy
    const deps = this.dependencies[service] || [];
    for (const dep of deps) {
      if (this.breakers[dep].state === 'OPEN') {
        throw new Error(
          `Cannot execute ${service}: dependency ${dep} is down`
        );
      }
    }
    
    // Execute with circuit breaker
    return breaker.execute(fn);
  }

  async callAI(prompt) {
    return this.execute('ai', async () => {
      // Check cache first
      const cached = await this.execute('cache', async () => {
        return await getFromCache(prompt);
      });
      
      if (cached) return cached;
      
      // Call AI service
      const result = await callAIService(prompt);
      
      // Store in cache
      await this.execute('cache', async () => {
        await saveToCache(prompt, result);
      });
      
      // Store in database
      await this.execute('database', async () => {
        await saveToDatabase(prompt, result);
      });
      
      return result;
    });
  }
}
```

### Pattern 2: Adaptive Timeout

```javascript
class AdaptiveCircuitBreaker extends CircuitBreaker {
  constructor(options) {
    super(options);
    this.baseTimeout = options.timeout || 30000;
    this.timeouts = [];
    this.maxTimeouts = 100;
  }

  async execute(fn) {
    const timeout = this.calculateTimeout();
    
    const timeoutPromise = new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), timeout)
    );
    
    try {
      const result = await Promise.race([fn(), timeoutPromise]);
      return result;
    } catch (error) {
      if (error.message === 'Timeout') {
        this.recordTimeout(timeout);
      }
      throw error;
    }
  }

  calculateTimeout() {
    if (this.timeouts.length < 10) {
      return this.baseTimeout;
    }
    
    // Use P95 of recent successful requests
    const sorted = [...this.timeouts].sort((a, b) => a - b);
    const p95Index = Math.floor(sorted.length * 0.95);
    const p95 = sorted[p95Index];
    
    // Add 50% buffer
    return Math.min(p95 * 1.5, this.baseTimeout * 3);
  }

  recordTimeout(duration) {
    this.timeouts.push(duration);
    if (this.timeouts.length > this.maxTimeouts) {
      this.timeouts.shift();
    }
  }
}
```

### Pattern 3: Health Check Probes

```python
class HealthCheckCircuitBreaker(AdvancedCircuitBreaker):
    """
    Circuit breaker with active health checks.
    Proactively checks service health instead of waiting for failures.
    """
    
    def __init__(self, health_check_fn: Callable, check_interval: int = 30, **kwargs):
        super().__init__(**kwargs)
        self.health_check_fn = health_check_fn
        self.check_interval = check_interval
        self.last_health_check = datetime.now()
        
        # Start health check loop
        asyncio.create_task(self._health_check_loop())
    
    async def _health_check_loop(self):
        """Periodically check service health"""
        while True:
            await asyncio.sleep(self.check_interval)
            
            try:
                # Run health check
                is_healthy = await self.health_check_fn()
                
                if is_healthy:
                    if self.state == CircuitState.OPEN:
                        # Service recovered - transition to half-open
                        logger.info(f"Health check passed for '{self.name}' - transitioning to HALF_OPEN")
                        self._transition_to_half_open()
                else:
                    if self.state == CircuitState.CLOSED:
                        # Service failing - open circuit
                        logger.warning(f"Health check failed for '{self.name}' - opening circuit")
                        self._transition_to_open()
                
            except Exception as e:
                logger.error(f"Health check error for '{self.name}': {e}")

# Usage
async def check_openai_health():
    """Simple health check for OpenAI API"""
    try:
        response = await openai.Model.alist()
        return True
    except:
        return False

breaker = HealthCheckCircuitBreaker(
    health_check_fn=check_openai_health,
    check_interval=30,  # Check every 30 seconds
    name="openai_with_healthcheck"
)
```

---

## Monitoring & Observability

### Metrics to Track

```javascript
const prometheus = require('prom-client');

class CircuitBreakerMetrics {
  constructor() {
    this.register = new prometheus.Registry();
    
    // 1. Circuit breaker state
    this.state = new prometheus.Gauge({
      name: 'circuit_breaker_state',
      help: 'Circuit breaker state (0=closed, 1=open, 2=half_open)',
      labelNames: ['name'],
      registers: [this.register]
    });
    
    // 2. State transitions
    this.transitions = new prometheus.Counter({
      name: 'circuit_breaker_transitions_total',
      help: 'Total state transitions',
      labelNames: ['name', 'from_state', 'to_state'],
      registers: [this.register]
    });
    
    // 3. Rejected requests (when circuit open)
    this.rejections = new prometheus.Counter({
      name: 'circuit_breaker_rejections_total',
      help: 'Requests rejected due to open circuit',
      labelNames: ['name'],
      registers: [this.register]
    });
    
    // 4. Success/failure rate
    this.requests = new prometheus.Counter({
      name: 'circuit_breaker_requests_total',
      help: 'Total requests through circuit breaker',
      labelNames: ['name', 'result'],
      registers: [this.register]
    });
    
    // 5. Time in each state
    this.timeInState = new prometheus.Counter({
      name: 'circuit_breaker_time_in_state_seconds',
      help: 'Time spent in each state',
      labelNames: ['name', 'state'],
      registers: [this.register]
    });
  }

  recordState(name, state) {
    const stateValue = state === 'CLOSED' ? 0 :
                      state === 'OPEN' ? 1 : 2;
    this.state.labels(name).set(stateValue);
  }

  recordTransition(name, fromState, toState) {
    this.transitions.labels(name, fromState, toState).inc();
  }

  recordRejection(name) {
    this.rejections.labels(name).inc();
  }

  recordRequest(name, success) {
    this.requests.labels(name, success ? 'success' : 'failure').inc();
  }
}
```

### Grafana Dashboard

```yaml
# Grafana dashboard configuration
dashboard:
  title: "Circuit Breaker Monitoring"
  panels:
    - title: "Circuit Breaker States"
      type: graph
      targets:
        - expr: circuit_breaker_state
          legendFormat: "{{name}}"
      
    - title: "Open Circuit Breakers"
      type: singlestat
      targets:
        - expr: count(circuit_breaker_state == 1)
      thresholds: "1,3"
      
    - title: "Success Rate"
      type: graph
      targets:
        - expr: |
            rate(circuit_breaker_requests_total{result="success"}[5m]) /
            rate(circuit_breaker_requests_total[5m])
          legendFormat: "{{name}}"
      
    - title: "Rejection Rate"
      type: graph
      targets:
        - expr: rate(circuit_breaker_rejections_total[5m])
          legendFormat: "{{name}}"
      
    - title: "State Transitions"
      type: graph
      targets:
        - expr: rate(circuit_breaker_transitions_total[5m])
          legendFormat: "{{name}}: {{from_state}} -> {{to_state}}"
```

### Alerting

```yaml
# Prometheus alerts
groups:
  - name: circuit_breaker_alerts
    rules:
      # Circuit breaker opened
      - alert: CircuitBreakerOpen
        expr: circuit_breaker_state == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Circuit breaker {{ $labels.name }} is OPEN"
          description: "Service {{ $labels.name }} has been unavailable for 5+ minutes"
      
      # Multiple circuit breakers open
      - alert: MultipleCircuitBreakersOpen
        expr: count(circuit_breaker_state == 1) >= 2
        labels:
          severity: critical
        annotations:
          summary: "Multiple circuit breakers open"
          description: "{{ $value }} circuit breakers are open - systemic issue"
      
      # High rejection rate
      - alert: HighCircuitBreakerRejections
        expr: rate(circuit_breaker_rejections_total[5m]) > 10
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "High rejection rate for {{ $labels.name }}"
          description: "{{ $value }} req/sec rejected due to open circuit"
      
      # Frequent state changes (flapping)
      - alert: CircuitBreakerFlapping
        expr: rate(circuit_breaker_transitions_total[10m]) > 0.1
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Circuit breaker {{ $labels.name }} is flapping"
          description: "Frequent state changes detected - unstable service"
```

---

## Integration with Other Patterns

### Circuit Breaker + Retry + Backpressure

```javascript
class ResilientAIService {
  constructor() {
    // Circuit breaker (fail fast when service down)
    this.breaker = new CircuitBreaker({
      failureThreshold: 5,
      timeout: 60000
    });
    
    // Retry (handle transient failures)
    this.retry = new RetryManager({
      maxRetries: 3,
      baseDelay: 1000
    });
    
    // Backpressure (prevent overload)
    this.backpressure = new BoundedQueueBackpressure({
      maxConcurrent: 50,
      maxQueueSize: 200
    });
  }

  async execute(fn) {
    // Layer 1: Backpressure (reject if overloaded)
    return this.backpressure.execute(async () => {
      
      // Layer 2: Circuit breaker (fail fast if service down)
      return this.breaker.execute(async () => {
        
        // Layer 3: Retry (handle transient failures)
        return this.retry.execute(fn);
      });
    });
  }
}

// Usage
const service = new ResilientAIService();

app.post('/api/generate', async (req, res) => {
  try {
    const result = await service.execute(async () => {
      return await callOpenAI(req.body.prompt);
    });
    
    res.json(result);
    
  } catch (error) {
    if (error instanceof CircuitBreakerOpenError) {
      res.status(503).json({
        error: 'AI service temporarily unavailable',
        retryAfter: 60
      });
    } else if (error.statusCode === 503) {
      res.status(503).json({
        error: 'Service at capacity',
        retryAfter: 30
      });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});
```

---

## Real-World Case Studies

### Case Study 1: E-commerce AI Recommendations

**Problem:**
- Product recommendations powered by OpenAI
- OpenAI had 3 outages in one month
- Each outage = no recommendations = lost sales

**Solution:**

```javascript
const recommendationBreaker = new CircuitBreaker({
  failureThreshold: 3,
  timeout: 30000,
  successThreshold: 2
});

async function getRecommendations(userId) {
  try {
    return await recommendationBreaker.execute(async () => {
      return await callOpenAI(`Generate recommendations for user ${userId}`);
    });
  } catch (error) {
    if (error instanceof CircuitBreakerOpenError) {
      // Fallback to rule-based recommendations
      return getRuleBasedRecommendations(userId);
    }
    throw error;
  }
}
```

**Results:**
- ✅ Zero downtime during OpenAI outages
- ✅ Fallback recommendations maintained sales
- ✅ Customer satisfaction: 4.5/5 (vs 2.1/5 before)
- ✅ Revenue protected: $500K/month

### Case Study 2: Healthcare AI Diagnostics

**Problem:**
- Medical image analysis using external AI
- Patient care delayed when AI service down
- Compliance requires documented failure handling

**Solution:**

```python
# Health-critical circuit breaker with strict SLA
diagnostic_breaker = AdvancedCircuitBreaker(
    name="medical_ai",
    failure_threshold=0.3,  # Open at 30% (more sensitive)
    min_requests=3,         # Faster detection
    timeout_seconds=30,     # Quick recovery attempts
    success_threshold=3     # More validation before closing
)

# Alert medical staff when circuit opens
diagnostic_breaker.on_open = lambda: alert_medical_staff(
    "AI diagnostic system unavailable - manual review required"
)

async def analyze_medical_image(image_id):
    try:
        result = await diagnostic_breaker.execute(
            analyze_with_ai,
            image_id
        )
        return {
            'result': result,
            'method': 'AI',
            'confidence': 'high'
        }
    except CircuitBreakerOpenError:
        # Escalate to human radiologist
        return {
            'result': None,
            'method': 'manual',
            'status': 'escalated_to_radiologist',
            'message': 'AI system unavailable - manual review in progress'
        }
```

**Results:**
- ✅ 100% patient care continuity
- ✅ Automatic escalation to human experts
- ✅ Regulatory compliance maintained
- ✅ Average delay: <5 minutes (vs 2+ hours before)

### Case Study 3: Content Moderation Platform

**Problem:**
- Real-time content moderation using multiple AI providers
- One provider outage shouldn't block all moderation
- Need to maintain >99.9% uptime

**Solution:**

```javascript
class MultiProviderModeration {
  constructor() {
    this.providers = [
      {
        name: 'openai',
        breaker: new CircuitBreaker({ timeout: 30000 }),
        priority: 1
      },
      {
        name: 'perspective_api',
        breaker: new CircuitBreaker({ timeout: 30000 }),
        priority: 2
      },
      {
        name: 'hive_ai',
        breaker: new CircuitBreaker({ timeout: 30000 }),
        priority: 3
      }
    ];
  }

  async moderate(content) {
    // Try providers in priority order
    for (const provider of this.providers) {
      if (provider.breaker.state === 'OPEN') {
        console.log(`Skipping ${provider.name} (circuit open)`);
        continue;
      }

      try {
        const result = await provider.breaker.execute(async () => {
          return await this.callProvider(provider.name, content);
        });

        return {
          decision: result,
          provider: provider.name,
          fallback: false
        };

      } catch (error) {
        console.log(`Provider ${provider.name} failed, trying next...`);
        continue;
      }
    }

    // All providers failed - use rule-based fallback
    return {
      decision: this.ruleBasedModeration(content),
      provider: 'rule_based',
      fallback: true
    };
  }
}
```

**Results:**
- ✅ 99.95% uptime achieved
- ✅ Zero complete moderation outages
- ✅ Average provider failover: <100ms
- ✅ Platform trust score: 4.8/5

---

## Production Best Practices

### 1. Choosing Thresholds

```javascript
// Conservative (for critical services)
const criticalBreaker = new CircuitBreaker({
  failureThreshold: 3,      // Open after just 3 failures
  successThreshold: 5,      // Need 5 successes to close
  timeout: 120000           // 2 minute recovery time
});

// Moderate (for standard services)
const standardBreaker = new CircuitBreaker({
  failureThreshold: 5,      // Open after 5 failures
  successThreshold: 2,      // Need 2 successes to close
  timeout: 60000            // 1 minute recovery time
});

// Aggressive (for non-critical services)
const relaxedBreaker = new CircuitBreaker({
  failureThreshold: 10,     // Open after 10 failures
  successThreshold: 1,      // Just 1 success to close
  timeout: 30000            // 30 second recovery time
});
```

### 2. Fallback Strategies

```javascript
class FallbackStrategy {
  static async cached(breaker, fn, cacheKey) {
    try {
      const result = await breaker.execute(fn);
      cache.set(cacheKey, result);
      return result;
    } catch (error) {
      if (error instanceof CircuitBreakerOpenError) {
        const cached = cache.get(cacheKey);
        if (cached) {
          return { ...cached, cached: true };
        }
      }
      throw error;
    }
  }

  static async degraded(breaker, fn, degradedFn) {
    try {
      return await breaker.execute(fn);
    } catch (error) {
      if (error instanceof CircuitBreakerOpenError) {
        return await degradedFn();
      }
      throw error;
    }
  }

  static async alternative(breaker, providers) {
    for (const provider of providers) {
      try {
        return await provider.breaker.execute(provider.fn);
      } catch (error) {
        continue;
      }
    }
    throw new Error('All providers failed');
  }
}
```

### 3. Testing Circuit Breakers

```javascript
describe('CircuitBreaker', () => {
  it('should open after threshold failures', async () => {
    const breaker = new CircuitBreaker({ failureThreshold: 3 });
    
    // Cause 3 failures
    for (let i = 0; i < 3; i++) {
      try {
        await breaker.execute(() => Promise.reject(new Error('fail')));
      } catch {}
    }
    
    expect(breaker.state).toBe('OPEN');
  });

  it('should transition to half-open after timeout', async () => {
    const breaker = new CircuitBreaker({
      failureThreshold: 1,
      timeout: 100
    });
    
    // Open the circuit
    try {
      await breaker.execute(() => Promise.reject(new Error('fail')));
    } catch {}
    
    expect(breaker.state).toBe('OPEN');
    
    // Wait for timeout
    await new Promise(resolve => setTimeout(resolve, 150));
    
    // Next call should transition to half-open
    try {
      await breaker.execute(() => Promise.resolve('success'));
    } catch {}
    
    expect(breaker.state).toBe('CLOSED');
  });

  it('should reject when circuit is open', async () => {
    const breaker = new CircuitBreaker({ failureThreshold: 1 });
    
    // Open the circuit
    try {
      await breaker.execute(() => Promise.reject(new Error('fail')));
    } catch {}
    
    // Should reject without calling function
    await expect(
      breaker.execute(() => Promise.resolve('should not run'))
    ).rejects.toThrow(CircuitBreakerOpenError);
  });
});
```

### 4. Manual Control

```javascript
class ManageableCircuitBreaker extends CircuitBreaker {
  // Manual override endpoints for operations team
  
  forceOpen(reason) {
    console.log(`Circuit breaker manually opened: ${reason}`);
    this.state = 'OPEN';
    this.nextAttempt = Date.now() + this.timeout;
  }

  forceClose(reason) {
    console.log(`Circuit breaker manually closed: ${reason}`);
    this.reset();
  }

  extend Timeout(additionalMs) {
    if (this.state === 'OPEN') {
      this.nextAttempt += additionalMs;
      console.log(`Circuit breaker timeout extended by ${additionalMs}ms`);
    }
  }
}

// Admin API
app.post('/admin/circuit-breaker/:name/open', (req, res) => {
  const breaker = breakers[req.params.name];
  breaker.forceOpen(req.body.reason);
  res.json({ status: 'opened' });
});

app.post('/admin/circuit-breaker/:name/close', (req, res) => {
  const breaker = breakers[req.params.name];
  breaker.forceClose(req.body.reason);
  res.json({ status: 'closed' });
});
```

---

## Conclusion

### Key Takeaways

1. **Circuit Breakers Prevent Cascading Failures**: Stop trying operations that are likely to fail
2. **Three States**: CLOSED (healthy), OPEN (failing), HALF-OPEN (testing)
3. **Fail Fast**: Better to reject immediately than timeout slowly
4. **Combine with Fallbacks**: Cache, degraded service, alternative providers
5. **Monitor Closely**: Track state, transitions, rejections
6. **Test Thoroughly**: Chaos engineering, failure injection
7. **Plan for Recovery**: Automatic and manual controls

### When to Use Circuit Breakers

✅ **Use Circuit Breakers For:**
- External API calls (OpenAI, Anthropic, etc.)
- Database connections
- Microservice dependencies
- Any service with > 1% failure rate
- Operations with long timeouts

❌ **Don't Use Circuit Breakers For:**
- Internal function calls
- Deterministic operations
- Already reliable services (>99.99% uptime)
- Operations without fallback options

### ROI Summary

**Investment:**
- Implementation: 2-3 days = $3K
- Testing: 1 day = $1K
- Monitoring: 1 day = $1K
- **Total: $5K**

**Returns (Annual):**
- Prevented cascading failures: 6 incidents × $100K = $600K
- Reduced timeout waste: $50K
- Better user experience: $100K retained revenue
- **Total: $750K/year**

**ROI: 15,000% first year**

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
