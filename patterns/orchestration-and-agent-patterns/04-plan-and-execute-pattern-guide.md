# Plan-and-Execute Pattern for AI Applications

> AI Engineering Series | March 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [How the Pattern Works](#how-the-pattern-works)
3. [Core Components](#core-components)
4. [Building a Plan-and-Execute System](#building-a-plan-and-execute-system)
5. [Planning Strategies](#planning-strategies)
6. [Execution Engine Variants](#execution-engine-variants)
7. [Plan Revision & Replanning](#plan-revision--replanning)
8. [AI-Specific Use Cases](#ai-specific-use-cases)
9. [Advanced Patterns](#advanced-patterns)
10. [Integration with Other Patterns](#integration-with-other-patterns)
11. [Monitoring & Observability](#monitoring--observability)
12. [Production Best Practices](#production-best-practices)
13. [Common Mistakes](#common-mistakes)
14. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is the Plan-and-Execute Pattern?

The **Plan-and-Execute Pattern** separates an agent's work into two distinct phases:

1. **Plan Phase** — A reasoning model surveys the entire task, thinks through the full approach, and produces an explicit, structured plan *before any action is taken*.
2. **Execute Phase** — A (potentially different, cheaper) executor works through the plan step-by-step, using tools, querying APIs, and producing results.

The key insight: **reasoning about what to do and actually doing it are different cognitive tasks**, and conflating them in one pass leads to worse outcomes — the model acts before it has fully thought through the consequences.

**Analogy:** A surgeon who studies the scans, reviews the patient's history, and plans the procedure in a pre-op meeting *before* picking up a scalpel. Acting without planning risks mistakes that are expensive or impossible to undo.

---

### Plan-and-Execute vs ReAct

```
ReAct (interleaved think-act):
─────────────────────────────────────────────────────
Thought → Act → Observe → Thought → Act → Observe → ...

  Each thought only sees results so far.
  The model can change direction at any step.
  Great for: exploration, open-ended tasks
  Weakness: loses sight of overall strategy mid-task
            can't parallelise steps
            hard to audit what was planned vs improvised

Plan-and-Execute (separate phases):
─────────────────────────────────────────────────────
[Full reasoning about whole task]
         ↓
PLAN: Step 1 → Step 2 → Step 3 → ... Step N
         ↓
Execute Step 1 → Execute Step 2 → ... → Execute Step N
         ↓
Synthesise results → Final answer

  Planner sees the complete picture before acting.
  Executor focuses only on carrying out each step.
  Great for: multi-step tasks, irreversible actions, auditability
  Weakness: rigid if environment changes mid-execution
            upfront planning cost
```

---

### When to Use This Pattern

| Use It When | Skip It When |
|---|---|
| Task has 3+ sequential steps | Task is a single-step lookup |
| Some actions are irreversible (send email, write file, call API) | Fully exploratory — plan is unknowable upfront |
| You need an audit trail of intent vs action | Task is simple enough for direct ReAct |
| Planner and executor can be different models | Low latency is critical (adds one LLM call) |
| Context window would overflow in a single-pass agent | Task changes so dynamically that any plan is stale immediately |
| Compliance or safety review of the plan is required | — |

---

## How the Pattern Works

### The Two-Phase Flow

```
User Goal
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                      PLANNER                              │
│                                                           │
│  Input: full task description + available tools           │
│  Model: large, capable (claude-sonnet-4-6 / gpt-4o)      │
│                                                           │
│  Output:                                                  │
│  {                                                        │
│    "goal": "...",                                         │
│    "reasoning": "...",          ← why this plan           │
│    "steps": [                                             │
│      {"id": 1, "action": "web_search",                   │
│       "input": "Python asyncio best practices"},         │
│      {"id": 2, "action": "read_file",                    │
│       "input": "current_code.py"},                       │
│      {"id": 3, "action": "write_file",                   │
│       "input": "refactored_code.py", "depends_on": [1,2]}│
│    ],                                                     │
│    "success_criteria": "...",                             │
│    "risks": ["..."]                                       │
│  }                                                        │
└────────────────────────┬─────────────────────────────────┘
                         │  structured plan
                         ▼
┌──────────────────────────────────────────────────────────┐
│                      EXECUTOR                             │
│                                                           │
│  Input: one step at a time + results of prior steps       │
│  Model: smaller/cheaper (claude-haiku / gpt-3.5)         │
│                                                           │
│  For each step:                                           │
│    1. Read the step instruction                           │
│    2. Call the specified tool / action                    │
│    3. Capture the result                                  │
│    4. Check: should plan be revised?                      │
│    5. Pass result to next step                            │
└────────────────────────┬─────────────────────────────────┘
                         │  all step results
                         ▼
┌──────────────────────────────────────────────────────────┐
│                    SYNTHESISER                            │
│  Combine all step results → final answer                  │
└──────────────────────────────────────────────────────────┘
```

---

### Step-by-Step Walkthrough

```
Goal: "Research best practices for async Python, then refactor our codebase"

─── PLAN PHASE ──────────────────────────────────────────────────────

Planner receives:
  - The goal
  - Available tools: [web_search, read_file, write_file, run_tests]
  - Constraints: [don't break existing tests, keep public API stable]

Planner produces:
  Step 1: web_search("Python asyncio best practices 2026")
          ← Gather current knowledge first
  Step 2: read_file("src/main.py")
          ← Understand what we're working with
  Step 3: read_file("tests/test_main.py")
          ← Know what must not break
  Step 4: write_file("src/main.py", refactored_content)
          ← Apply improvements from steps 1+2 without breaking step 3
  Step 5: run_tests("tests/")
          ← Verify the refactor didn't break anything
  
  Reasoning: "Must understand current state (steps 2-3) before changing it
              (step 4). Must verify correctness (step 5) before declaring done."
  
  Risk: "If tests fail in step 5, need to revert step 4 changes."

─── EXECUTE PHASE ────────────────────────────────────────────────────

Executor runs Step 1: calls web_search → gets results → stores
Executor runs Step 2: calls read_file → gets code → stores
Executor runs Step 3: calls read_file → gets tests → stores
Executor runs Step 4: calls write_file with improvements based on 1+2+3
Executor runs Step 5: calls run_tests → ✅ all passing

─── SYNTHESISE ───────────────────────────────────────────────────────

Final answer: "Refactored main.py. Applied: async context managers, 
               proper cancellation handling, task groups. All 47 tests passing."
```

---

## Core Components

```
Plan-and-Execute System
├── Planner         large model, full task view, produces structured plan
├── Plan            structured list of steps with actions, inputs, dependencies
├── Executor        works through steps sequentially, calls tools
├── Tool Registry   catalogue of actions the executor can invoke
├── State Store     accumulates results from each step
├── Replan Trigger  detects when reality diverges from plan
└── Synthesiser     combines all step results into final answer
```

---

## Building a Plan-and-Execute System

### Complete Implementation

```python
# pip install anthropic

import asyncio
import json
import time
from typing import Optional, Any, Callable
from dataclasses import dataclass, field
from enum import Enum
import anthropic


# ═══════════════════════════════════════════════════════════════════
# DATA MODELS
# ═══════════════════════════════════════════════════════════════════

class StepStatus(Enum):
    PENDING   = "pending"
    RUNNING   = "running"
    DONE      = "done"
    FAILED    = "failed"
    SKIPPED   = "skipped"

@dataclass
class PlanStep:
    """A single step in the execution plan."""
    id:          int
    description: str               # human-readable description
    action:      str               # tool/action name to call
    input:       Any               # input to the action
    depends_on:  list[int] = field(default_factory=list)  # step IDs
    optional:    bool      = False # if True, failure doesn't abort plan
    timeout:     int       = 60    # seconds
    # Runtime state
    status:      StepStatus    = StepStatus.PENDING
    result:      Optional[Any] = None
    error:       Optional[str] = None
    started_at:  Optional[float] = None
    finished_at: Optional[float] = None

@dataclass
class Plan:
    """The full plan produced by the planner."""
    goal:             str
    reasoning:        str               # planner's thinking
    steps:            list[PlanStep]
    success_criteria: str
    risks:            list[str] = field(default_factory=list)
    created_at:       float     = field(default_factory=time.time)
    version:          int       = 1     # increments on replan

@dataclass
class ExecutionState:
    """Accumulated results across all executed steps."""
    plan:           Plan
    step_results:   dict[int, Any]  = field(default_factory=dict)
    current_step:   int             = 0
    completed:      bool            = False
    replans:        int             = 0
    started_at:     float           = field(default_factory=time.time)


# ═══════════════════════════════════════════════════════════════════
# TOOL REGISTRY
# ═══════════════════════════════════════════════════════════════════

class ToolRegistry:
    """
    Registry of tools available to the executor.
    Each tool is a callable: (input, state) -> result
    """

    def __init__(self):
        self._tools: dict[str, Callable] = {}
        self._schemas: dict[str, dict]   = {}

    def register(self, name: str, fn: Callable, schema: dict = None):
        self._tools[name]   = fn
        self._schemas[name] = schema or {"description": name}

    async def call(self, action: str, input_data: Any,
                   state: ExecutionState) -> Any:
        if action not in self._tools:
            raise ValueError(f"Unknown action: '{action}'. "
                             f"Available: {list(self._tools.keys())}")
        tool = self._tools[action]
        if asyncio.iscoroutinefunction(tool):
            return await tool(input_data, state)
        return tool(input_data, state)

    def describe(self) -> list[dict]:
        return [
            {"name": name, **schema}
            for name, schema in self._schemas.items()
        ]


# ═══════════════════════════════════════════════════════════════════
# PLANNER
# ═══════════════════════════════════════════════════════════════════

class Planner:
    """
    Uses a powerful LLM to think through the entire task and produce
    a structured execution plan before any action is taken.
    """

    SYSTEM_PROMPT = """You are a meticulous task planner. Your job is to think through
a complex task completely and produce a precise, ordered execution plan.

CRITICAL RULES:
1. Produce the COMPLETE plan upfront — do not act, only plan
2. Each step must specify exactly ONE action from the available tools
3. Express all dependencies explicitly via depends_on
4. Identify risks and how to handle them
5. Define clear success criteria

Return ONLY valid JSON (no markdown fences):
{
  "goal": "<concise restatement of the goal>",
  "reasoning": "<your full thinking: why this sequence, what could go wrong>",
  "steps": [
    {
      "id": 1,
      "description": "<human-readable description>",
      "action": "<tool_name>",
      "input": "<input to the tool — string, dict, or value>",
      "depends_on": [],
      "optional": false
    }
  ],
  "success_criteria": "<how to know the task is complete>",
  "risks": ["<risk 1>", "<risk 2>"]
}"""

    def __init__(self, api_key: str, model: str = "claude-sonnet-4-6"):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.model  = model

    def create_plan(self, goal: str, tools: list[dict],
                    constraints: list[str] = None) -> Plan:
        """Think through the full task and return a structured plan."""

        constraints_str = ""
        if constraints:
            constraints_str = "\n\nCONSTRAINTS:\n" + "\n".join(f"- {c}" for c in constraints)

        prompt = f"""Create a complete execution plan for this task.

GOAL: {goal}

AVAILABLE TOOLS:
{json.dumps(tools, indent=2)}{constraints_str}

Think through the full task. What needs to happen first? What depends on what?
What could go wrong? Produce the complete plan now."""

        response = self.client.messages.create(
            model=self.model,
            max_tokens=2000,
            system=self.SYSTEM_PROMPT,
            messages=[{"role": "user", "content": prompt}],
        )

        raw  = response.content[0].text.replace("```json", "").replace("```", "").strip()
        data = json.loads(raw)

        steps = [
            PlanStep(
                id          = s["id"],
                description = s["description"],
                action      = s["action"],
                input       = s["input"],
                depends_on  = s.get("depends_on", []),
                optional    = s.get("optional", False),
            )
            for s in data["steps"]
        ]

        return Plan(
            goal             = data["goal"],
            reasoning        = data["reasoning"],
            steps            = steps,
            success_criteria = data["success_criteria"],
            risks            = data.get("risks", []),
        )

    def revise_plan(self, original_plan: Plan, failed_step: PlanStep,
                    error: str, completed_results: dict) -> Plan:
        """
        Called when execution diverges from the plan.
        Produces a revised plan for remaining steps.
        """
        completed_str = json.dumps({
            f"step_{k}": str(v)[:200]
            for k, v in completed_results.items()
        }, indent=2)

        prompt = f"""The execution plan hit an obstacle. Revise the remaining steps.

ORIGINAL GOAL: {original_plan.goal}

ORIGINAL PLAN REASONING: {original_plan.reasoning}

COMPLETED STEPS SO FAR:
{completed_str}

FAILED STEP: Step {failed_step.id} — "{failed_step.description}"
ERROR: {error}

Produce a REVISED plan for the REMAINING work only.
Keep the same JSON format. Number remaining steps starting from {failed_step.id}."""

        response = self.client.messages.create(
            model=self.model,
            max_tokens=2000,
            system=self.SYSTEM_PROMPT,
            messages=[{"role": "user", "content": prompt}],
        )

        raw  = response.content[0].text.replace("```json", "").replace("```", "").strip()
        data = json.loads(raw)

        steps = [
            PlanStep(
                id          = s["id"],
                description = s["description"],
                action      = s["action"],
                input       = s["input"],
                depends_on  = s.get("depends_on", []),
                optional    = s.get("optional", False),
            )
            for s in data["steps"]
        ]

        revised = Plan(
            goal             = original_plan.goal,
            reasoning        = data["reasoning"],
            steps            = steps,
            success_criteria = original_plan.success_criteria,
            risks            = data.get("risks", []),
            version          = original_plan.version + 1,
        )
        return revised


# ═══════════════════════════════════════════════════════════════════
# EXECUTOR
# ═══════════════════════════════════════════════════════════════════

class Executor:
    """
    Executes a plan step-by-step. Uses a smaller/cheaper model
    for each individual step — no big-picture reasoning needed here.
    """

    STEP_SYSTEM = """You are a precise task executor. You are given one step to complete.
Call exactly the tool specified. Do not improvise, skip, or add steps.
If the tool call succeeds, report the result clearly.
If something is ambiguous, use the most reasonable interpretation."""

    def __init__(self, api_key: str,
                 tool_registry: ToolRegistry,
                 model: str = "claude-haiku-4-5-20251001"):
        self.client   = anthropic.Anthropic(api_key=api_key)
        self.tools    = tool_registry
        self.model    = model

    async def execute_step(self, step: PlanStep,
                           state: ExecutionState) -> Any:
        """Execute a single plan step, calling the appropriate tool."""
        step.status     = StepStatus.RUNNING
        step.started_at = time.time()

        print(f"  → Step {step.id}: {step.description}")

        # Build context from completed prior steps
        prior_context = ""
        if step.depends_on:
            relevant = {
                f"step_{sid}": str(state.step_results.get(sid, "not available"))[:300]
                for sid in step.depends_on
            }
            prior_context = f"\n\nResults from prior steps:\n{json.dumps(relevant, indent=2)}"

        # Execute the tool directly
        try:
            result = await asyncio.wait_for(
                self.tools.call(step.action, step.input, state),
                timeout=step.timeout,
            )
            step.status      = StepStatus.DONE
            step.result      = result
            step.finished_at = time.time()
            print(f"     ✅ Done ({round(step.finished_at - step.started_at, 1)}s)")
            return result

        except asyncio.TimeoutError:
            step.status = StepStatus.FAILED
            step.error  = f"Timeout after {step.timeout}s"
            step.finished_at = time.time()
            print(f"     ⏱ Timeout")
            raise

        except Exception as e:
            step.status      = StepStatus.FAILED
            step.error       = str(e)
            step.finished_at = time.time()
            print(f"     ❌ Failed: {e}")
            raise

    async def execute_plan(self, plan: Plan,
                           on_step_complete: Callable = None) -> ExecutionState:
        """Execute all steps in the plan, respecting dependencies."""
        state = ExecutionState(plan=plan)

        for step in plan.steps:
            # Wait for dependencies
            for dep_id in step.depends_on:
                dep_step = next((s for s in plan.steps if s.id == dep_id), None)
                if dep_step and dep_step.status == StepStatus.FAILED:
                    if step.optional:
                        step.status = StepStatus.SKIPPED
                        print(f"  ⊘ Step {step.id} skipped (dependency failed, step is optional)")
                        continue
                    else:
                        raise RuntimeError(
                            f"Step {step.id} cannot run: dependency step {dep_id} failed."
                        )

            try:
                result = await self.execute_step(step, state)
                state.step_results[step.id] = result
                state.current_step = step.id

                if on_step_complete:
                    on_step_complete(step, result, state)

            except Exception as e:
                state.step_results[step.id] = {"error": str(e)}
                if not step.optional:
                    raise

        state.completed = True
        return state


# ═══════════════════════════════════════════════════════════════════
# SYNTHESISER
# ═══════════════════════════════════════════════════════════════════

class Synthesiser:
    """Combines all step results into a coherent final answer."""

    SYSTEM = """You are a results synthesiser. Combine outputs from multiple
execution steps into one coherent, complete final answer.
Be concise. Acknowledge any steps that failed or were skipped."""

    def __init__(self, api_key: str, model: str = "claude-sonnet-4-6"):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.model  = model

    def synthesise(self, state: ExecutionState) -> str:
        results_str = json.dumps({
            f"step_{sid}_({plan_step.description})": str(v)[:500]
            for sid, v in state.step_results.items()
            for plan_step in [next(s for s in state.plan.steps if s.id == sid)]
        }, indent=2)

        failed = [s for s in state.plan.steps if s.status == StepStatus.FAILED]
        skipped = [s for s in state.plan.steps if s.status == StepStatus.SKIPPED]

        prompt = f"""Synthesise these execution results into a final answer.

ORIGINAL GOAL: {state.plan.goal}
SUCCESS CRITERIA: {state.plan.success_criteria}

STEP RESULTS:
{results_str}

{f"FAILED STEPS: {[s.description for s in failed]}" if failed else ""}
{f"SKIPPED STEPS: {[s.description for s in skipped]}" if skipped else ""}

Produce the final synthesised answer."""

        response = self.client.messages.create(
            model=self.model, max_tokens=2000,
            system=self.SYSTEM,
            messages=[{"role": "user", "content": prompt}],
        )
        return response.content[0].text


# ═══════════════════════════════════════════════════════════════════
# PLAN-AND-EXECUTE AGENT
# ═══════════════════════════════════════════════════════════════════

class PlanAndExecuteAgent:
    """
    Full Plan-and-Execute agent.
    1. Planner reasons about the full task → structured plan
    2. Executor works through steps, calling tools
    3. Replan if execution diverges from plan
    4. Synthesiser combines all results → final answer
    """

    MAX_REPLANS = 2

    def __init__(self, api_key: str):
        self.tool_registry = ToolRegistry()
        self.planner       = Planner(api_key)
        self.executor      = Executor(api_key, self.tool_registry)
        self.synthesiser   = Synthesiser(api_key)
        self._register_default_tools()

    def _register_default_tools(self):
        """Register built-in tools. Add your own via register_tool()."""

        async def think(input_data, state):
            """No-op tool for intermediate reasoning steps."""
            return {"thought": input_data}

        async def summarise(input_data, state):
            """Summarise prior step results."""
            prior = list(state.step_results.values())
            return {"summary": f"Summarised {len(prior)} prior results: {str(prior)[:200]}"}

        self.tool_registry.register("think", think,
            {"description": "Record a reasoning step without calling external tools"})
        self.tool_registry.register("summarise", summarise,
            {"description": "Summarise results accumulated so far"})

    def register_tool(self, name: str, fn: Callable, schema: dict):
        """Register a custom tool for use in plans."""
        self.tool_registry.register(name, fn, schema)

    async def run(self, goal: str, constraints: list[str] = None,
                  allow_replan: bool = True) -> dict:
        """Full pipeline: plan → execute → (replan if needed) → synthesise."""
        start = time.time()
        print(f"\n{'='*60}")
        print(f"PLAN-AND-EXECUTE: {goal[:80]}")
        print(f"{'='*60}")

        # ── Phase 1: Plan ────────────────────────────────────────
        print("\n[PHASE 1] PLANNING...")
        tools = self.tool_registry.describe()
        plan  = self.planner.create_plan(goal, tools, constraints)

        print(f"  Plan created: {len(plan.steps)} steps (v{plan.version})")
        print(f"  Reasoning: {plan.reasoning[:120]}...")
        for step in plan.steps:
            deps = f" (after {step.depends_on})" if step.depends_on else ""
            print(f"    Step {step.id}: [{step.action}] {step.description}{deps}")

        # ── Phase 2: Execute ─────────────────────────────────────
        print(f"\n[PHASE 2] EXECUTING {len(plan.steps)} STEPS...")
        replans = 0
        state   = None

        while replans <= self.MAX_REPLANS:
            try:
                state = await self.executor.execute_plan(plan)
                break   # success — exit retry loop

            except RuntimeError as e:
                if not allow_replan or replans >= self.MAX_REPLANS:
                    raise

                # Find the failed step
                failed_step = next(
                    (s for s in plan.steps if s.status == StepStatus.FAILED), None
                )
                if not failed_step:
                    raise

                replans += 1
                print(f"\n[REPLAN {replans}] Revising plan after failure...")
                plan = self.planner.revise_plan(
                    plan, failed_step, failed_step.error or str(e),
                    state.step_results if state else {}
                )
                print(f"  Revised plan: {len(plan.steps)} remaining steps")

        # ── Phase 3: Synthesise ──────────────────────────────────
        print(f"\n[PHASE 3] SYNTHESISING RESULTS...")
        final_answer = self.synthesiser.synthesise(state)

        elapsed    = round(time.time() - start, 2)
        completed  = sum(1 for s in plan.steps if s.status == StepStatus.DONE)
        failed     = sum(1 for s in plan.steps if s.status == StepStatus.FAILED)
        skipped    = sum(1 for s in plan.steps if s.status == StepStatus.SKIPPED)

        print(f"\n✅ Done in {elapsed}s | "
              f"{completed} completed, {failed} failed, {skipped} skipped, "
              f"{replans} replan(s)")

        return {
            "goal":           goal,
            "final_answer":   final_answer,
            "plan_version":   plan.version,
            "steps_total":    len(plan.steps),
            "steps_done":     completed,
            "steps_failed":   failed,
            "steps_skipped":  skipped,
            "replans":        replans,
            "elapsed_seconds": elapsed,
            "plan_reasoning": plan.reasoning,
            "plan_risks":     plan.risks,
        }
```

---

### Wiring in Real Tools

```python
import httpx
import subprocess
from pathlib import Path

def build_production_agent(api_key: str) -> PlanAndExecuteAgent:
    agent = PlanAndExecuteAgent(api_key)

    # ── Web Search ───────────────────────────────────────────────
    async def web_search(query: str, state) -> dict:
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                "https://api.search.example.com/search",
                params={"q": query, "key": "YOUR_SEARCH_KEY"},
                timeout=10,
            )
            return resp.json()

    agent.register_tool("web_search", web_search, {
        "description": "Search the web for current information",
        "input": "search query string"
    })

    # ── File Operations ──────────────────────────────────────────
    def read_file(path: str, state) -> str:
        return Path(path).read_text(encoding="utf-8")

    def write_file(input_data: dict, state) -> dict:
        path    = input_data["path"]
        content = input_data["content"]
        Path(path).write_text(content, encoding="utf-8")
        return {"written": path, "bytes": len(content)}

    agent.register_tool("read_file",  read_file,  {"description": "Read a file from disk", "input": "file path"})
    agent.register_tool("write_file", write_file, {"description": "Write content to a file", "input": '{"path": "...", "content": "..."}'})

    # ── Code Execution ───────────────────────────────────────────
    async def run_command(cmd: str, state) -> dict:
        proc = await asyncio.create_subprocess_shell(
            cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        stdout, stderr = await asyncio.wait_for(proc.communicate(), timeout=30)
        return {
            "returncode": proc.returncode,
            "stdout":     stdout.decode()[:2000],
            "stderr":     stderr.decode()[:500],
        }

    agent.register_tool("run_command", run_command, {
        "description": "Execute a shell command and return output",
        "input": "shell command string"
    })

    # ── LLM Sub-task ─────────────────────────────────────────────
    def llm_task(input_data: dict, state) -> str:
        """Call an LLM for a specific sub-task within a step."""
        import anthropic as _a
        c    = _a.Anthropic(api_key=api_key)
        resp = c.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=1000,
            messages=[{"role": "user", "content": input_data["prompt"]}]
        )
        return resp.content[0].text

    agent.register_tool("llm_task", llm_task, {
        "description": "Delegate a sub-task to an LLM",
        "input": '{"prompt": "...", "context": "..."}'
    })

    return agent


# ── Run It ───────────────────────────────────────────────────────
async def main():
    import os
    agent  = build_production_agent(os.environ["ANTHROPIC_API_KEY"])
    result = await agent.run(
        goal = "Search for the top 3 Python async patterns in 2026, "
               "summarise them, and write a markdown cheatsheet to async_cheatsheet.md",
        constraints = [
            "Only use authoritative sources (official docs, major engineering blogs)",
            "Keep the cheatsheet under 500 words",
        ]
    )
    print("\n" + "="*60 + "\nFINAL ANSWER\n" + "="*60)
    print(result["final_answer"])

asyncio.run(main())
```

---

## Planning Strategies

### Strategy 1: Constraint-First Planning

Give the planner explicit constraints before it writes a single step. Forces the model to bake constraints into the plan rather than hoping the executor respects them.

```python
def plan_with_constraints(agent: PlanAndExecuteAgent, goal: str) -> Plan:
    constraints = [
        "Never delete files — only create or modify",
        "Do not send any emails or messages without explicit user confirmation",
        "All web requests must use HTTPS",
        "Maximum 5 external API calls per plan",
        "Estimated cost must not exceed $0.10",
    ]
    return agent.planner.create_plan(
        goal        = goal,
        tools       = agent.tool_registry.describe(),
        constraints = constraints,
    )
```

---

### Strategy 2: Risk-Weighted Planning

Ask the planner to explicitly reason about risks and produce mitigation steps:

```python
RISK_AWARE_PLANNER_SYSTEM = """You are a meticulous, risk-aware task planner.
For each plan, you must:
1. Identify all IRREVERSIBLE actions (file deletions, API calls, emails sent)
2. Place read/validate steps BEFORE irreversible write steps
3. Add an explicit rollback step after any irreversible action
4. Mark optional steps as optional: true so execution can continue if they fail

Return the same JSON format, but add:
"irreversible_steps": [list of step IDs that cannot be undone],
"rollback_steps":     [{"if_step_fails": N, "run_step": M}]"""
```

---

### Strategy 3: Layered Planning (Plan + Sub-Plans)

For very complex tasks, plan at two levels — a high-level plan, then sub-plans for each complex step:

```python
async def layered_plan(agent: PlanAndExecuteAgent, goal: str) -> dict:
    """
    Level 1: High-level plan (5-8 major phases)
    Level 2: Each complex phase gets its own detailed sub-plan
    """
    # Level 1: coarse plan
    coarse_plan = agent.planner.create_plan(goal, agent.tool_registry.describe())

    # Level 2: expand complex steps into sub-plans
    detailed_steps = []
    for step in coarse_plan.steps:
        complexity = len(step.description.split())  # rough proxy
        if complexity > 10 or step.action == "llm_task":
            # Complex step — expand into sub-plan
            sub_plan = agent.planner.create_plan(
                goal   = step.description,
                tools  = agent.tool_registry.describe(),
            )
            detailed_steps.extend(sub_plan.steps)
        else:
            detailed_steps.append(step)

    coarse_plan.steps = detailed_steps
    return coarse_plan
```

---

### Strategy 4: Parallel Step Planning

Identify steps that can run concurrently and mark them for parallel execution:

```python
PARALLEL_PLANNER_SYSTEM = """You are a task planner who optimises for speed.
When steps are INDEPENDENT (output of A not needed by B), assign them
the SAME parallel_group number so they can run concurrently.

Add "parallel_group": <int> to each step.
Steps with the same group number run in PARALLEL.
Increase the group number when a synchronisation point is needed.

Example:
  Steps 1, 2, 3 are independent  → parallel_group: 1
  Step 4 needs results of 1+2+3  → parallel_group: 2
  Step 5 needs result of 4       → parallel_group: 3"""
```

---

## Execution Engine Variants

### Sequential Executor (Default — Simple and Safe)

```python
async def execute_sequential(plan: Plan, executor: Executor) -> ExecutionState:
    """Execute steps one at a time in order. Simplest and most debuggable."""
    state = ExecutionState(plan=plan)
    for step in plan.steps:
        result = await executor.execute_step(step, state)
        state.step_results[step.id] = result
    state.completed = True
    return state
```

---

### Parallel Executor (Faster — for Independent Steps)

```python
async def execute_with_parallelism(plan: Plan, executor: Executor) -> ExecutionState:
    """Execute steps concurrently where parallel_group matches."""
    state  = ExecutionState(plan=plan)
    groups: dict[int, list[PlanStep]] = {}

    for step in plan.steps:
        group = getattr(step, "parallel_group", 1)
        groups.setdefault(group, []).append(step)

    for group_num in sorted(groups):
        group_steps = groups[group_num]

        # Inject prior results into steps that depend on them
        for step in group_steps:
            step_context = {dep: state.step_results.get(dep) for dep in step.depends_on}
            step.input = _resolve_dependencies(step.input, step_context)

        # Run all steps in this group concurrently
        group_results = await asyncio.gather(
            *[executor.execute_step(step, state) for step in group_steps],
            return_exceptions=True,
        )
        for step, result in zip(group_steps, group_results):
            if isinstance(result, Exception):
                step.status = StepStatus.FAILED
                step.error  = str(result)
            else:
                state.step_results[step.id] = result

    state.completed = True
    return state


def _resolve_dependencies(input_data: Any, context: dict) -> Any:
    """Replace {step_N_result} placeholders in inputs with actual values."""
    if isinstance(input_data, str):
        for step_id, result in context.items():
            placeholder = f"{{step_{step_id}_result}}"
            if placeholder in input_data:
                input_data = input_data.replace(placeholder, str(result))
    return input_data
```

---

### Checkpoint Executor (Safe for Long-Running Tasks)

```python
import pickle
from pathlib import Path

class CheckpointExecutor(Executor):
    """
    Saves execution state after each step.
    Allows resuming from the last checkpoint if the process crashes.
    """

    def __init__(self, *args, checkpoint_dir: str = "/tmp/checkpoints", **kwargs):
        super().__init__(*args, **kwargs)
        self.checkpoint_dir = Path(checkpoint_dir)
        self.checkpoint_dir.mkdir(parents=True, exist_ok=True)

    def _save_checkpoint(self, state: ExecutionState):
        path = self.checkpoint_dir / f"plan_{state.plan.goal[:20]}.pkl"
        with open(path, "wb") as f:
            pickle.dump(state, f)

    def _load_checkpoint(self, goal: str) -> Optional[ExecutionState]:
        path = self.checkpoint_dir / f"plan_{goal[:20]}.pkl"
        if path.exists():
            with open(path, "rb") as f:
                return pickle.load(f)
        return None

    async def execute_step(self, step: PlanStep,
                           state: ExecutionState) -> Any:
        # Skip already-completed steps (resume from checkpoint)
        if step.id in state.step_results:
            print(f"  ⊘ Step {step.id} already done (from checkpoint)")
            step.status = StepStatus.DONE
            return state.step_results[step.id]

        result = await super().execute_step(step, state)
        self._save_checkpoint(state)   # checkpoint after every successful step
        return result
```

---

## Plan Revision & Replanning

### When to Replan

```
Replan Triggers:
─────────────────────────────────────────────────────────

1. HARD FAILURE
   A critical (non-optional) step fails with an error.
   → Replanner diagnoses why and redesigns remaining steps.

2. UNEXPECTED RESULT
   A step succeeds but returns data that makes later steps invalid.
   Example: "Search for competitors" returns 0 results.
   The plan assumed results existed, but now the analysis step is moot.
   → Replanner adjusts subsequent steps for the new reality.

3. SCOPE CHANGE
   Mid-execution, new information reveals the goal needs adjustment.
   → Replanner re-derives remaining steps given updated goal.

4. RESOURCE CONSTRAINT
   A tool becomes unavailable mid-execution (rate limit, outage).
   → Replanner routes around the unavailable tool.

When NOT to Replan:
─────────────────────────────────────────────────────────
  - Optional step failure (just skip it)
  - Minor result variation that doesn't affect downstream steps
  - Final synthesis step (just acknowledge gaps)
```

### Adaptive Replanning

```python
class AdaptivePlanAndExecute(PlanAndExecuteAgent):
    """
    Monitors execution in real-time and triggers replanning
    whenever the executor encounters unexpected results.
    """

    UNEXPECTED_RESULT_PROMPT = """Compare this step result against what the plan expected.
Does this result change what later steps should do?

STEP: {description}
EXPECTED: {expected}
ACTUAL RESULT: {actual}
REMAINING PLAN: {remaining}

Return JSON:
{{
  "plan_still_valid": true/false,
  "reason": "<why>",
  "suggested_adjustment": "<if not valid, what should change>"
}}"""

    async def run_adaptive(self, goal: str) -> dict:
        plan = self.planner.create_plan(goal, self.tool_registry.describe())
        state = ExecutionState(plan=plan)

        for i, step in enumerate(plan.steps):
            try:
                result = await self.executor.execute_step(step, state)
                state.step_results[step.id] = result

                # Check if result invalidates the remaining plan
                remaining = plan.steps[i + 1:]
                if remaining:
                    validity = await self._check_plan_validity(
                        step, result, remaining
                    )
                    if not validity["plan_still_valid"]:
                        print(f"  ⚠️  Replanning: {validity['reason']}")
                        plan = self.planner.revise_plan(
                            plan, step, validity["reason"], state.step_results
                        )

            except Exception as e:
                if not step.optional:
                    plan = self.planner.revise_plan(
                        plan, step, str(e), state.step_results
                    )

        state.completed = True
        return {"final_answer": self.synthesiser.synthesise(state)}

    async def _check_plan_validity(self, step: PlanStep,
                                    result: Any,
                                    remaining: list[PlanStep]) -> dict:
        prompt = self.UNEXPECTED_RESULT_PROMPT.format(
            description = step.description,
            expected    = step.input,
            actual      = str(result)[:300],
            remaining   = [s.description for s in remaining],
        )
        resp = self.planner.client.messages.create(
            model="claude-haiku-4-5-20251001", max_tokens=300,
            messages=[{"role": "user", "content": prompt}]
        )
        return json.loads(
            resp.content[0].text.replace("```json","").replace("```","").strip()
        )
```

---

## AI-Specific Use Cases

### Use Case 1: Automated Code Review and Refactor

```python
async def code_review_and_refactor(repo_path: str, language: str = "Python") -> dict:
    """
    Full plan:
    1. Read all source files
    2. Search for best practices documentation
    3. Identify issues and improvements
    4. Write refactored code
    5. Run tests to verify
    """
    agent = build_production_agent(os.environ["ANTHROPIC_API_KEY"])

    return await agent.run(
        goal = f"Review and refactor the {language} codebase at {repo_path}. "
               f"Apply current best practices. Do not break existing tests.",
        constraints = [
            "Read all files before writing any",
            "Run tests after every write operation",
            "If tests fail after a write, restore the original file",
            "Do not modify test files",
        ]
    )
```

---

### Use Case 2: Multi-Source Research Report

```python
async def research_report(topic: str, output_path: str) -> dict:
    """
    Plan:
    1. Search multiple angles of the topic in parallel
    2. Read and extract key information
    3. Cross-reference and validate key claims
    4. Write the structured report
    5. Validate report quality
    """
    agent = build_production_agent(os.environ["ANTHROPIC_API_KEY"])

    return await agent.run(
        goal = f"Research '{topic}' from multiple perspectives and write "
               f"a comprehensive, cited report to {output_path}",
        constraints = [
            "Search at least 3 distinct angles",
            "Cross-reference key statistics before including them",
            "Include inline citations for every factual claim",
            "Maximum report length: 1500 words",
        ]
    )
```

---

### Use Case 3: Data Pipeline Automation

```python
async def data_pipeline(source: str, destination: str, transform_spec: str) -> dict:
    """
    Plan:
    1. Read and validate source data schema
    2. Check destination compatibility
    3. Apply transformations
    4. Validate output against expected schema
    5. Write to destination
    6. Generate pipeline summary report
    """
    agent = build_production_agent(os.environ["ANTHROPIC_API_KEY"])

    return await agent.run(
        goal = f"Transform data from {source} to {destination} "
               f"applying: {transform_spec}",
        constraints = [
            "Validate source schema before any transformation",
            "Never overwrite destination — write to a temp file first",
            "Verify row counts match before finalising",
            "Rollback if row count mismatch > 1%",
        ]
    )
```

---

### Use Case 4: Agentic Customer Support Resolution

```python
async def resolve_support_ticket(ticket_id: str, customer_id: str) -> dict:
    """
    Plan:
    1. Fetch ticket details
    2. Fetch customer account history
    3. Search knowledge base for matching known issues
    4. Determine resolution action
    5. Apply resolution (refund, account update, etc.)
    6. Send confirmation to customer
    7. Close and categorise ticket
    """
    agent = build_production_agent(os.environ["ANTHROPIC_API_KEY"])

    return await agent.run(
        goal = f"Fully resolve support ticket {ticket_id} for customer {customer_id}",
        constraints = [
            "Read ticket and account history BEFORE taking any action",
            "Never process refunds over $500 without explicit approval",
            "Always send a confirmation message after any account change",
            "Log every action taken for audit trail",
        ]
    )
```

---

## Advanced Patterns

### Human-in-the-Loop Plan Approval

Show the plan to a human before execution — critical for irreversible actions:

```python
class HumanApprovedAgent(PlanAndExecuteAgent):
    """
    Presents the plan to a human operator for approval
    before any execution begins.
    """

    async def run(self, goal: str, constraints: list[str] = None, **kwargs) -> dict:
        # Generate plan as normal
        plan = self.planner.create_plan(goal, self.tool_registry.describe(), constraints)

        # Present for human review
        print("\n" + "="*60)
        print("PLAN FOR REVIEW (not yet executed)")
        print("="*60)
        print(f"Goal: {plan.goal}")
        print(f"Reasoning: {plan.reasoning[:200]}")
        print(f"\nSteps ({len(plan.steps)}):")
        for step in plan.steps:
            marker = "⚠️ " if step.action in ["write_file", "run_command", "send_email"] else "  "
            print(f"  {marker}Step {step.id}: [{step.action}] {step.description}")

        if plan.risks:
            print(f"\nRisks identified:")
            for risk in plan.risks:
                print(f"  ⚠️  {risk}")

        # Await approval
        approval = input("\nApprove this plan? (yes / no / edit): ").strip().lower()

        if approval == "no":
            return {"goal": goal, "final_answer": "Plan rejected by operator.", "status": "rejected"}

        if approval == "edit":
            edited_goal = input("Describe the changes you want: ")
            return await self.run(
                goal        = edited_goal,
                constraints = constraints
            )

        # Approved — execute
        state         = await self.executor.execute_plan(plan)
        final_answer  = self.synthesiser.synthesise(state)
        return {"goal": goal, "final_answer": final_answer, "status": "approved_and_executed"}
```

---

### Plan Caching

Cache plans for recurring task patterns to skip planning latency:

```python
import hashlib
import json
from pathlib import Path

class CachedPlanner(Planner):
    """Caches plans for identical goal+constraints combinations."""

    def __init__(self, *args, cache_dir: str = "/tmp/plan_cache", **kwargs):
        super().__init__(*args, **kwargs)
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)

    def _cache_key(self, goal: str, constraints: list[str] = None) -> str:
        payload = f"{goal}|{sorted(constraints or [])}"
        return hashlib.md5(payload.encode()).hexdigest()

    def create_plan(self, goal: str, tools: list[dict],
                    constraints: list[str] = None) -> Plan:
        key   = self._cache_key(goal, constraints)
        cache = self.cache_dir / f"{key}.json"

        if cache.exists():
            print("  📋 Plan loaded from cache")
            data  = json.loads(cache.read_text())
            steps = [PlanStep(**s) for s in data["steps"]]
            return Plan(
                goal=data["goal"], reasoning=data["reasoning"],
                steps=steps, success_criteria=data["success_criteria"],
                risks=data.get("risks", [])
            )

        plan = super().create_plan(goal, tools, constraints)
        # Serialise and cache
        cache.write_text(json.dumps({
            "goal":             plan.goal,
            "reasoning":        plan.reasoning,
            "steps":            [vars(s) for s in plan.steps],
            "success_criteria": plan.success_criteria,
            "risks":            plan.risks,
        }, default=str))
        print("  💾 Plan saved to cache")
        return plan
```

---

### Multi-Model Plan-and-Execute

Use the best model for planning, cheapest model for execution:

```python
def build_cost_optimised_agent(api_key: str) -> PlanAndExecuteAgent:
    """
    Planner: claude-sonnet-4-6        — full reasoning power for planning
    Executor: claude-haiku-4-5        — cheap for routine step execution
    Synthesiser: claude-sonnet-4-6    — quality for final output

    Typical cost split:
      Planning:    ~$0.010  (one long call)
      Execution:   ~$0.002  (N short haiku calls)
      Synthesis:   ~$0.005  (one medium call)
      Total:       ~$0.017  vs ~$0.060 if all Sonnet
    """
    agent = PlanAndExecuteAgent.__new__(PlanAndExecuteAgent)
    agent.tool_registry = ToolRegistry()
    agent.planner       = Planner(api_key, model="claude-sonnet-4-6")
    agent.executor      = Executor(api_key, agent.tool_registry,
                                   model="claude-haiku-4-5-20251001")
    agent.synthesiser   = Synthesiser(api_key, model="claude-sonnet-4-6")
    agent._register_default_tools()
    return agent
```

---

## Integration with Other Patterns

### Plan-and-Execute + Router

Route the planning itself — use a larger model for complex plans, smaller for simple ones:

```python
class RouterAwarePlanner(Planner):
    def create_plan(self, goal: str, tools, constraints=None) -> Plan:
        # Estimate complexity of the goal
        complexity = len(goal.split()) + goal.count("?") * 5

        # Route to appropriate planning model
        if complexity > 30:
            self.model = "claude-sonnet-4-6"    # complex — full power
        else:
            self.model = "claude-haiku-4-5-20251001"  # simple — cheap enough

        return super().create_plan(goal, tools, constraints)
```

---

### Plan-and-Execute + RAG

Ground the planner's knowledge in retrieved documents:

```python
class RAGGroundedPlanner(Planner):
    """Planner that retrieves relevant context before generating the plan."""

    def __init__(self, api_key: str, rag_pipeline):
        super().__init__(api_key)
        self.rag = rag_pipeline

    def create_plan(self, goal: str, tools, constraints=None) -> Plan:
        # Retrieve relevant procedures, runbooks, or documentation
        context = self.rag.query(f"procedure for: {goal}")

        # Inject into planning prompt as grounding context
        enriched_goal = (
            f"{goal}\n\n"
            f"Relevant procedures from knowledge base:\n{context['answer']}\n"
            f"Sources: {context['sources']}"
        )
        return super().create_plan(enriched_goal, tools, constraints)
```

---

### Plan-and-Execute + Circuit Breaker

Protect expensive execution steps with circuit breakers:

```python
class ResilientExecutor(Executor):
    """Wraps each tool call with a circuit breaker."""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Simple failure counters per tool
        self._failures:  dict[str, int]   = {}
        self._open:      dict[str, bool]  = {}
        self._opened_at: dict[str, float] = {}

    async def execute_step(self, step: PlanStep, state: ExecutionState) -> Any:
        tool = step.action

        # Check circuit
        if self._open.get(tool):
            if time.time() - self._open_at.get(tool, 0) < 60:
                if step.optional:
                    step.status = StepStatus.SKIPPED
                    return f"[Skipped — {tool} circuit open]"
                raise RuntimeError(f"Circuit OPEN for tool '{tool}'")
            self._open[tool] = False   # half-open

        try:
            result = await super().execute_step(step, state)
            self._failures[tool] = 0   # reset on success
            return result
        except Exception:
            self._failures[tool] = self._failures.get(tool, 0) + 1
            if self._failures[tool] >= 3:
                self._open[tool]      = True
                self._opened_at[tool] = time.time()
                print(f"  🔴 Circuit OPENED for tool '{tool}'")
            raise
```

---

## Monitoring & Observability

### Execution Metrics

```python
def execution_report(result: dict, plan: Plan) -> str:
    """Generate a structured execution report."""
    step_times = {
        s.id: round(s.finished_at - s.started_at, 2)
        for s in plan.steps
        if s.started_at and s.finished_at
    }
    bottleneck = max(step_times, key=step_times.get) if step_times else None

    lines = [
        f"Goal:             {result['goal'][:80]}",
        f"Status:           {'✅ Complete' if result['steps_failed'] == 0 else '⚠️ Partial'}",
        f"Steps:            {result['steps_done']} done / "
                           f"{result['steps_failed']} failed / "
                           f"{result['steps_skipped']} skipped",
        f"Replans:          {result['replans']}",
        f"Total time:       {result['elapsed_seconds']}s",
        f"Bottleneck step:  Step {bottleneck} ({step_times.get(bottleneck, 0)}s)" if bottleneck else "",
        f"Plan version:     v{result['plan_version']}",
        f"Risks identified: {len(result.get('plan_risks', []))}",
    ]
    return "\n".join(l for l in lines if l)
```

### Key Metrics to Track

| Metric | Why It Matters | Alert Threshold |
|---|---|---|
| **Plan generation time** | Slow planning delays useful work | > 15s |
| **Plan accuracy** | % of plans executed without replan | < 80% (means planner is weak) |
| **Replan rate** | How often execution diverges from plan | > 20% of tasks |
| **Step failure rate** | Per-tool health | > 10% per tool |
| **Plan-to-execute cost ratio** | Planning shouldn't dominate total cost | Plan > 60% of total cost |
| **Irreversible action rate** | Proportion of steps that can't be undone | Track for safety auditing |

---

## Production Best Practices

### 1. Make Plans Inspectable Before Execution

```python
def print_plan(plan: Plan):
    """Always print the plan before running it — enables human review and logging."""
    print(f"\nPLAN v{plan.version}: {plan.goal}")
    print(f"Reasoning: {plan.reasoning[:200]}...")
    if plan.risks:
        print("Risks:")
        for r in plan.risks:
            print(f"  ⚠️  {r}")
    print(f"Steps ({len(plan.steps)}):")
    for s in plan.steps:
        deps = f" → needs [{s.depends_on}]" if s.depends_on else ""
        opt  = " (optional)" if s.optional else ""
        print(f"  {s.id}. [{s.action}] {s.description}{deps}{opt}")
```

---

### 2. Always Separate Read Steps from Write Steps

```
BAD plan:
  Step 1: read_and_modify_file("config.json")   ← read AND write in one step
  ← Can't roll back if modification is wrong

GOOD plan:
  Step 1: read_file("config.json")              ← read only
  Step 2: llm_task({"prompt": "generate improved config based on step 1"})
  Step 3: write_file({"path": "config.json.new", ...})  ← write to temp
  Step 4: run_command("diff config.json config.json.new")  ← verify diff
  Step 5: run_command("mv config.json.new config.json")    ← apply if ok
```

---

### 3. Mark Irreversible Steps as Checkpoints

```python
IRREVERSIBLE_ACTIONS = {"write_file", "run_command", "send_email",
                         "delete_file", "api_post", "db_write"}

def validate_plan_safety(plan: Plan) -> list[str]:
    """Identify unsafe patterns before execution starts."""
    warnings = []
    for i, step in enumerate(plan.steps):
        if step.action in IRREVERSIBLE_ACTIONS:
            # Check that a verification step precedes it
            prior_actions = [plan.steps[j].action for j in range(i)]
            if "read_file" not in prior_actions and "web_search" not in prior_actions:
                warnings.append(
                    f"Step {step.id} ({step.action}) is irreversible "
                    f"but no prior read/verify step exists."
                )
    return warnings
```

---

### 4. Cap Plan Length and Replan Budget

```python
MAX_PLAN_STEPS  = 10    # plans longer than this are probably too ambitious
MAX_REPLANS     = 2     # more than 2 replans = fundamentally broken plan
MAX_PLAN_TOKENS = 3000  # planning prompt shouldn't become enormous
```

---

### 5. Version and Log Every Plan

```python
import hashlib, json
from datetime import datetime

def log_plan(plan: Plan, goal: str):
    """Log every plan for auditing, debugging, and training data."""
    log_entry = {
        "timestamp":        datetime.utcnow().isoformat(),
        "goal":             goal,
        "plan_version":     plan.version,
        "plan_hash":        hashlib.md5(json.dumps([vars(s) for s in plan.steps],
                                                   default=str).encode()).hexdigest(),
        "num_steps":        len(plan.steps),
        "irreversible":     [s.id for s in plan.steps
                             if s.action in IRREVERSIBLE_ACTIONS],
        "risks_identified": plan.risks,
    }
    print(f"[PLAN LOG] {json.dumps(log_entry)}")
    # Write to your logging system (Datadog, CloudWatch, etc.)
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| Planner acts instead of plans | Plan phase calls tools; no separation | Strictly separate — planner returns JSON only, no tool calls |
| Steps not atomic | One step does read + transform + write | Split into three separate steps |
| No dependency modelling | Step 3 runs before step 2's output is ready | Always set `depends_on` for data dependencies |
| Irreversible steps without validation | File overwritten before verifying content | Always add a read/diff step before any write step |
| No replan budget | Infinite replanning loop | Set `MAX_REPLANS = 2`; fail hard after that |
| Planner model too weak | Produces vague, unexecutable plans | Use the most capable available model for planning |
| Executor model too large | Expensive for simple step execution | Executor only needs to follow instructions — use haiku/mini |
| No human review gate | Irreversible actions taken without oversight | For production: always show plan and require approval before irreversible steps |
| Planning prompt too long | Planner output quality degrades | Keep planning prompt < 3000 tokens; break into sub-plans if needed |
| Treating plan as immutable | Reality diverges, but plan isn't updated | Implement adaptive replanning triggered by unexpected results |

---

## Interview Cheat Sheet

**Q: What is the Plan-and-Execute pattern?**
A: A two-phase AI agent architecture. In the Plan phase, a capable model surveys the entire task and produces a structured, step-by-step plan before taking any action. In the Execute phase, an executor (often a smaller, cheaper model) works through the plan step-by-step, calling tools and producing results. A Synthesiser then combines all results into the final answer.

**Q: How does it differ from ReAct?**
A: ReAct interleaves thinking and acting — the model reasons, takes one action, observes the result, then reasons again. It never has a complete picture of the full task at once. Plan-and-Execute separates planning from execution entirely — the planner thinks through the whole task upfront, then the executor just carries out the predetermined steps. ReAct is better for exploration; Plan-and-Execute is better for tasks with irreversible actions, auditability requirements, or steps that can be parallelised.

**Q: Why use different models for planning vs execution?**
A: Planning requires deep reasoning — surveying the full task, anticipating risks, modelling dependencies. This needs a powerful model. Execution is simpler — just follow the instruction for this one step and call a tool. A cheap, fast model handles this well. Result: 60–80% cost reduction vs using the large model for everything.

**Q: When do you replan?**
A: Four triggers — hard failure of a critical step, an unexpected result that invalidates subsequent steps, a scope change discovered mid-execution, or a resource becoming unavailable (rate limit, outage). Cap replanning at 2–3 attempts; if the plan keeps failing, the task itself likely needs to be restated.

**Q: How do you handle irreversible actions safely?**
A: Three safeguards. First, the planner should always place a read/validate step before any irreversible write step. Second, mark irreversible steps in the plan so they're visible for human review. Third, for production systems, show the plan and require explicit approval before executing any step with an irreversible action (file write, API call, email sent).

**Q: What is the plan-first advantage for multi-step tasks?**
A: The planner can see the full dependency graph before acting. It can order steps correctly (read before write), identify parallelism opportunities (steps without dependencies can run concurrently), and catch logical errors — e.g., "step 4 depends on data from step 3 but I scheduled them in reverse order." A ReAct agent discovers these problems mid-execution, often after irreversible actions have already been taken.

**Q: How do you evaluate plan quality?**
A: Three metrics. Plan accuracy — what percentage of plans execute without requiring a replan (target > 80%). Step failure rate per tool — high failure rate signals either bad tool implementation or the planner calling the tool incorrectly. Success criteria match — after synthesis, does the final answer actually satisfy the stated success criteria (use an LLM judge).

**Q: What's the relationship between Plan-and-Execute and Orchestrator–Subagent?**
A: They are complementary. Plan-and-Execute focuses on *temporal separation* — plan first, then act. Orchestrator–Subagent focuses on *specialisation* — delegate to agents with different capabilities. In practice they combine naturally: the Orchestrator creates a plan (Plan-and-Execute) and then each step is delegated to a specialised Subagent (Orchestrator–Subagent).

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team