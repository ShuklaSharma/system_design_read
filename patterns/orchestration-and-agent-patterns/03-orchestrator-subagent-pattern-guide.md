# Orchestrator–Subagent Pattern for AI Applications

> AI Engineering Series | March 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [How the Pattern Works](#how-the-pattern-works)
3. [Core Components](#core-components)
4. [Building an Orchestrator–Subagent System](#building-an-orchestratorsubagent-system)
5. [Subagent Specialisation Patterns](#subagent-specialisation-patterns)
6. [Task Decomposition Strategies](#task-decomposition-strategies)
7. [Execution Models](#execution-models)
8. [AI-Specific Use Cases](#ai-specific-use-cases)
9. [Advanced Patterns](#advanced-patterns)
10. [Integration with Other Patterns](#integration-with-other-patterns)
11. [Monitoring & Observability](#monitoring--observability)
12. [Production Best Practices](#production-best-practices)
13. [Common Mistakes](#common-mistakes)
14. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is the Orchestrator–Subagent Pattern?

The **Orchestrator–Subagent Pattern** is a multi-agent architecture where a central **Orchestrator** agent receives a complex task, breaks it into focused sub-tasks, delegates each to a **specialised Subagent**, collects results, and synthesises a final answer.

The Orchestrator handles *what* needs doing and *in what order*. Subagents handle *how* to do their specific piece — each with its own model, tools, system prompt, and context.

**Analogy:** A general contractor building a house. They don't pour concrete, wire electricity, or install plumbing themselves. They decompose the project, assign each trade to a specialist, check progress, and hand the finished house to the client.

---

### The Problem Without This Pattern

```
WITHOUT Orchestrator–Subagent:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Task: "Research our top 3 competitors, analyse pricing,
       check recent news, and write an executive summary."

Single LLM approach:
  → Massive prompt with all instructions at once
  → Context window fills up mid-task; model forgets early steps
  → No specialisation — same model does research AND writing
  → Everything is sequential; can't parallelise research steps
  → If step 3 fails, restart from scratch
  → No audit trail of which step produced which output

Result:
  - Shallow analysis (context limits depth per topic)
  - Slow (sequential, one model)
  - Fragile (one failure = total failure)
  - No partial recovery
```

### The Solution With Orchestrator–Subagent

```
WITH Orchestrator–Subagent:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Orchestrator creates plan:

  Group 1 (parallel) ──────────────────────────────────────
    t1: Research CompA  → Research Subagent
    t2: Research CompB  → Research Subagent  ← all 3 run at once
    t3: Research CompC  → Research Subagent
  ───────────────────────────────────────────────────────────
  Group 2 (after group 1)
    t4: Price analysis  → Analysis Subagent  ← uses t1+t2+t3
    t5: News summary    → Research Subagent  ← runs parallel with t4
  ───────────────────────────────────────────────────────────
  Group 3 (after group 2)
    t6: Exec summary    → Writer Subagent    ← uses t4+t5
  ───────────────────────────────────────────────────────────

Result:
  - Deep analysis (each subagent has focused context)
  - ~3x faster (parallel research in group 1)
  - Specialised quality per task type
  - Retry individual steps on failure, not the whole task
  - Full audit trail per subagent
```

---

### When to Use It

| Use It When | Skip It When |
|---|---|
| Task has 3+ distinct, separable steps | Task is atomic — can't meaningfully split |
| Steps need different specialisations | One model handles everything well |
| Steps can run in parallel | Latency budget is extremely tight |
| Task exceeds one context window | Coordination overhead dominates task time |
| Partial failure recovery matters | Steps share so much state isolation is impossible |

---

## How the Pattern Works

### The Full Flow

```
User Request
     │
     ▼
┌─────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                      │
│                                                      │
│  1. PLAN      decompose task → ordered sub-tasks    │
│  2. ASSIGN    match each sub-task to a subagent     │
│  3. EXECUTE   dispatch (sequential or parallel)     │
│  4. MONITOR   track progress, handle failures       │
│  5. COLLECT   gather all subagent results           │
│  6. SYNTHESISE combine into final answer            │
└──────────────────────┬──────────────────────────────┘
                       │  dispatches sub-tasks
           ┌───────────┼────────────┐
           ▼           ▼            ▼
     ┌──────────┐ ┌──────────┐ ┌──────────┐
     │ Research │ │ Analysis │ │  Writer  │
     │ Subagent │ │ Subagent │ │ Subagent │
     └────┬─────┘ └────┬─────┘ └────┬─────┘
          │            │             │
     [web search] [data tools] [formatting]
          │            │             │
          └────────────▼─────────────┘
                  Results → Orchestrator
                       │
                       ▼
               Final Synthesised Answer
```

### Step-by-Step Walkthrough

```
Task: "Write a competitive analysis of cloud AI platforms for our CTO"

── ORCHESTRATOR PLANNING ────────────────────────────────────────────

Orchestrator produces this plan:

{
  "goal": "Competitive analysis: cloud AI platforms for CTO",
  "sub_tasks": [
    {"id":"t1","type":"research","instruction":"Research AWS AI/ML services and pricing","group":1},
    {"id":"t2","type":"research","instruction":"Research Azure AI/ML services and pricing","group":1},
    {"id":"t3","type":"research","instruction":"Research GCP AI/ML services and pricing","group":1},
    {"id":"t4","type":"analysis","instruction":"Compare the three platforms","depends_on":["t1","t2","t3"],"group":2},
    {"id":"t5","type":"writer","instruction":"Write executive report for CTO","depends_on":["t4"],"group":3}
  ]
}

── GROUP 1: Parallel research (~30s, not 90s) ───────────────────────

Research Subagent A → AWS findings  → JSON summary
Research Subagent B → Azure findings → JSON summary
Research Subagent C → GCP findings  → JSON summary

── GROUP 2: Analysis (receives all three summaries) ─────────────────

Analysis Subagent → structured comparison JSON

── GROUP 3: Writing (receives structured analysis) ──────────────────

Writer Subagent → polished markdown report

── ORCHESTRATOR SYNTHESIS ───────────────────────────────────────────

Quality check → assemble → return final answer to user
```

---

## Core Components

```
Orchestrator–Subagent System
├── Orchestrator Agent      plans, assigns, monitors, synthesises
├── Subagent Registry       catalogue of available specialised agents
├── SubTask Model           structured unit of work with metadata
├── Execution Engine        parallelism + dependency tracking
├── Context Passer          shares prior results between dependent tasks
├── Retry Handler           retries failed tasks with exponential backoff
└── Execution Tracer        full audit log of every step
```

---

## Building an Orchestrator–Subagent System

### Complete End-to-End Implementation

```python
# pip install anthropic

import asyncio
import json
import time
import hashlib
from typing import Optional, Any
from dataclasses import dataclass, field
from enum import Enum
import anthropic


# ═══════════════════════════════════════════════════════════════════
# DATA MODELS
# ═══════════════════════════════════════════════════════════════════

class TaskStatus(Enum):
    PENDING   = "pending"
    RUNNING   = "running"
    DONE      = "done"
    FAILED    = "failed"

@dataclass
class SubTask:
    """A single unit of delegated work."""
    id:              str
    type:            str          # matches a registered subagent name
    instruction:     str          # self-contained instruction for the subagent
    context:         dict  = field(default_factory=dict)
    depends_on:      list  = field(default_factory=list)  # task IDs to wait for
    parallel_group:  int   = 1    # tasks with same group number run concurrently
    max_retries:     int   = 2
    timeout_seconds: int   = 120
    status:          TaskStatus    = TaskStatus.PENDING
    result:          Optional[Any] = None
    error:           Optional[str] = None
    attempts:        int           = 0

@dataclass
class AgentResult:
    """Output returned by a subagent."""
    task_id:  str
    success:  bool
    output:   Any
    metadata: dict = field(default_factory=dict)

@dataclass
class ExecutionPlan:
    """The orchestrator's decomposed plan."""
    plan_id:               str
    goal:                  str
    sub_tasks:             list[SubTask]
    synthesis_instruction: str = "Combine all agent results into a final answer."


# ═══════════════════════════════════════════════════════════════════
# BASE SUBAGENT
# ═══════════════════════════════════════════════════════════════════

class BaseSubagent:
    """All subagents inherit from this. Override execute()."""
    name:        str = "base"
    description: str = "Base subagent"
    model:       str = "claude-sonnet-4-6"
    SYSTEM:      str = ""

    def __init__(self):
        self.client = anthropic.Anthropic()

    async def execute(self, task: SubTask) -> AgentResult:
        raise NotImplementedError

    def _llm(self, system: str, user: str, max_tokens: int = 1500) -> str:
        resp = self.client.messages.create(
            model=self.model,
            max_tokens=max_tokens,
            system=system,
            messages=[{"role": "user", "content": user}],
        )
        return resp.content[0].text

    def _parse_json(self, text: str) -> dict:
        """Strip markdown fences and parse JSON."""
        clean = text.replace("```json", "").replace("```", "").strip()
        return json.loads(clean)


# ═══════════════════════════════════════════════════════════════════
# BUILT-IN SPECIALISED SUBAGENTS
# ═══════════════════════════════════════════════════════════════════

class ResearchSubagent(BaseSubagent):
    """Researches topics and returns structured JSON summaries."""
    name        = "research"
    description = "Researches topics and returns structured JSON findings"
    SYSTEM      = """You are an expert research analyst.
Research the given topic and return ONLY valid JSON (no markdown fences):
{
  "topic":        "<topic>",
  "key_findings": ["finding 1", ...],
  "data_points":  {"metric": "value"},
  "confidence":   0.0-1.0,
  "gaps":         ["what is still unknown"]
}"""

    async def execute(self, task: SubTask) -> AgentResult:
        prompt = f"Research this topic:\n\n{task.instruction}"
        if task.context.get("prior_results"):
            prompt += f"\n\nRelevant prior context:\n{json.dumps(task.context['prior_results'], indent=2)}"
        try:
            raw    = self._llm(self.SYSTEM, prompt, max_tokens=1500)
            result = self._parse_json(raw)
            return AgentResult(task.id, True, result, {"agent": self.name})
        except json.JSONDecodeError as e:
            return AgentResult(task.id, False, None, {"error": f"JSON parse: {e}"})


class AnalysisSubagent(BaseSubagent):
    """Synthesises multiple research inputs into comparative analysis."""
    name        = "analysis"
    description = "Analyses and compares multiple data inputs"
    SYSTEM      = """You are a senior business analyst.
Synthesise the provided data and return ONLY valid JSON:
{
  "summary":         "<3-sentence executive summary>",
  "key_comparisons": [{"dimension": "Price", "winner": "A", "notes": "..."}],
  "strengths":       {"entity": ["strength 1", ...]},
  "weaknesses":      {"entity": ["weakness 1", ...]},
  "recommendation":  "<clear recommendation>",
  "confidence":      0.0-1.0
}"""

    async def execute(self, task: SubTask) -> AgentResult:
        prior  = task.context.get("prior_results", {})
        prompt = f"Analyse goal: {task.instruction}\n\nDATA:\n{json.dumps(prior, indent=2)}"
        try:
            raw    = self._llm(self.SYSTEM, prompt, max_tokens=2000)
            result = self._parse_json(raw)
            return AgentResult(task.id, True, result, {"agent": self.name})
        except json.JSONDecodeError as e:
            return AgentResult(task.id, False, None, {"error": f"JSON parse: {e}"})


class WriterSubagent(BaseSubagent):
    """Produces polished written output from structured data."""
    name        = "writer"
    description = "Writes polished reports, summaries, and documents"
    SYSTEM      = """You are a professional business writer.
Transform structured analysis into a polished markdown document.
- Lead with an executive summary (3-5 sentences)
- Use clear ## section headers
- Include a markdown comparison table where relevant
- End with numbered, actionable recommendations
- Professional but readable tone. No padding."""

    async def execute(self, task: SubTask) -> AgentResult:
        prior  = task.context.get("prior_results", {})
        prompt = f"Write: {task.instruction}\n\nDATA:\n{json.dumps(prior, indent=2)}"
        text   = self._llm(self.SYSTEM, prompt, max_tokens=3000)
        return AgentResult(task.id, True, text,
                           {"agent": self.name, "word_count": len(text.split())})


class CodeSubagent(BaseSubagent):
    """Writes, reviews, and explains code."""
    name        = "code"
    description = "Writes production-ready code with tests and documentation"
    SYSTEM      = """You are a senior software engineer.
Write clean, well-documented, production-ready code with:
type hints, docstrings, error handling, and inline comments."""

    async def execute(self, task: SubTask) -> AgentResult:
        prompt = f"{task.instruction}\nContext: {json.dumps(task.context) if task.context else 'None'}"
        text   = self._llm(self.SYSTEM, prompt, max_tokens=2500)
        return AgentResult(task.id, True, text, {"agent": self.name})


class ValidationSubagent(BaseSubagent):
    """Quality-checks outputs from other subagents."""
    name        = "validation"
    description = "Reviews outputs for quality, accuracy, and completeness"
    SYSTEM      = """You are a QA reviewer.
Evaluate the provided content and return ONLY valid JSON:
{
  "passed":          true/false,
  "quality_score":   0-10,
  "issues":          ["issue 1", ...],
  "suggestions":     ["suggestion 1", ...],
  "approved_output": "<corrected version if minor issues, else null>"
}"""

    async def execute(self, task: SubTask) -> AgentResult:
        content = task.context.get("content_to_validate", "")
        prompt  = f"Criteria: {task.instruction}\n\nContent:\n{content}"
        try:
            raw    = self._llm(self.SYSTEM, prompt, max_tokens=1000)
            result = self._parse_json(raw)
            return AgentResult(task.id, True, result, {"agent": self.name})
        except json.JSONDecodeError as e:
            return AgentResult(task.id, False, None, {"error": str(e)})


# ═══════════════════════════════════════════════════════════════════
# SUBAGENT REGISTRY
# ═══════════════════════════════════════════════════════════════════

class SubagentRegistry:
    """Catalogue of all available subagents."""

    def __init__(self):
        self._agents: dict[str, BaseSubagent] = {}
        for cls in [ResearchSubagent, AnalysisSubagent,
                    WriterSubagent, CodeSubagent, ValidationSubagent]:
            inst = cls()
            self._agents[inst.name] = inst

    def register(self, agent: BaseSubagent):
        self._agents[agent.name] = agent

    def get(self, name: str) -> Optional[BaseSubagent]:
        return self._agents.get(name)

    def describe(self) -> list[dict]:
        return [{"name": a.name, "description": a.description}
                for a in self._agents.values()]


# ═══════════════════════════════════════════════════════════════════
# ORCHESTRATOR
# ═══════════════════════════════════════════════════════════════════

class Orchestrator:
    """
    Central coordinator.
    1. Decomposes tasks into sub-tasks via LLM planning
    2. Executes with parallelism + dependency tracking
    3. Retries failed sub-tasks with exponential backoff
    4. Synthesises a final answer from all results
    """

    PLANNER_SYSTEM = """You are a task orchestrator. Break complex tasks into focused sub-tasks.
Return ONLY valid JSON (no markdown fences):
{
  "goal": "<concise goal>",
  "synthesis_instruction": "<how to combine all results>",
  "sub_tasks": [
    {
      "id": "t1",
      "type": "<agent_type>",
      "instruction": "<specific, self-contained instruction>",
      "depends_on": [],
      "parallel_group": 1
    }
  ]
}

Rules:
- Tasks in the SAME parallel_group run CONCURRENTLY
- A task only starts when ALL its depends_on tasks are done
- Keep each instruction focused on ONE thing
- Maximise concurrency — independent tasks get the same group number
- Limit to 8 sub-tasks maximum"""

    SYNTHESISER_SYSTEM = """You are a senior analyst. Combine outputs from multiple
specialised agents into one coherent, polished final answer.
Resolve contradictions, acknowledge any gaps, and present unified conclusions."""

    def __init__(self, registry: SubagentRegistry):
        self.registry = registry
        self.client   = anthropic.Anthropic()
        self.trace:   list[dict] = []

    def _log(self, event: str, **kwargs):
        entry = {"ts": round(time.time(), 2), "event": event, **kwargs}
        self.trace.append(entry)
        snippet = {k: str(v)[:80] for k, v in kwargs.items()}
        print(f"  [{event}] {snippet}")

    # ── Plan ────────────────────────────────────────────────────────

    def plan(self, task: str) -> ExecutionPlan:
        self._log("planning", task=task[:100])

        prompt = (f"Create an execution plan for:\n\nTASK: {task}\n\n"
                  f"Available agents:\n{json.dumps(self.registry.describe(), indent=2)}")

        resp = self.client.messages.create(
            model="claude-sonnet-4-6", max_tokens=2000,
            system=self.PLANNER_SYSTEM,
            messages=[{"role": "user", "content": prompt}],
        )
        data = json.loads(
            resp.content[0].text.replace("```json", "").replace("```", "").strip()
        )

        sub_tasks = [
            SubTask(
                id             = st["id"],
                type           = st["type"],
                instruction    = st["instruction"],
                depends_on     = st.get("depends_on", []),
                parallel_group = st.get("parallel_group", 1),
            )
            for st in data["sub_tasks"]
        ]

        plan = ExecutionPlan(
            plan_id               = hashlib.md5(task.encode()).hexdigest()[:8],
            goal                  = data["goal"],
            sub_tasks             = sub_tasks,
            synthesis_instruction = data.get("synthesis_instruction", "Combine all results."),
        )
        self._log("plan_ready", plan_id=plan.plan_id, num_tasks=len(sub_tasks))
        return plan

    # ── Execute ─────────────────────────────────────────────────────

    async def execute(self, plan: ExecutionPlan) -> dict[str, AgentResult]:
        results: dict[str, AgentResult] = {}

        groups: dict[int, list[SubTask]] = {}
        for task in plan.sub_tasks:
            groups.setdefault(task.parallel_group, []).append(task)

        for group_num in sorted(groups):
            group_tasks = groups[group_num]
            self._log("group_start", group=group_num, tasks=[t.id for t in group_tasks])

            # Inject prior results into each task's context
            prior = {tid: r.output for tid, r in results.items() if r.success}
            for task in group_tasks:
                task.context["prior_results"] = {
                    dep: prior[dep] for dep in task.depends_on if dep in prior
                }

            # Run all tasks in this group concurrently
            group_results = await asyncio.gather(
                *[self._run_with_retry(task) for task in group_tasks],
                return_exceptions=True,
            )
            for task, result in zip(group_tasks, group_results):
                if isinstance(result, Exception):
                    result = AgentResult(task.id, False, None, {"error": str(result)})
                results[task.id] = result
                self._log("task_done", task_id=task.id, type=task.type, success=result.success)

        return results

    async def _run_with_retry(self, task: SubTask) -> AgentResult:
        agent = self.registry.get(task.type)
        if not agent:
            return AgentResult(task.id, False, None,
                               {"error": f"No agent for type '{task.type}'"})

        for attempt in range(task.max_retries + 1):
            task.attempts = attempt + 1
            self._log("attempt", task_id=task.id, attempt=attempt + 1)

            try:
                result = await asyncio.wait_for(
                    agent.execute(task), timeout=task.timeout_seconds
                )
                if result.success:
                    return result
                self._log("attempt_failed", task_id=task.id,
                          error=str(result.metadata.get("error", ""))[:80])

            except asyncio.TimeoutError:
                self._log("timeout", task_id=task.id)
                if attempt == task.max_retries:
                    return AgentResult(task.id, False, None, {"error": "Timeout"})

            except Exception as e:
                if attempt == task.max_retries:
                    return AgentResult(task.id, False, None, {"error": str(e)})

            await asyncio.sleep(2 ** attempt)   # exponential backoff

        return AgentResult(task.id, False, None, {"error": "Max retries exceeded"})

    # ── Synthesise ──────────────────────────────────────────────────

    def synthesise(self, plan: ExecutionPlan, results: dict[str, AgentResult]) -> str:
        self._log("synthesising", num_results=len(results))

        successes = {tid: r.output for tid, r in results.items() if r.success}
        failures  = [tid for tid, r in results.items() if not r.success]

        prompt = (f"GOAL: {plan.goal}\n"
                  f"INSTRUCTION: {plan.synthesis_instruction}\n\n"
                  f"AGENT OUTPUTS:\n{json.dumps(successes, indent=2)}\n"
                  + (f"\nFAILED TASKS (acknowledge gaps): {failures}" if failures else "")
                  + "\n\nProduce the final synthesised answer now.")

        resp = self.client.messages.create(
            model="claude-sonnet-4-6", max_tokens=4000,
            system=self.SYNTHESISER_SYSTEM,
            messages=[{"role": "user", "content": prompt}],
        )
        return resp.content[0].text

    # ── Main Entry ──────────────────────────────────────────────────

    async def run(self, task: str) -> dict:
        print(f"\n{'='*60}\nORCHESTRATOR: {task[:80]}\n{'='*60}")
        self.trace = []
        start = time.time()

        plan         = self.plan(task)
        results      = await self.execute(plan)
        final_answer = self.synthesise(plan, results)
        elapsed      = round(time.time() - start, 2)
        successes    = sum(1 for r in results.values() if r.success)

        print(f"\n✅ {successes}/{len(results)} tasks succeeded in {elapsed}s")
        return {
            "goal":            plan.goal,
            "final_answer":    final_answer,
            "plan_id":         plan.plan_id,
            "tasks_total":     len(results),
            "tasks_succeeded": successes,
            "tasks_failed":    len(results) - successes,
            "elapsed_seconds": elapsed,
            "trace":           self.trace,
        }


# ── Usage ────────────────────────────────────────────────────────────

async def main():
    registry     = SubagentRegistry()
    orchestrator = Orchestrator(registry)

    result = await orchestrator.run(
        "Research AWS, Azure, and GCP AI/ML platforms. "
        "Compare their LLM services and pricing. "
        "Write an executive summary recommending the best for a mid-size startup."
    )

    print("\n" + "="*60 + "\nFINAL ANSWER\n" + "="*60)
    print(result["final_answer"])

asyncio.run(main())
```

---

## Subagent Specialisation Patterns

### Pattern 1: Stateless Subagent (Recommended Default)

All context passed explicitly via `task.context`. No memory between calls. Easiest to test, scale, and reason about.

```python
class StatelessSubagent(BaseSubagent):
    """Everything the agent needs comes from task.instruction + task.context."""
    name = "stateless_example"

    async def execute(self, task: SubTask) -> AgentResult:
        prompt = task.instruction
        if task.context.get("prior_results"):
            prompt += f"\n\nPrior results:\n{json.dumps(task.context['prior_results'])}"
        output = self._llm(self.SYSTEM, prompt)
        return AgentResult(task.id, True, output)
```

---

### Pattern 2: Tool-Equipped Subagent

Subagents that invoke real tools (web search, DB queries, code execution):

```python
class ToolSubagent(BaseSubagent):
    """Research subagent with real web search via Anthropic tool use."""
    name = "research_with_tools"

    TOOLS = [{
        "name":        "web_search",
        "description": "Search the web for current information",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"]
        }
    }]

    async def execute(self, task: SubTask) -> AgentResult:
        messages = [{"role": "user", "content": task.instruction}]
        all_text = []

        for _ in range(5):   # max 5 tool calls per subagent run
            resp = self.client.messages.create(
                model=self.model, max_tokens=1024,
                tools=self.TOOLS, messages=messages,
            )
            for block in resp.content:
                if hasattr(block, "text"):
                    all_text.append(block.text)

            if resp.stop_reason != "tool_use":
                break

            tool_results = []
            for block in resp.content:
                if block.type == "tool_use":
                    search_out = await self._search(block.input["query"])
                    tool_results.append({
                        "type":        "tool_result",
                        "tool_use_id": block.id,
                        "content":     search_out,
                    })
            messages.append({"role": "assistant", "content": resp.content})
            messages.append({"role": "user",      "content": tool_results})

        return AgentResult(task.id, True, "\n".join(all_text), {"agent": self.name})

    async def _search(self, query: str) -> str:
        # Integrate with Serper, Tavily, SerpAPI, etc.
        return f"[Search results for '{query}' — plug in your search API here]"
```

---

### Pattern 3: Validator Gate

Insert a validation subagent between steps to catch poor outputs before they propagate:

```python
async def execute_with_validation(task: SubTask, registry: SubagentRegistry,
                                   criteria: str) -> AgentResult:
    """Run a task and validate its output before passing downstream."""
    # 1. Run the primary task
    primary = registry.get(task.type)
    result  = await primary.execute(task)
    if not result.success:
        return result

    # 2. Validate the output
    val_task = SubTask(
        id="val_" + task.id, type="validation",
        instruction=criteria,
        context={"content_to_validate": str(result.output)},
    )
    validator  = registry.get("validation")
    val_result = await validator.execute(val_task)

    if val_result.success and val_result.output:
        score = val_result.output.get("quality_score", 10)
        if not val_result.output.get("passed") and score < 6:
            # Quality too low — retry with feedback
            task.instruction += f"\n\nFeedback from prior attempt: {val_result.output.get('issues')}"
            return await primary.execute(task)

    return result
```

---

## Task Decomposition Strategies

### Sequential Decomposition

Each step's output is the next step's input.

```
Task: "Debug error → fix code → write tests → update docs"

t1: analyse_error  (group 1)
t2: fix_code       (group 2, depends_on: [t1])
t3: write_tests    (group 3, depends_on: [t2])
t4: update_docs    (group 4, depends_on: [t3])

Timeline: ──t1──t2──t3──t4──
```

---

### Parallel Decomposition

Independent tasks run simultaneously.

```
Task: "Research 4 competitors"

t1: research_A  (group 1)
t2: research_B  (group 1)   ← all 4 run at once
t3: research_C  (group 1)
t4: research_D  (group 1)
t5: compare     (group 2, depends_on: [t1,t2,t3,t4])

Timeline: ──[t1,t2,t3,t4 in parallel]──t5──
           4x faster than sequential!
```

---

### Map-Reduce Decomposition

Fan out to many subagents for the same operation, then fan in to one aggregator:

```python
async def map_reduce(items: list[str], operation: str,
                     orchestrator: Orchestrator) -> AgentResult:
    # MAP: one sub-task per item, all in parallel (group 1)
    map_tasks = [
        SubTask(id=f"map_{i}", type="research",
                instruction=f"{operation}: {item}", parallel_group=1)
        for i, item in enumerate(items)
    ]
    map_results = await asyncio.gather(
        *[orchestrator._run_with_retry(t) for t in map_tasks]
    )

    # REDUCE: single aggregation task (group 2)
    reduce_task = SubTask(
        id="reduce", type="analysis",
        instruction=f"Synthesise all {len(items)} results into one unified analysis",
        context={"prior_results": {
            f"item_{i}": r.output for i, r in enumerate(map_results) if r.success
        }},
        parallel_group=2,
    )
    reducer = orchestrator.registry.get("analysis")
    return await reducer.execute(reduce_task)
```

---

### Hierarchical Decomposition

Sub-tasks that are themselves too complex get spawned as mini-orchestrations:

```python
class HierarchicalOrchestrator(Orchestrator):
    MAX_DEPTH = 3

    async def run_deep(self, task: str, depth: int = 0) -> dict:
        if depth >= self.MAX_DEPTH:
            agent = self.registry.get("research")
            leaf  = SubTask(id="leaf", type="research", instruction=task)
            r     = await agent.execute(leaf)
            return {"final_answer": r.output, "depth": depth}

        result = await self.run(task)

        # Retry any failed task as its own mini-orchestration
        for entry in self.trace:
            if entry.get("event") == "task_done" and not entry.get("success"):
                sub = await self.run_deep(
                    f"Alternative for: {entry.get('task_id', '')}",
                    depth + 1
                )
                result.setdefault("sub_orchestrations", []).append(sub)

        return result
```

---

## Execution Models

```
SEQUENTIAL (safe, slow)
─────────────────────────────────────────────
t1 ──► t2 ──► t3 ──► t4
Use when: strict data dependencies between every step.

PARALLEL (fast, independent)
─────────────────────────────────────────────
t1 ─┐
t2 ─┼──► t5 (aggregation)
t3 ─┘
Use when: tasks are independent.

HYBRID (most real-world tasks)
─────────────────────────────────────────────
Group 1 (parallel):    t1, t2, t3
           ↓
Group 2 (parallel):    t4, t5
           ↓
Group 3 (sequential):  t6 ──► t7
Use when: mixed — some tasks independent, some sequential.
```

### Realistic Plan Example

```python
# "Write a product launch blog post"
sub_tasks = [
    # Group 1 — independent (run in parallel)
    SubTask("t1", "research", "Research target audience pain points",   parallel_group=1),
    SubTask("t2", "research", "Research competitor positioning",        parallel_group=1),
    SubTask("t3", "research", "Gather product feature highlights",      parallel_group=1),

    # Group 2 — depends on group 1
    SubTask("t4", "analysis", "Craft messaging strategy",
            depends_on=["t1","t2","t3"], parallel_group=2),
    SubTask("t5", "research", "Find 3 customer success angles",
            depends_on=["t1","t3"],      parallel_group=2),

    # Group 3 — write the post
    SubTask("t6", "writer",     "Write 1000-word launch blog post",
            depends_on=["t4","t5"],      parallel_group=3),

    # Group 4 — validate before shipping
    SubTask("t7", "validation", "Check tone, accuracy, and SEO quality",
            depends_on=["t6"],           parallel_group=4),
]
```

---

## AI-Specific Use Cases

### Use Case 1: Automated Research Report

```python
async def research_report(topic: str) -> str:
    registry = SubagentRegistry()
    orch     = Orchestrator(registry)
    result   = await orch.run(
        f"Create a comprehensive research report on: {topic}\n"
        f"- Research at least 3 distinct angles\n"
        f"- Include data points and statistics\n"
        f"- Structure as an executive report with summary\n"
        f"- End with specific, actionable conclusions"
    )
    return result["final_answer"]
```

---

### Use Case 2: Software Development Pipeline

```python
async def dev_pipeline(feature_request: str) -> dict:
    """Analyse → Design → Code → Tests → Docs (with parallelism where possible)."""
    registry = SubagentRegistry()
    orch     = Orchestrator(registry)
    return await orch.run(
        f"Complete this software development task end-to-end:\n\n"
        f"FEATURE: {feature_request}\n\n"
        f"Deliverables:\n"
        f"1. Requirements analysis\n"
        f"2. Architecture design\n"
        f"3. Production-ready implementation code\n"
        f"4. Unit tests covering key scenarios\n"
        f"5. API documentation"
    )
```

---

### Use Case 3: Multi-Format Content Pipeline

```python
async def content_pipeline(topic: str, formats: list[str]) -> dict:
    """Research once → produce all formats simultaneously."""
    registry    = SubagentRegistry()
    orch        = Orchestrator(registry)
    formats_str = ", ".join(formats)
    return await orch.run(
        f"Research this topic then produce standalone content in ALL formats: {formats_str}\n"
        f"TOPIC: {topic}\n"
        f"Each format must be self-contained, tailored to its conventions, "
        f"and consistent in facts across all formats."
    )

result = asyncio.run(content_pipeline(
    topic   = "Why AI agents are replacing traditional automation",
    formats = ["blog post", "Twitter/X thread", "executive summary", "FAQ"]
))
```

---

### Use Case 4: Competitive Intelligence

```python
async def competitive_intel(our_product: str, competitors: list[str]) -> dict:
    registry  = SubagentRegistry()
    orch      = Orchestrator(registry)
    comp_list = ", ".join(competitors)
    return await orch.run(
        f"Competitive intelligence analysis.\n"
        f"OUR PRODUCT: {our_product}\n"
        f"COMPETITORS: {comp_list}\n\n"
        f"For EACH competitor research: features, pricing, recent news, sentiment.\n"
        f"Then produce: side-by-side comparison, our strengths/gaps, "
        f"and strategic roadmap recommendations."
    )
```

---

## Advanced Patterns

### Dynamic Subagent Spawning

Generate a custom subagent on the fly for task types not in the registry:

```python
class DynamicOrchestrator(Orchestrator):

    async def _run_with_retry(self, task: SubTask) -> AgentResult:
        if not self.registry.get(task.type):
            await self._spawn_agent(task.type)
        return await super()._run_with_retry(task)

    async def _spawn_agent(self, specialisation: str):
        resp = self.client.messages.create(
            model="claude-sonnet-4-6", max_tokens=400,
            messages=[{"role": "user",
                       "content": f"Write a concise system prompt for an AI agent "
                                  f"specialised in: {specialisation}. Return only the prompt text."}]
        )
        system_prompt = resp.content[0].text

        class DynamicAgent(BaseSubagent):
            name        = specialisation[:30]
            description = f"Dynamically spawned: {specialisation}"
            async def execute(self_inner, task: SubTask) -> AgentResult:
                out = self_inner._llm(system_prompt, task.instruction, max_tokens=1500)
                return AgentResult(task.id, True, out, {"agent": self_inner.name})

        self.registry.register(DynamicAgent())
        self._log("agent_spawned", type=specialisation)
```

---

### Self-Healing Orchestrator

Diagnose failures and pick smart recovery strategies:

```python
class SelfHealingOrchestrator(Orchestrator):

    async def _run_with_retry(self, task: SubTask) -> AgentResult:
        result = await super()._run_with_retry(task)
        if not result.success:
            return await self._recover(task, result.metadata.get("error", "unknown"))
        return result

    async def _recover(self, task: SubTask, error: str) -> AgentResult:
        prompt = (f"A subagent failed. Choose a recovery strategy.\n"
                  f"Task type: {task.type}\n"
                  f"Instruction: {task.instruction[:150]}\n"
                  f"Error: {error}\n\n"
                  f"Strategies: retry_same | retry_simpler | use_fallback | skip\n"
                  f'Return JSON: {{"strategy":"<s>","reason":"<r>","simplified_instruction":"<if simpler>"}}')

        resp     = self.client.messages.create(
            model="claude-sonnet-4-6", max_tokens=200,
            messages=[{"role": "user", "content": prompt}]
        )
        decision = json.loads(resp.content[0].text.replace("```json","").replace("```","").strip())
        strategy = decision.get("strategy", "skip")
        self._log("recovery", task_id=task.id, strategy=strategy)

        if strategy == "retry_simpler":
            task.instruction = decision.get("simplified_instruction", task.instruction)
            return await super()._run_with_retry(task)
        elif strategy == "use_fallback":
            task.type = "research" if task.type != "research" else "analysis"
            return await super()._run_with_retry(task)
        else:  # skip
            return AgentResult(task.id, True,
                               f"[Step skipped: {error}]", {"skipped": True})
```

---

### A/B Orchestration

Test two decomposition strategies simultaneously and pick the winner:

```python
async def ab_orchestrate(task: str, registry: SubagentRegistry) -> dict:
    orch_a = Orchestrator(registry)
    orch_b = Orchestrator(registry)

    result_a, result_b = await asyncio.gather(
        orch_a.run(f"Strategy A (depth-first): {task}"),
        orch_b.run(f"Strategy B (breadth-first): {task}"),
    )

    client    = anthropic.Anthropic()
    judge_rsp = client.messages.create(
        model="claude-sonnet-4-6", max_tokens=200,
        messages=[{"role": "user", "content":
            f"Goal: '{task}'\n\nA:\n{result_a['final_answer'][:400]}\n\n"
            f"B:\n{result_b['final_answer'][:400]}\n\n"
            f'Which is better? Return JSON: {{"winner":"A"or"B","reason":"<1 sentence>"}}'}]
    )
    d      = json.loads(judge_rsp.content[0].text.replace("```json","").replace("```","").strip())
    winner = result_a if d["winner"] == "A" else result_b
    winner.update({"ab_winner": d["winner"], "ab_reason": d["reason"]})
    return winner
```

---

## Integration with Other Patterns

### Orchestrator + Router

Use a Router to select the best subagent for each sub-task:

```python
class RouterAwareOrchestrator(Orchestrator):
    def __init__(self, registry, router):
        super().__init__(registry)
        self.router = router

    async def _run_with_retry(self, task: SubTask) -> AgentResult:
        decision  = self.router.route(task.instruction)
        type_map  = {"code": "code", "complex": "analysis", "simple": "research"}
        task.type = type_map.get(decision.route.value, task.type)
        return await super()._run_with_retry(task)
```

---

### Orchestrator + RAG

Ground each subagent's answers in retrieved documents:

```python
class RAGSubagent(BaseSubagent):
    name = "rag_research"

    def __init__(self, rag_pipeline):
        super().__init__()
        self.rag = rag_pipeline

    async def execute(self, task: SubTask) -> AgentResult:
        rag_result  = self.rag.query(task.instruction)
        augmented   = (f"{task.instruction}\n\n"
                       f"Grounding context:\n{rag_result['answer']}\n"
                       f"Sources: {rag_result['sources']}")
        output = self._llm(self.SYSTEM, augmented, max_tokens=1500)
        return AgentResult(task.id, True, output,
                           {"agent": self.name, "sources": rag_result["sources"]})
```

---

### Orchestrator + Circuit Breaker

Protect each subagent type with its own circuit breaker:

```python
class ResilientOrchestrator(Orchestrator):
    def __init__(self, registry):
        super().__init__(registry)
        self._breakers: dict[str, dict] = {}

    def _breaker(self, name: str) -> dict:
        return self._breakers.setdefault(
            name, {"failures": 0, "open": False, "opened_at": 0}
        )

    async def _run_with_retry(self, task: SubTask) -> AgentResult:
        b = self._breaker(task.type)
        if b["open"]:
            if time.time() - b["opened_at"] < 60:
                return AgentResult(task.id, False, None,
                                   {"error": f"Circuit OPEN for '{task.type}'"})
            b["open"] = False   # half-open

        result = await super()._run_with_retry(task)
        if result.success:
            b["failures"] = 0
        else:
            b["failures"] += 1
            if b["failures"] >= 3:
                b.update({"open": True, "opened_at": time.time()})
                self._log("circuit_opened", agent=task.type)
        return result
```

---

## Monitoring & Observability

```python
def analyse_trace(trace: list[dict]) -> dict:
    """Extract key performance metrics from an orchestration trace."""
    times, attempts, failures = {}, {}, 0

    for e in trace:
        tid = e.get("task_id")
        if not tid:
            continue
        if e["event"] == "attempt":
            times.setdefault(tid, {})["start"] = e["ts"]
            attempts[tid] = e.get("attempt", 1)
        elif e["event"] == "task_done":
            times.setdefault(tid, {})["end"] = e["ts"]
            if not e.get("success"):
                failures += 1

    durations  = {
        tid: round(t["end"] - t["start"], 2)
        for tid, t in times.items() if "start" in t and "end" in t
    }
    bottleneck = max(durations, key=durations.get) if durations else None

    return {
        "task_durations_sec":  durations,
        "bottleneck_task":     bottleneck,
        "bottleneck_sec":      durations.get(bottleneck, 0),
        "total_retries":       sum(v - 1 for v in attempts.values() if v > 1),
        "total_failures":      failures,
    }
```

### Key Metrics to Track

| Metric | Why It Matters | Alert Threshold |
|---|---|---|
| **Plan generation time** | Slow planning wastes latency before work starts | > 15s |
| **Subagent success rate** | Identifies unreliable agent types | < 90% per type |
| **Parallel speedup** | Validates that concurrency is working | < 1.5x vs sequential |
| **Retry rate** | High retries signal fragile subagents | > 20% of calls |
| **Total pipeline cost** | Multi-agent pipelines are inherently more expensive | Set per-task budget |
| **Synthesis quality score** | End-to-end quality (LLM-as-judge) | < 7 / 10 |

---

## Production Best Practices

### 1. Write Atomic, Self-Contained Instructions

```
❌ BAD:  "Research the topic, then analyse it, then write a summary."
         Three tasks in one — can't retry or parallelise cleanly.

✅ GOOD: "Research [specific topic] and return key findings as JSON."
         One task, one output format, one responsibility.
```

---

### 2. Always Define Output Schemas

```python
# Every subagent should document and validate its output format.
# Validate before returning AgentResult — silent schema violations
# break downstream agents in ways that are very hard to debug.
OUTPUT_SCHEMA = {
    "type": "object",
    "required": ["result", "confidence"],
    "properties": {
        "result":     {"type": "string"},
        "confidence": {"type": "number"},
    }
}
```

---

### 3. Set Timeouts at Every Level

```python
TIMEOUTS = {
    "plan_generation":     30,   # orchestrator planning LLM call
    "subagent_research":  120,   # research agents need time to think
    "subagent_analysis":   90,
    "subagent_writer":     60,
    "subagent_validation": 45,
    "synthesis":           60,
    "total_pipeline":     600,   # hard cap on the entire orchestration
}
```

---

### 4. Design for Partial Success

```python
def synthesise_with_gaps(successes: dict, failures: list) -> str:
    """Never fail the whole orchestration because one step failed."""
    gap_note = ""
    if failures:
        gap_note = (f"\n\n⚠️ Note: The following steps could not be completed "
                    f"and are not reflected in this output: {failures}")
    return synthesise_results(successes) + gap_note
```

---

### 5. Test Each Subagent in Isolation First

```python
async def test_subagent(agent: BaseSubagent, test_cases: list[dict]) -> float:
    passed = 0
    for tc in test_cases:
        task   = SubTask(id="test", type=agent.name, instruction=tc["input"])
        result = await agent.execute(task)
        if tc["expect"].lower() in str(result.output).lower():
            passed += 1
        else:
            print(f"  FAIL: '{tc['input'][:50]}' → {str(result.output)[:80]}")
    accuracy = passed / len(test_cases)
    print(f"{agent.name}: {accuracy:.0%} ({passed}/{len(test_cases)})")
    return accuracy
```

---

### 6. Cap Plan Size and Recursion Depth

```python
MAX_SUBTASKS       = 8     # keep plans focused; decompose in phases if needed
MAX_RECURSION_DEPTH = 3    # hard cap for hierarchical orchestration
MAX_TOTAL_TOKENS   = 200_000  # track cumulative tokens; abort if approaching cap
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| Instructions not atomic | Subagent tries to do too much, quality degrades | One task = one clear responsibility and output format |
| No output schema | Downstream subagent receives unexpected format | Define and validate JSON schemas per subagent |
| Missing timeouts | One hanging subagent freezes the whole pipeline | Set `timeout_seconds` on every SubTask |
| No dependency modelling | Subagent runs before its input is ready | Always set `depends_on` for data dependencies |
| Everything sequential | Misses 3–5x parallelism speedup | Assign same `parallel_group` to independent tasks |
| No partial failure handling | One failed step kills entire orchestration | Synthesise from partial results, acknowledge gaps |
| No subagent isolation tests | Integration failures impossible to debug | Unit-test each subagent before wiring together |
| Unbounded recursion | Stack overflow on complex hierarchical plans | Always enforce `MAX_DEPTH` |
| No cost cap | Multi-agent pipelines can be very expensive | Track tokens per subagent, set total budget |
| Plans too large | 15-step plan produces chaos, not clarity | Limit to ≤ 8 sub-tasks; decompose across phases |

---

## Interview Cheat Sheet

**Q: What is the Orchestrator–Subagent pattern?**
A: A multi-agent architecture where a central Orchestrator receives a complex task, decomposes it into focused sub-tasks, delegates each to a specialised Subagent, monitors execution, and synthesises a final answer. The Orchestrator handles *what* and *when*; Subagents handle *how*.

**Q: What problems does it solve vs a single LLM?**
A: Three main ones — context window limits (complex multi-step tasks overflow one model's window, each subagent gets focused bounded context), specialisation (different steps need different models or system prompts), and parallelism (independent sub-tasks run concurrently, giving 3–5x speedup).

**Q: How do you decompose a task into sub-tasks?**
A: Use an LLM planner that outputs a JSON plan. Each sub-task has an `id`, `type` (which agent), `instruction` (atomic and self-contained), `depends_on` (task IDs to wait for), and `parallel_group` (tasks with the same group number run concurrently). Limit plans to ≤ 8 tasks.

**Q: How do you handle subagent failures?**
A: Multiple layers. Retry with exponential backoff (2–3 attempts). Diagnose the failure: retry with a simpler instruction, swap to a fallback agent type, or skip and acknowledge the gap. Always design the synthesis step to work with partial results — never fail the whole orchestration for one step.

**Q: When should tasks run in parallel vs sequential?**
A: Sequential when task B needs task A's output as input (strict data dependency). Parallel when tasks are independent — e.g., researching 5 competitors simultaneously. Assign the same `parallel_group` to concurrent tasks; increment the group number when prior tasks must finish first.

**Q: What is a task contract and why does it matter?**
A: A formal JSON schema defining a subagent's expected input and output format. Without it, one subagent's unexpected output silently breaks the next. Contracts make each subagent independently testable and debuggable.

**Q: How does Orchestrator–Subagent integrate with RAG?**
A: Subagents can run RAG retrieval before generating — querying a vector DB for relevant context and passing it to the LLM. The orchestrator treats a RAG-equipped subagent identically to any other, just registering it under a different name.

**Q: How does it differ from ReAct?**
A: ReAct is a single agent in a reasoning loop — think, act, observe, repeat. Orchestrator–Subagent is a multi-agent system — the orchestrator plans up front and delegates to independent specialists. ReAct suits single-thread exploration; Orchestrator–Subagent suits complex tasks requiring parallelism and specialisation across distinct domains.

**Q: How do you evaluate orchestration quality?**
A: Track subagent success rate (>90%), parallel group speedup (>1.5x vs sequential), retry rate per agent type (<20%), and final output quality using an LLM-as-judge rubric. Also monitor total cost per run — multi-agent pipelines are more expensive than single-model calls and need a per-task budget.

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team