# Bulkhead Pattern for AI Applications

## Table of Contents
1. [Introduction](#introduction)
2. [Why Bulkheads Matter for AI Systems](#why-bulkheads-matter)
3. [Bulkhead Types](#bulkhead-types)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Monitoring & Observability](#monitoring-observability)
8. [Integration with Other Patterns](#integration-with-other-patterns)
9. [Real-World Case Studies](#real-world-case-studies)
10. [Production Best Practices](#production-best-practices)

---

## Introduction

### What is a Bulkhead?

A **Bulkhead** is a design pattern that isolates different parts of a system into separate resource pools to prevent failures in one area from cascading to others. Named after the watertight compartments in ships that prevent a single breach from sinking the entire vessel.

**Analogy:** Like compartments in a submarine - if one section floods, watertight doors prevent the entire ship from sinking.

### The Problem Without Bulkheads

```
Scenario: Shared GPU pool for all AI services

Without Bulkheads:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
All services share 10 GPUs

14:00 - Image generation spike (1000 requests)
14:01 - All 10 GPUs consumed by image generation
14:02 - LLM service can't get GPU → requests fail
14:03 - Embedding service can't get GPU → requests fail
14:05 - Entire AI platform unresponsive
14:10 - OOM errors start appearing
14:15 - All GPU pods crash

Result:
- Complete platform outage
- All services down (not just image generation)
- 15 minutes downtime
- Manual intervention required
- Lost revenue across all services
```

### The Solution With Bulkheads

```
With Bulkheads:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Isolated resource pools:
- Image generation: 4 GPUs (isolated)
- LLM service: 4 GPUs (isolated)
- Embeddings: 2 GPUs (isolated)

14:00 - Image generation spike (1000 requests)
14:01 - Image generation uses its 4 GPUs fully
14:02 - LLM service continues normally (own 4 GPUs)
14:03 - Embeddings continue normally (own 2 GPUs)
14:05 - Image generation queue builds up
14:06 - Only image generation affected

Result:
- No outage
- 2/3 of services running normally
- Image generation degrades gracefully
- No cascading failures
- Revenue protected for LLM & embeddings
```

---

## Why Bulkheads Matter for AI Systems

### Unique AI System Challenges

1. **Expensive Shared Resources**
   - GPUs: $1-5 per hour each
   - API quotas: Limited by provider (OpenAI, Anthropic)
   - Memory: Fixed amount, can't expand on demand
   - Connection pools: Database, vector DB connections

2. **Variable Resource Consumption**
   - Text generation: 100ms - 10s
   - Image generation: 30s - 3 minutes
   - Video generation: 5 - 30 minutes
   - One slow operation can block everything

3. **Multi-Tenant Requirements**
   - Enterprise customers need guaranteed capacity
   - Free tier shouldn't affect paid tier
   - Different SLAs per customer segment
   - Noisy neighbor problem

4. **Cost Explosion Risk**
   - One service can monopolize expensive resources
   - API costs can spiral out of control
   - GPU waste from inefficient sharing
   - No budget enforcement without isolation

### Real Incident: No Bulkheads

**Black Friday 2023 - AI Shopping Assistant**

```
Timeline:
00:00 - Normal traffic: 50 req/sec across all services
06:00 - Black Friday sales start
06:05 - Traffic surge: 500 req/sec (10x normal)
06:10 - Product recommendations (free tier) consume all GPUs
06:12 - Premium chat service (paid) can't get resources
06:15 - Premium customers complaining
06:20 - Emergency: Scale up GPUs
06:25 - Still not enough - free tier consuming everything
06:30 - Manual intervention: Disable free tier
07:00 - Finally stable (free tier disabled)
10:00 - Re-enable free tier with limits

Impact:
- 4 hours of degraded service
- Premium customers affected by free tier
- $200K in emergency GPU costs
- Lost $500K revenue (cart abandonment)
- 5,000 support tickets
- Brand damage on biggest sales day
```

**Same Incident WITH Bulkheads:**

```
Timeline:
00:00 - Normal traffic: 50 req/sec
      - Premium: 20 GPUs guaranteed
      - Free tier: 10 GPUs max
06:00 - Black Friday sales start
06:05 - Traffic surge: 500 req/sec (10x normal)
06:10 - Free tier hits 10 GPU limit
06:11 - Free tier requests start queueing
06:12 - Premium service unaffected (own 20 GPUs)
06:15 - Free tier shows "High demand" message
      
Impact:
- Zero downtime
- Premium customers unaffected
- Free tier gracefully degraded
- Clear user messaging
- No emergency intervention
- Revenue protected
```

---

## Bulkhead Types

### 1. Thread Pool Bulkheads

Isolate by limiting concurrent operations per service.

```
Service A: Max 10 concurrent threads
Service B: Max 20 concurrent threads  
Service C: Max 5 concurrent threads
```

**Use For:** CPU-bound operations, API rate limiting

### 2. Connection Pool Bulkheads

Isolate database/external service connections.

```
Service A: 20 database connections
Service B: 30 database connections
Service C: 10 database connections
```

**Use For:** Database access, external APIs

### 3. GPU Memory Bulkheads

Isolate GPU memory per model/service.

```
Model A: 8GB GPU memory max
Model B: 12GB GPU memory max
Model C: 4GB GPU memory max
```

**Use For:** Model inference, GPU-intensive workloads

### 4. API Quota Bulkheads

Isolate external API usage per service.

```
Service A: 10K OpenAI tokens/day
Service B: 50K OpenAI tokens/day
Service C: 5K OpenAI tokens/day
```

**Use For:** Third-party API costs, rate limits

---

## Implementation Patterns

### Pattern 1: Simple Thread Pool Bulkhead

```javascript
const pLimit = require('p-limit');

class ThreadPoolBulkhead {
  constructor(config) {
    // Create separate limiters for each service
    this.pools = {};
    
    for (const [service, limit] of Object.entries(config)) {
      this.pools[service] = pLimit(limit);
    }
  }

  async execute(service, fn) {
    const pool = this.pools[service];
    
    if (!pool) {
      throw new Error(`Unknown service: ${service}`);
    }

    return pool(fn);
  }

  getUtilization(service) {
    const pool = this.pools[service];
    return {
      pending: pool.pendingCount,
      active: pool.activeCount
    };
  }
}

// Usage
const bulkhead = new ThreadPoolBulkhead({
  'llm': 10,           // Max 10 concurrent LLM requests
  'image': 5,          // Max 5 concurrent image generations
  'embedding': 50      // Max 50 concurrent embeddings
});

// LLM requests
async function generateText(prompt) {
  return bulkhead.execute('llm', async () => {
    return await callOpenAI(prompt);
  });
}

// Image requests
async function generateImage(prompt) {
  return bulkhead.execute('image', async () => {
    return await callStabilityAI(prompt);
  });
}

// Embeddings
async function getEmbedding(text) {
  return bulkhead.execute('embedding', async () => {
    return await callOpenAIEmbedding(text);
  });
}

// Express API
const express = require('express');
const app = express();

app.post('/api/chat', async (req, res) => {
  try {
    const result = await generateText(req.body.message);
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/api/image', async (req, res) => {
  try {
    const result = await generateImage(req.body.prompt);
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Monitor utilization
app.get('/api/metrics', (req, res) => {
  res.json({
    llm: bulkhead.getUtilization('llm'),
    image: bulkhead.getUtilization('image'),
    embedding: bulkhead.getUtilization('embedding')
  });
});
```

### Pattern 2: Connection Pool Bulkhead

```javascript
const genericPool = require('generic-pool');

class ConnectionPoolBulkhead {
  constructor() {
    this.pools = {};
  }

  createPool(name, config) {
    this.pools[name] = genericPool.createPool({
      create: async () => {
        // Create connection
        return await config.createConnection();
      },
      destroy: async (conn) => {
        // Cleanup
        await config.destroyConnection(conn);
      },
      validate: async (conn) => {
        // Check if connection is still healthy
        return await config.validateConnection(conn);
      }
    }, {
      max: config.maxConnections,
      min: config.minConnections,
      testOnBorrow: true,
      acquireTimeoutMillis: config.timeout || 5000
    });

    return this.pools[name];
  }

  async execute(poolName, fn) {
    const pool = this.pools[poolName];
    
    if (!pool) {
      throw new Error(`Pool ${poolName} not found`);
    }

    const connection = await pool.acquire();
    
    try {
      return await fn(connection);
    } finally {
      await pool.release(connection);
    }
  }

  async getStats(poolName) {
    const pool = this.pools[poolName];
    
    return {
      size: pool.size,
      available: pool.available,
      pending: pool.pending,
      borrowed: pool.borrowed
    };
  }
}

// Usage with OpenAI
const bulkhead = new ConnectionPoolBulkhead();

// Create separate pools for different services
bulkhead.createPool('llm', {
  maxConnections: 10,
  minConnections: 2,
  createConnection: async () => {
    return {
      client: new OpenAI({ apiKey: process.env.OPENAI_API_KEY }),
      createdAt: Date.now()
    };
  },
  destroyConnection: async (conn) => {
    // Cleanup if needed
  },
  validateConnection: async (conn) => {
    // Check if still valid (e.g., not too old)
    return Date.now() - conn.createdAt < 300000; // 5 minutes
  }
});

bulkhead.createPool('embeddings', {
  maxConnections: 20,
  minConnections: 5,
  createConnection: async () => {
    return {
      client: new OpenAI({ apiKey: process.env.OPENAI_API_KEY }),
      createdAt: Date.now()
    };
  },
  destroyConnection: async (conn) => {},
  validateConnection: async (conn) => {
    return Date.now() - conn.createdAt < 300000;
  }
});

// Use the pools
async function callLLM(prompt) {
  return bulkhead.execute('llm', async (conn) => {
    const response = await conn.client.chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: prompt }]
    });
    
    return response.choices[0].message.content;
  });
}

async function getEmbedding(text) {
  return bulkhead.execute('embeddings', async (conn) => {
    const response = await conn.client.embeddings.create({
      model: 'text-embedding-ada-002',
      input: text
    });
    
    return response.data[0].embedding;
  });
}
```

### Pattern 3: GPU Memory Bulkhead

```python
import asyncio
import torch
from typing import Dict, Optional
from dataclasses import dataclass

@dataclass
class GPUBulkhead:
    """Configuration for a GPU memory bulkhead"""
    name: str
    max_memory_mb: int
    max_concurrent: int
    semaphore: asyncio.Semaphore

class GPUMemoryBulkheads:
    """
    Isolate GPU memory per service to prevent OOM errors.
    
    Example:
        LLM: 8GB max, 2 concurrent
        Image Gen: 12GB max, 1 concurrent
        Embeddings: 2GB max, 4 concurrent
    """
    
    def __init__(self, total_gpu_memory_mb: int):
        self.total_memory = total_gpu_memory_mb
        self.bulkheads: Dict[str, GPUBulkhead] = {}
        self.allocated_memory = 0
    
    def create_bulkhead(
        self,
        name: str,
        max_memory_mb: int,
        max_concurrent: int
    ):
        """Create a new GPU memory bulkhead"""
        
        # Check if we have enough memory
        if self.allocated_memory + max_memory_mb > self.total_memory:
            raise ValueError(
                f"Cannot allocate {max_memory_mb}MB - only "
                f"{self.total_memory - self.allocated_memory}MB available"
            )
        
        self.bulkheads[name] = GPUBulkhead(
            name=name,
            max_memory_mb=max_memory_mb,
            max_concurrent=max_concurrent,
            semaphore=asyncio.Semaphore(max_concurrent)
        )
        
        self.allocated_memory += max_memory_mb
        
        print(f"Created bulkhead '{name}': {max_memory_mb}MB, {max_concurrent} concurrent")
    
    async def execute(self, bulkhead_name: str, fn):
        """Execute function with GPU memory isolation"""
        
        bulkhead = self.bulkheads.get(bulkhead_name)
        if not bulkhead:
            raise ValueError(f"Bulkhead '{bulkhead_name}' not found")
        
        # Acquire semaphore (limits concurrent operations)
        async with bulkhead.semaphore:
            # Set memory limit for this operation
            memory_fraction = bulkhead.max_memory_mb / self.total_memory
            
            # Configure PyTorch memory limit
            torch.cuda.set_per_process_memory_fraction(memory_fraction)
            
            try:
                result = await fn()
                return result
            finally:
                # Clear cache to free memory
                torch.cuda.empty_cache()
    
    def get_stats(self) -> Dict:
        """Get utilization stats for all bulkheads"""
        return {
            name: {
                'max_memory_mb': bulkhead.max_memory_mb,
                'max_concurrent': bulkhead.max_concurrent,
                'active': bulkhead.max_concurrent - bulkhead.semaphore._value,
                'utilization': (bulkhead.max_concurrent - bulkhead.semaphore._value) / 
                              bulkhead.max_concurrent
            }
            for name, bulkhead in self.bulkheads.items()
        }

# Usage
gpu_bulkheads = GPUMemoryBulkheads(total_gpu_memory_mb=24576)  # 24GB GPU

# Create isolated memory pools
gpu_bulkheads.create_bulkhead('llm', max_memory_mb=8192, max_concurrent=2)
gpu_bulkheads.create_bulkhead('image_gen', max_memory_mb=12288, max_concurrent=1)
gpu_bulkheads.create_bulkhead('embeddings', max_memory_mb=4096, max_concurrent=4)

# Use the bulkheads
async def generate_text_llm(prompt: str):
    async def _generate():
        # Your LLM inference code
        model = load_model('llm')
        return model.generate(prompt)
    
    return await gpu_bulkheads.execute('llm', _generate)

async def generate_image(prompt: str):
    async def _generate():
        # Your image generation code
        model = load_model('stable-diffusion')
        return model.generate(prompt)
    
    return await gpu_bulkheads.execute('image_gen', _generate)

async def get_embeddings(texts: list):
    async def _generate():
        # Your embedding code
        model = load_model('embeddings')
        return model.encode(texts)
    
    return await gpu_bulkheads.execute('embeddings', _generate)

# FastAPI integration
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.post("/generate/text")
async def generate_text_endpoint(prompt: str):
    try:
        result = await generate_text_llm(prompt)
        return {"text": result}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/generate/image")
async def generate_image_endpoint(prompt: str):
    try:
        result = await generate_image(prompt)
        return {"image": result}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/metrics/gpu")
async def get_gpu_metrics():
    return gpu_bulkheads.get_stats()
```

### Pattern 4: API Quota Bulkhead

```javascript
class APIQuotaBulkhead {
  constructor(config) {
    // Track usage per service per time period
    this.quotas = {};
    this.usage = {};
    
    for (const [service, quota] of Object.entries(config)) {
      this.quotas[service] = quota;
      this.usage[service] = {
        daily: 0,
        monthly: 0,
        lastReset: Date.now()
      };
    }
    
    // Reset daily quotas at midnight
    this.startResetTimer();
  }

  async execute(service, fn, cost = 1) {
    const quota = this.quotas[service];
    const usage = this.usage[service];
    
    if (!quota) {
      throw new Error(`Service ${service} not configured`);
    }

    // Check daily quota
    if (usage.daily + cost > quota.daily) {
      throw new QuotaExceededError(
        `Daily quota exceeded for ${service}`,
        'daily',
        usage.daily,
        quota.daily
      );
    }

    // Check monthly quota
    if (usage.monthly + cost > quota.monthly) {
      throw new QuotaExceededError(
        `Monthly quota exceeded for ${service}`,
        'monthly',
        usage.monthly,
        quota.monthly
      );
    }

    // Execute function
    try {
      const result = await fn();
      
      // Deduct from quota on success
      usage.daily += cost;
      usage.monthly += cost;
      
      return result;
    } catch (error) {
      // Don't charge for failed requests
      throw error;
    }
  }

  getUsage(service) {
    const usage = this.usage[service];
    const quota = this.quotas[service];
    
    return {
      daily: {
        used: usage.daily,
        limit: quota.daily,
        remaining: quota.daily - usage.daily,
        percent: (usage.daily / quota.daily) * 100
      },
      monthly: {
        used: usage.monthly,
        limit: quota.monthly,
        remaining: quota.monthly - usage.monthly,
        percent: (usage.monthly / quota.monthly) * 100
      }
    };
  }

  startResetTimer() {
    // Reset daily quotas at midnight
    const now = new Date();
    const tomorrow = new Date(now);
    tomorrow.setDate(tomorrow.getDate() + 1);
    tomorrow.setHours(0, 0, 0, 0);
    
    const msUntilMidnight = tomorrow - now;
    
    setTimeout(() => {
      this.resetDaily();
      // Set up next reset
      setInterval(() => this.resetDaily(), 24 * 60 * 60 * 1000);
    }, msUntilMidnight);
  }

  resetDaily() {
    console.log('Resetting daily quotas...');
    for (const service in this.usage) {
      this.usage[service].daily = 0;
    }
  }

  resetMonthly() {
    console.log('Resetting monthly quotas...');
    for (const service in this.usage) {
      this.usage[service].monthly = 0;
    }
  }
}

class QuotaExceededError extends Error {
  constructor(message, period, used, limit) {
    super(message);
    this.name = 'QuotaExceededError';
    this.period = period;
    this.used = used;
    this.limit = limit;
  }
}

// Usage with different cost per operation
const quotaBulkhead = new APIQuotaBulkhead({
  'openai_llm': {
    daily: 100000,    // 100K tokens per day
    monthly: 3000000  // 3M tokens per month
  },
  'openai_embeddings': {
    daily: 500000,
    monthly: 15000000
  },
  'stability_images': {
    daily: 1000,      // 1000 images per day
    monthly: 30000
  }
});

// LLM with token cost
async function callOpenAI(prompt, maxTokens = 1000) {
  const estimatedCost = maxTokens; // Rough estimate
  
  return quotaBulkhead.execute('openai_llm', async () => {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }],
        max_tokens: maxTokens
      })
    });
    
    return response.json();
  }, estimatedCost);
}

// Express API with quota info
app.post('/api/generate', async (req, res) => {
  try {
    const result = await callOpenAI(req.body.prompt, req.body.maxTokens);
    
    // Include quota info in headers
    const usage = quotaBulkhead.getUsage('openai_llm');
    res.set('X-Quota-Remaining-Daily', usage.daily.remaining);
    res.set('X-Quota-Remaining-Monthly', usage.monthly.remaining);
    
    res.json(result);
    
  } catch (error) {
    if (error instanceof QuotaExceededError) {
      res.status(429)
        .json({
          error: 'Quota exceeded',
          period: error.period,
          used: error.used,
          limit: error.limit,
          message: `${error.period} quota exhausted. Resets at midnight.`
        });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});

// Admin endpoint to check quota usage
app.get('/admin/quotas', (req, res) => {
  const quotas = {
    openai_llm: quotaBulkhead.getUsage('openai_llm'),
    openai_embeddings: quotaBulkhead.getUsage('openai_embeddings'),
    stability_images: quotaBulkhead.getUsage('stability_images')
  };
  
  res.json(quotas);
});
```

---

## AI-Specific Use Cases

### Use Case 1: Multi-Model Inference Platform

```python
import asyncio
from typing import Dict, List
from dataclasses import dataclass

@dataclass
class ModelConfig:
    """Configuration for a model bulkhead"""
    max_concurrent: int
    max_memory_mb: int
    timeout_seconds: int

class MultiModelPlatform:
    """
    Platform serving multiple AI models with resource isolation.
    
    Models:
    - GPT-4 (expensive, high priority)
    - Claude (expensive, high priority)
    - Llama-2 (moderate, medium priority)
    - Embeddings (cheap, low priority)
    """
    
    def __init__(self, total_gpu_memory_mb: int):
        self.gpu_bulkheads = GPUMemoryBulkheads(total_gpu_memory_mb)
        
        # Configure bulkheads per model
        self.models = {
            'gpt4': ModelConfig(
                max_concurrent=5,
                max_memory_mb=8192,
                timeout_seconds=30
            ),
            'claude': ModelConfig(
                max_concurrent=5,
                max_memory_mb=8192,
                timeout_seconds=30
            ),
            'llama2': ModelConfig(
                max_concurrent=10,
                max_memory_mb=6144,
                timeout_seconds=20
            ),
            'embeddings': ModelConfig(
                max_concurrent=50,
                max_memory_mb=2048,
                timeout_seconds=5
            )
        }
        
        # Create GPU bulkheads
        for name, config in self.models.items():
            self.gpu_bulkheads.create_bulkhead(
                name=name,
                max_memory_mb=config.max_memory_mb,
                max_concurrent=config.max_concurrent
            )
    
    async def generate(self, model: str, prompt: str) -> Dict:
        """Generate with specified model"""
        
        if model not in self.models:
            raise ValueError(f"Unknown model: {model}")
        
        config = self.models[model]
        
        async def _generate():
            # Model-specific inference
            if model == 'gpt4':
                return await self._call_openai(prompt)
            elif model == 'claude':
                return await self._call_anthropic(prompt)
            elif model == 'llama2':
                return await self._run_llama2(prompt)
            elif model == 'embeddings':
                return await self._generate_embeddings(prompt)
        
        try:
            # Execute with timeout
            result = await asyncio.wait_for(
                self.gpu_bulkheads.execute(model, _generate),
                timeout=config.timeout_seconds
            )
            
            return {
                'text': result,
                'model': model,
                'success': True
            }
            
        except asyncio.TimeoutError:
            return {
                'text': None,
                'model': model,
                'success': False,
                'error': f'Timeout after {config.timeout_seconds}s'
            }
        except Exception as e:
            return {
                'text': None,
                'model': model,
                'success': False,
                'error': str(e)
            }
    
    async def _call_openai(self, prompt: str) -> str:
        """Call OpenAI API"""
        import openai
        
        response = await openai.ChatCompletion.acreate(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )
        
        return response.choices[0].message.content
    
    async def _call_anthropic(self, prompt: str) -> str:
        """Call Anthropic API"""
        import anthropic
        
        client = anthropic.AsyncAnthropic()
        message = await client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1000,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return message.content[0].text
    
    async def _run_llama2(self, prompt: str) -> str:
        """Run local Llama-2 model"""
        # Your local model inference code
        pass
    
    async def _generate_embeddings(self, text: str) -> List[float]:
        """Generate embeddings"""
        # Your embedding code
        pass
    
    def get_dashboard(self) -> Dict:
        """Get real-time dashboard data"""
        stats = self.gpu_bulkheads.get_stats()
        
        return {
            'models': {
                name: {
                    'active': stats[name]['active'],
                    'capacity': stats[name]['max_concurrent'],
                    'utilization': f"{stats[name]['utilization']*100:.1f}%",
                    'memory_mb': stats[name]['max_memory_mb']
                }
                for name in stats
            },
            'total_memory_mb': self.gpu_bulkheads.total_memory,
            'allocated_memory_mb': self.gpu_bulkheads.allocated_memory
        }

# FastAPI application
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()
platform = MultiModelPlatform(total_gpu_memory_mb=24576)

class GenerateRequest(BaseModel):
    model: str
    prompt: str

@app.post("/generate")
async def generate_endpoint(request: GenerateRequest):
    result = await platform.generate(request.model, request.prompt)
    
    if not result['success']:
        raise HTTPException(
            status_code=500,
            detail=result['error']
        )
    
    return result

@app.get("/dashboard")
async def dashboard():
    return platform.get_dashboard()
```

### Use Case 2: Multi-Tenant SaaS Platform

```javascript
const pLimit = require('p-limit');

class MultiTenantBulkheads {
  constructor() {
    // Different tiers get different resources
    this.tiers = {
      enterprise: {
        llm: pLimit(50),       // 50 concurrent
        image: pLimit(20),
        embedding: pLimit(100),
        dailyQuota: 1000000    // 1M requests/day
      },
      pro: {
        llm: pLimit(20),
        image: pLimit(5),
        embedding: pLimit(50),
        dailyQuota: 100000
      },
      free: {
        llm: pLimit(5),
        image: pLimit(1),
        embedding: pLimit(10),
        dailyQuota: 1000
      }
    };
    
    // Track usage per tenant
    this.usage = new Map();
  }

  async execute(tenantId, tier, service, fn) {
    // Get tenant's bulkhead
    const bulkheads = this.tiers[tier];
    
    if (!bulkheads) {
      throw new Error(`Invalid tier: ${tier}`);
    }

    const limiter = bulkheads[service];
    
    if (!limiter) {
      throw new Error(`Invalid service: ${service}`);
    }

    // Check daily quota
    const usage = this.getUsage(tenantId);
    if (usage >= bulkheads.dailyQuota) {
      throw new QuotaExceededError(
        `Daily quota of ${bulkheads.dailyQuota} exceeded for tenant ${tenantId}`
      );
    }

    // Execute with tier-specific limiter
    try {
      const result = await limiter(fn);
      
      // Track usage
      this.incrementUsage(tenantId);
      
      return result;
    } catch (error) {
      throw error;
    }
  }

  getUsage(tenantId) {
    const key = `${tenantId}:${this.getToday()}`;
    return this.usage.get(key) || 0;
  }

  incrementUsage(tenantId) {
    const key = `${tenantId}:${this.getToday()}`;
    const current = this.usage.get(key) || 0;
    this.usage.set(key, current + 1);
  }

  getToday() {
    return new Date().toISOString().split('T')[0];
  }

  getUtilization(tier, service) {
    const limiter = this.tiers[tier]?.[service];
    if (!limiter) return null;
    
    return {
      active: limiter.activeCount,
      pending: limiter.pendingCount,
      limit: limiter.limit
    };
  }

  getTenantStats(tenantId, tier) {
    const usage = this.getUsage(tenantId);
    const quota = this.tiers[tier].dailyQuota;
    
    return {
      usage: usage,
      quota: quota,
      remaining: quota - usage,
      utilization: (usage / quota) * 100
    };
  }
}

class QuotaExceededError extends Error {
  constructor(message) {
    super(message);
    this.name = 'QuotaExceededError';
    this.statusCode = 429;
  }
}

// Express application
const express = require('express');
const app = express();
const bulkheads = new MultiTenantBulkheads();

// Middleware to extract tenant info
async function getTenant(req) {
  const apiKey = req.headers['x-api-key'];
  // Fetch tenant info from database
  return {
    id: 'tenant-123',
    tier: 'pro' // or 'enterprise', 'free'
  };
}

app.post('/api/generate', async (req, res) => {
  try {
    const tenant = await getTenant(req);
    const { prompt, service } = req.body;
    
    // Execute with tenant's bulkhead
    const result = await bulkheads.execute(
      tenant.id,
      tenant.tier,
      service || 'llm',
      async () => {
        return await callAIService(prompt);
      }
    );
    
    // Include usage info in response headers
    const stats = bulkheads.getTenantStats(tenant.id, tenant.tier);
    res.set('X-Quota-Remaining', stats.remaining);
    res.set('X-Quota-Used', stats.usage);
    
    res.json(result);
    
  } catch (error) {
    if (error instanceof QuotaExceededError) {
      res.status(429).json({
        error: 'Quota exceeded',
        message: error.message,
        upgrade: 'Consider upgrading to Pro for higher limits'
      });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});

// Admin endpoint - tenant dashboard
app.get('/admin/tenant/:tenantId', async (req, res) => {
  const { tenantId } = req.params;
  const tenant = await getTenantFromDB(tenantId);
  
  res.json({
    tenant: tenant,
    stats: bulkheads.getTenantStats(tenantId, tenant.tier),
    utilization: {
      llm: bulkheads.getUtilization(tenant.tier, 'llm'),
      image: bulkheads.getUtilization(tenant.tier, 'image'),
      embedding: bulkheads.getUtilization(tenant.tier, 'embedding')
    }
  });
});
```

### Use Case 3: Cost-Controlled AI Pipeline

```python
from typing import Dict, Optional
import asyncio
from datetime import datetime, timedelta

class CostControlledPipeline:
    """
    AI pipeline with cost-based bulkheads.
    
    Budget enforcement:
    - Development: $100/day
    - Staging: $500/day
    - Production: $5000/day
    """
    
    def __init__(self):
        self.environments = {
            'development': {
                'daily_budget': 100,
                'cost_per_llm': 0.03,
                'cost_per_image': 0.10,
                'cost_per_embedding': 0.0001,
                'max_concurrent': 5
            },
            'staging': {
                'daily_budget': 500,
                'cost_per_llm': 0.03,
                'cost_per_image': 0.10,
                'cost_per_embedding': 0.0001,
                'max_concurrent': 20
            },
            'production': {
                'daily_budget': 5000,
                'cost_per_llm': 0.03,
                'cost_per_image': 0.10,
                'cost_per_embedding': 0.0001,
                'max_concurrent': 100
            }
        }
        
        # Track spending per environment
        self.spending = {env: 0.0 for env in self.environments}
        self.last_reset = datetime.now()
        
        # Semaphores per environment
        self.semaphores = {
            env: asyncio.Semaphore(config['max_concurrent'])
            for env, config in self.environments.items()
        }
    
    async def execute(
        self,
        environment: str,
        operation: str,
        fn
    ) -> Dict:
        """Execute operation with cost control"""
        
        env_config = self.environments.get(environment)
        if not env_config:
            raise ValueError(f"Unknown environment: {environment}")
        
        # Check daily budget
        cost_key = f'cost_per_{operation}'
        operation_cost = env_config.get(cost_key, 0.01)
        
        current_spend = self.spending[environment]
        
        if current_spend + operation_cost > env_config['daily_budget']:
            # Budget exceeded
            return {
                'success': False,
                'error': 'Budget exceeded',
                'current_spend': current_spend,
                'daily_budget': env_config['daily_budget'],
                'projected_cost': operation_cost
            }
        
        # Acquire semaphore
        async with self.semaphores[environment]:
            try:
                # Execute operation
                result = await fn()
                
                # Charge for successful operation
                self.spending[environment] += operation_cost
                
                return {
                    'success': True,
                    'result': result,
                    'cost': operation_cost,
                    'total_spent_today': self.spending[environment]
                }
                
            except Exception as e:
                # Don't charge for failed operations
                return {
                    'success': False,
                    'error': str(e),
                    'cost': 0
                }
    
    def get_budget_status(self, environment: str) -> Dict:
        """Get budget status for environment"""
        env_config = self.environments[environment]
        spent = self.spending[environment]
        
        return {
            'environment': environment,
            'daily_budget': env_config['daily_budget'],
            'spent': spent,
            'remaining': env_config['daily_budget'] - spent,
            'utilization': (spent / env_config['daily_budget']) * 100,
            'concurrent_capacity': env_config['max_concurrent'],
            'concurrent_active': env_config['max_concurrent'] - 
                               self.semaphores[environment]._value
        }
    
    def reset_daily_budgets(self):
        """Reset budgets (called daily at midnight)"""
        self.spending = {env: 0.0 for env in self.environments}
        self.last_reset = datetime.now()
        print(f"Daily budgets reset at {self.last_reset}")

# Usage
pipeline = CostControlledPipeline()

async def generate_llm_dev(prompt: str):
    """Generate text in development environment"""
    async def _generate():
        # Your LLM code
        return await call_openai(prompt)
    
    return await pipeline.execute('development', 'llm', _generate)

async def generate_image_prod(prompt: str):
    """Generate image in production"""
    async def _generate():
        # Your image generation code
        return await call_stability_ai(prompt)
    
    return await pipeline.execute('production', 'image', _generate)

# FastAPI endpoints
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.post("/dev/generate")
async def dev_generate(prompt: str):
    result = await generate_llm_dev(prompt)
    
    if not result['success']:
        if result.get('error') == 'Budget exceeded':
            raise HTTPException(
                status_code=429,
                detail={
                    'error': 'Development budget exceeded',
                    'spent': result['current_spend'],
                    'budget': result['daily_budget']
                }
            )
        else:
            raise HTTPException(status_code=500, detail=result['error'])
    
    return result

@app.get("/admin/budgets")
async def get_budgets():
    return {
        'development': pipeline.get_budget_status('development'),
        'staging': pipeline.get_budget_status('staging'),
        'production': pipeline.get_budget_status('production')
    }

@app.post("/admin/reset-budgets")
async def reset_budgets():
    pipeline.reset_daily_budgets()
    return {"status": "Budgets reset"}
```

---

## Advanced Patterns

### Pattern 1: Dynamic Bulkheads (Auto-Scaling)

```javascript
class DynamicBulkhead {
  constructor(config) {
    this.minConcurrency = config.min || 5;
    this.maxConcurrency = config.max || 100;
    this.currentConcurrency = this.minConcurrency;
    
    this.targetLatency = config.targetLatency || 2000; // 2s
    this.scaleUpThreshold = config.scaleUpThreshold || 0.8;
    this.scaleDownThreshold = config.scaleDownThreshold || 0.3;
    
    // Metrics
    this.latencies = [];
    this.activeRequests = 0;
    
    // Start auto-scaling loop
    setInterval(() => this.adjustConcurrency(), 30000); // Every 30s
  }

  async execute(fn) {
    // Wait if at capacity
    while (this.activeRequests >= this.currentConcurrency) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }

    this.activeRequests++;
    const startTime = Date.now();

    try {
      const result = await fn();
      const latency = Date.now() - startTime;
      this.recordLatency(latency);
      return result;
    } finally {
      this.activeRequests--;
    }
  }

  recordLatency(latency) {
    this.latencies.push(latency);
    if (this.latencies.length > 100) {
      this.latencies.shift();
    }
  }

  adjustConcurrency() {
    if (this.latencies.length < 10) return;

    const avgLatency = this.latencies.reduce((a, b) => a + b) / this.latencies.length;
    const utilization = this.activeRequests / this.currentConcurrency;

    console.log(`Current: ${this.currentConcurrency}, Util: ${utilization.toFixed(2)}, Latency: ${avgLatency.toFixed(0)}ms`);

    // Scale up if high utilization and low latency
    if (utilization > this.scaleUpThreshold && avgLatency < this.targetLatency) {
      const newConcurrency = Math.min(
        Math.ceil(this.currentConcurrency * 1.2),
        this.maxConcurrency
      );
      
      if (newConcurrency > this.currentConcurrency) {
        console.log(`⬆️  Scaling up: ${this.currentConcurrency} → ${newConcurrency}`);
        this.currentConcurrency = newConcurrency;
      }
    }

    // Scale down if low utilization
    if (utilization < this.scaleDownThreshold) {
      const newConcurrency = Math.max(
        Math.floor(this.currentConcurrency * 0.9),
        this.minConcurrency
      );
      
      if (newConcurrency < this.currentConcurrency) {
        console.log(`⬇️  Scaling down: ${this.currentConcurrency} → ${newConcurrency}`);
        this.currentConcurrency = newConcurrency;
      }
    }
  }

  getStats() {
    return {
      currentConcurrency: this.currentConcurrency,
      activeRequests: this.activeRequests,
      utilization: this.activeRequests / this.currentConcurrency,
      avgLatency: this.latencies.reduce((a, b) => a + b, 0) / this.latencies.length
    };
  }
}

// Usage
const dynamicBulkhead = new DynamicBulkhead({
  min: 5,
  max: 100,
  targetLatency: 2000
});

app.post('/api/generate', async (req, res) => {
  const result = await dynamicBulkhead.execute(async () => {
    return await callAIService(req.body.prompt);
  });
  
  res.json(result);
});

app.get('/metrics', (req, res) => {
  res.json(dynamicBulkhead.getStats());
});
```

### Pattern 2: Priority-Based Bulkheads

```python
import asyncio
from typing import Dict, Callable
from enum import Enum

class Priority(Enum):
    HIGH = 1
    MEDIUM = 2
    LOW = 3

class PriorityBulkhead:
    """
    Bulkhead with priority queues.
    High priority requests get served first.
    """
    
    def __init__(self, max_concurrent: int):
        self.max_concurrent = max_concurrent
        self.active = 0
        self.queues = {
            Priority.HIGH: asyncio.Queue(),
            Priority.MEDIUM: asyncio.Queue(),
            Priority.LOW: asyncio.Queue()
        }
        
        # Start processor
        asyncio.create_task(self._process_queues())
    
    async def execute(self, fn: Callable, priority: Priority = Priority.MEDIUM):
        """Execute with priority"""
        future = asyncio.Future()
        
        # Add to appropriate queue
        await self.queues[priority].put((fn, future))
        
        # Wait for result
        return await future
    
    async def _process_queues(self):
        """Process requests by priority"""
        while True:
            # Wait for capacity
            while self.active >= self.max_concurrent:
                await asyncio.sleep(0.1)
            
            # Get next request (priority order)
            fn, future = await self._get_next_request()
            
            # Process in background
            asyncio.create_task(self._execute_request(fn, future))
    
    async def _get_next_request(self):
        """Get next request from highest priority queue"""
        # Check queues in priority order
        for priority in [Priority.HIGH, Priority.MEDIUM, Priority.LOW]:
            queue = self.queues[priority]
            if not queue.empty():
                return await queue.get()
        
        # All queues empty - wait
        while True:
            for priority in [Priority.HIGH, Priority.MEDIUM, Priority.LOW]:
                queue = self.queues[priority]
                if not queue.empty():
                    return await queue.get()
            await asyncio.sleep(0.1)
    
    async def _execute_request(self, fn, future):
        """Execute a request"""
        self.active += 1
        try:
            result = await fn()
            future.set_result(result)
        except Exception as e:
            future.set_exception(e)
        finally:
            self.active -= 1
    
    def get_stats(self) -> Dict:
        return {
            'active': self.active,
            'capacity': self.max_concurrent,
            'queued': {
                'high': self.queues[Priority.HIGH].qsize(),
                'medium': self.queues[Priority.MEDIUM].qsize(),
                'low': self.queues[Priority.LOW].qsize()
            }
        }

# Usage
bulkhead = PriorityBulkhead(max_concurrent=10)

async def handle_request(user_tier: str, prompt: str):
    # Determine priority from user tier
    priority = {
        'enterprise': Priority.HIGH,
        'pro': Priority.MEDIUM,
        'free': Priority.LOW
    }.get(user_tier, Priority.LOW)
    
    # Execute with priority
    result = await bulkhead.execute(
        lambda: generate_text(prompt),
        priority=priority
    )
    
    return result
```

### Pattern 3: Cascading Bulkheads

```javascript
class CascadingBulkheads {
  constructor() {
    // Create hierarchy of bulkheads
    this.bulkheads = {
      // Layer 1: External APIs
      openai: pLimit(10),
      anthropic: pLimit(10),
      stability: pLimit(5),
      
      // Layer 2: Our services (depend on Layer 1)
      chat: pLimit(50),
      imageGen: pLimit(20),
      voiceClone: pLimit(10)
    };
    
    // Define dependencies
    this.dependencies = {
      chat: ['openai', 'anthropic'],
      imageGen: ['stability'],
      voiceClone: ['openai']
    };
  }

  async execute(service, fn) {
    // Check if dependencies are healthy
    const deps = this.dependencies[service] || [];
    for (const dep of deps) {
      if (this.getUtilization(dep) > 0.9) {
        throw new Error(
          `Cannot execute ${service}: dependency ${dep} at capacity`
        );
      }
    }

    // Execute with service bulkhead
    const bulkhead = this.bulkheads[service];
    return bulkhead(fn);
  }

  getUtilization(service) {
    const bulkhead = this.bulkheads[service];
    return (bulkhead.activeCount + bulkhead.pendingCount) / 
           bulkhead.concurrency;
  }

  getHealthStatus() {
    return Object.entries(this.bulkheads).map(([name, bulkhead]) => ({
      service: name,
      utilization: this.getUtilization(name),
      active: bulkhead.activeCount,
      pending: bulkhead.pendingCount,
      healthy: this.getUtilization(name) < 0.8
    }));
  }
}
```

---

## Monitoring & Observability

### Metrics to Track

```javascript
const prometheus = require('prom-client');

class BulkheadMetrics {
  constructor() {
    this.register = new prometheus.Registry();
    
    // 1. Pool utilization
    this.utilization = new prometheus.Gauge({
      name: 'bulkhead_utilization',
      help: 'Bulkhead pool utilization (0-1)',
      labelNames: ['service'],
      registers: [this.register]
    });
    
    // 2. Active requests
    this.activeRequests = new prometheus.Gauge({
      name: 'bulkhead_active_requests',
      help: 'Currently active requests',
      labelNames: ['service'],
      registers: [this.register]
    });
    
    // 3. Queued requests
    this.queuedRequests = new prometheus.Gauge({
      name: 'bulkhead_queued_requests',
      help: 'Requests waiting in queue',
      labelNames: ['service'],
      registers: [this.register]
    });
    
    // 4. Rejections
    this.rejections = new prometheus.Counter({
      name: 'bulkhead_rejections_total',
      help: 'Total rejected requests',
      labelNames: ['service', 'reason'],
      registers: [this.register]
    });
    
    // 5. Wait time
    this.waitTime = new prometheus.Histogram({
      name: 'bulkhead_wait_seconds',
      help: 'Time spent waiting for bulkhead',
      labelNames: ['service'],
      buckets: [0.1, 0.5, 1, 2, 5, 10, 30],
      registers: [this.register]
    });
  }

  recordUtilization(service, active, capacity) {
    this.utilization.labels(service).set(active / capacity);
    this.activeRequests.labels(service).set(active);
  }

  recordQueue(service, depth) {
    this.queuedRequests.labels(service).set(depth);
  }

  recordRejection(service, reason) {
    this.rejections.labels(service, reason).inc();
  }

  recordWaitTime(service, seconds) {
    this.waitTime.labels(service).observe(seconds);
  }
}
```

### Grafana Dashboard

```yaml
# Grafana dashboard for bulkheads
panels:
  - title: "Bulkhead Utilization"
    type: graph
    targets:
      - expr: bulkhead_utilization
        legendFormat: "{{service}}"
    alert:
      conditions:
        - evaluator:
            params: [0.8]
            type: gt

  - title: "Active vs Queued Requests"
    type: graph
    targets:
      - expr: bulkhead_active_requests
        legendFormat: "{{service}} - active"
      - expr: bulkhead_queued_requests
        legendFormat: "{{service}} - queued"

  - title: "Rejection Rate"
    type: graph
    targets:
      - expr: rate(bulkhead_rejections_total[5m])
        legendFormat: "{{service}} - {{reason}}"

  - title: "P95 Wait Time"
    type: graph
    targets:
      - expr: |
          histogram_quantile(0.95, 
            rate(bulkhead_wait_seconds_bucket[5m])
          )
        legendFormat: "{{service}}"
```

### Alerting

```yaml
# Prometheus alerts
groups:
  - name: bulkhead_alerts
    rules:
      # High utilization
      - alert: BulkheadHighUtilization
        expr: bulkhead_utilization > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Bulkhead {{ $labels.service }} high utilization"
          description: "{{ $labels.service }} at {{ $value | humanizePercentage }}"

      # Queue building
      - alert: BulkheadQueueGrowing
        expr: bulkhead_queued_requests > 100
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} queue backlog"
          description: "{{ $value }} requests queued"

      # High rejection rate
      - alert: BulkheadHighRejections
        expr: rate(bulkhead_rejections_total[5m]) > 1
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service }} rejecting requests"
          description: "{{ $value }} rejections/sec"
```

---

## Integration with Other Patterns

### Bulkhead + Circuit Breaker + Retry

```javascript
class ResilientBulkhead {
  constructor(config) {
    // Bulkhead (resource isolation)
    this.bulkhead = new ThreadPoolBulkhead({
      [config.name]: config.maxConcurrent
    });
    
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
  }

  async execute(fn) {
    // Layer 1: Bulkhead (prevent overload)
    return this.bulkhead.execute(config.name, async () => {
      
      // Layer 2: Circuit breaker (fail fast if down)
      return this.breaker.execute(async () => {
        
        // Layer 3: Retry (handle transient failures)
        return this.retry.execute(fn);
      });
    });
  }
}

// Usage
const resilientService = new ResilientBulkhead({
  name: 'openai',
  maxConcurrent: 10
});

app.post('/api/generate', async (req, res) => {
  try {
    const result = await resilientService.execute(async () => {
      return await callOpenAI(req.body.prompt);
    });
    
    res.json(result);
  } catch (error) {
    // Handle different error types
    res.status(500).json({ error: error.message });
  }
});
```

---

## Real-World Case Studies

### Case Study 1: Video Streaming Platform (AI Recommendations)

**Problem:**
- Recommendation engine using multiple AI models
- User watching video → generate recommendations in real-time
- Free tier users overwhelming system → premium users affected

**Solution:**

```javascript
const bulkheads = new MultiTenantBulkheads();

// Configure tiers
bulkheads.configure({
  premium: {
    recommendations: pLimit(100),
    transcription: pLimit(50)
  },
  free: {
    recommendations: pLimit(20),
    transcription: pLimit(5)
  }
});
```

**Results:**
- ✅ Premium users: 99.9% success rate
- ✅ Free users: 92% success rate (acceptable degradation)
- ✅ Revenue protected: $2M/month
- ✅ Zero premium user complaints
- ✅ 40% cost reduction (prevented overuse)

### Case Study 2: Healthcare AI Diagnostics

**Problem:**
- Multiple diagnostic AI models sharing GPU resources
- Emergency diagnostics competing with routine scans
- Need guaranteed capacity for critical cases

**Solution:**

```python
# Priority-based GPU bulkheads
gpu_bulkheads = GPUMemoryBulkheads(total_gpu_memory_mb=48000)

# Emergency gets guaranteed capacity
gpu_bulkheads.create_bulkhead('emergency', max_memory_mb=16000, max_concurrent=4)
gpu_bulkheads.create_bulkhead('routine', max_memory_mb=24000, max_concurrent=8)
gpu_bulkheads.create_bulkhead('research', max_memory_mb=8000, max_concurrent=2)
```

**Results:**
- ✅ Emergency cases: 100% capacity guaranteed
- ✅ Zero delays for critical diagnostics
- ✅ Routine scans: 95% success rate
- ✅ Research: 85% success rate (acceptable)
- ✅ Regulatory compliance maintained

### Case Study 3: E-Commerce AI Search

**Problem:**
- AI-powered search using embeddings + LLM
- Black Friday traffic spikes
- Search outage = lost sales

**Solution:**

```javascript
// Isolated bulkheads per search component
const searchBulkheads = new ThreadPoolBulkhead({
  'embedding': 100,      // Fast operation
  'llm_rerank': 20,      // Slow operation
  'cache_lookup': 200    // Very fast
});

async function aiSearch(query) {
  // Step 1: Embedding (isolated)
  const embedding = await searchBulkheads.execute('embedding', async () => {
    return await getEmbedding(query);
  });

  // Step 2: Vector search (isolated)
  const results = await vectorSearch(embedding);

  // Step 3: LLM rerank (isolated)
  const reranked = await searchBulkheads.execute('llm_rerank', async () => {
    return await llmRerank(query, results);
  });

  return reranked;
}
```

**Results:**
- ✅ Handled 50x Black Friday traffic
- ✅ Zero search outages
- ✅ 99.5% success rate during peak
- ✅ $5M revenue protected
- ✅ Best Black Friday ever

---

## Production Best Practices

### 1. Choosing Pool Sizes

```javascript
// Conservative (for critical services)
const criticalBulkhead = pLimit(5);

// Moderate (for standard services)
const standardBulkhead = pLimit(20);

// Aggressive (for high-throughput services)
const aggressiveBulkhead = pLimit(100);

// Rule of thumb: 
// Start with expected_rps * avg_duration_seconds
// Example: 10 req/sec * 2 sec = 20 concurrent
```

### 2. Testing Bulkheads

```javascript
describe('Bulkhead', () => {
  it('should isolate services', async () => {
    const bulkhead = new ThreadPoolBulkhead({
      serviceA: 5,
      serviceB: 10
    });

    // Saturate serviceA
    const promisesA = Array(10).fill(0).map(() =>
      bulkhead.execute('serviceA', () => sleep(1000))
    );

    // serviceB should still work
    const startTime = Date.now();
    await bulkhead.execute('serviceB', () => Promise.resolve('OK'));
    const duration = Date.now() - startTime;

    expect(duration).toBeLessThan(100); // Should be immediate
  });

  it('should enforce limits', async () => {
    const bulkhead = new ThreadPoolBulkhead({ service: 2 });

    let concurrent = 0;
    let maxConcurrent = 0;

    const promises = Array(10).fill(0).map(() =>
      bulkhead.execute('service', async () => {
        concurrent++;
        maxConcurrent = Math.max(maxConcurrent, concurrent);
        await sleep(100);
        concurrent--;
      })
    );

    await Promise.all(promises);

    expect(maxConcurrent).toBe(2);
  });
});
```

### 3. Capacity Planning

```javascript
class CapacityPlanner {
  static calculate(config) {
    // Input:
    // - expectedRPS: Expected requests per second
    // - avgDurationMs: Average request duration
    // - targetUtilization: Target utilization (e.g., 0.7 for 70%)
    
    const { expectedRPS, avgDurationMs, targetUtilization = 0.7 } = config;
    
    // Little's Law: L = λ * W
    // L = concurrent requests needed
    // λ = arrival rate (RPS)
    // W = average time in system (duration)
    
    const avgDurationSec = avgDurationMs / 1000;
    const concurrentNeeded = expectedRPS * avgDurationSec;
    
    // Add headroom for bursts
    const withHeadroom = concurrentNeeded / targetUtilization;
    
    return {
      recommended: Math.ceil(withHeadroom),
      minimum: Math.ceil(concurrentNeeded),
      analysis: {
        expectedRPS,
        avgDurationSec,
        concurrentNeeded,
        targetUtilization,
        headroom: withHeadroom - concurrentNeeded
      }
    };
  }
}

// Example
const sizing = CapacityPlanner.calculate({
  expectedRPS: 10,        // 10 requests per second
  avgDurationMs: 2000,    // 2 seconds average
  targetUtilization: 0.7  // 70% target utilization
});

console.log(sizing);
// {
//   recommended: 29,
//   minimum: 20,
//   analysis: { ... }
// }
```

### 4. Graceful Degradation

```javascript
class GracefulBulkhead {
  constructor(config) {
    this.pool = pLimit(config.maxConcurrent);
    this.maxQueue = config.maxQueue || config.maxConcurrent * 2;
    this.fallback = config.fallback;
  }

  async execute(fn) {
    // Check queue depth
    if (this.pool.pendingCount > this.maxQueue) {
      // Queue too long - use fallback
      if (this.fallback) {
        console.log('Queue full, using fallback');
        return await this.fallback();
      }
      
      throw new Error('Service at capacity');
    }

    // Execute normally
    return this.pool(fn);
  }
}

// Usage with cache fallback
const bulkhead = new GracefulBulkhead({
  maxConcurrent: 10,
  maxQueue: 20,
  fallback: async () => {
    // Return cached result
    return getCachedResult();
  }
});
```

---

## Conclusion

### Key Takeaways

1. **Bulkheads Prevent Cascading Failures**: Isolate resources so one service can't take down others
2. **Essential for AI Systems**: GPU memory, API quotas, and costs must be controlled
3. **Multiple Types**: Thread pools, connection pools, GPU memory, API quotas
4. **Combine with Other Patterns**: Works with circuit breakers, retry, backpressure
5. **Monitor Closely**: Track utilization, queue depth, rejections
6. **Test Thoroughly**: Verify isolation under load
7. **Plan Capacity**: Use Little's Law and headroom calculations

### When to Use Bulkheads

✅ **Use Bulkheads For:**
- Shared expensive resources (GPUs, API quotas)
- Multi-tenant systems
- Services with different priorities
- Cost control and budget enforcement
- Systems with variable workload patterns

❌ **Don't Use Bulkheads For:**
- Single-service applications
- Unlimited resources
- Services that never compete for resources
- Development/testing environments (usually)

### ROI Summary

**Investment:**
- Implementation: 3-5 days = $4K
- Testing: 2 days = $2K
- Monitoring: 1 day = $1K
- **Total: $7K**

**Returns (Annual):**
- Prevented outages: 4 incidents × $500K = $2M
- Cost control (prevented overuse): $600K
- Premium customer retention: $400K
- **Total: $3M/year**

**ROI: 42,857% first year**

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
