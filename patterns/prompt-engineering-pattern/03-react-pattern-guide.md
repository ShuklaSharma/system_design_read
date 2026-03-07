# ReAct (Reason + Act) Pattern

> Prompt Engineering Series — Pattern 03 of 03 | AI Engineering | March 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why ReAct Works](#why-react-works)
3. [ReAct vs Alternatives](#react-vs-alternatives)
4. [Implementation Patterns](#implementation-patterns)
5. [AI-Specific Use Cases](#ai-specific-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Production Safeguards](#production-safeguards)
8. [Combining with Other Patterns](#combining-with-other-patterns)
9. [Production Best Practices](#production-best-practices)
10. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is ReAct?

**ReAct (Reason + Act)** is a prompting pattern that interleaves natural language reasoning with discrete tool actions in a loop. The model alternates between thinking about what to do next (*Thought*) and executing an action (*Action*), then observing the result (*Observation*) before thinking again.

**Origin:** Introduced by Yao et al. (Princeton / Google, 2022) — demonstrated that combining reasoning traces with action execution outperformed both pure CoT and pure action-only approaches on knowledge-intensive tasks.

```
ReAct Loop:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Thought:      What do I need to know or do next?
  Action:       tool_name(argument)
  Observation:  [tool result returned by system]
  Thought:      What does this tell me? What's next?
  Action:       another_tool(argument)
  Observation:  [result]
  ...
  Thought:      I have enough information to answer.
  Answer:       [final response grounded in observations]
```

### The Problem Without ReAct

```
Scenario: User asks "What is the current EUR/USD exchange rate?"

Pure CoT (no tools):
─────────────────────────────────────────
Thought: The EUR/USD rate is approximately 1.08...
Answer: The EUR/USD exchange rate is about 1.08.

Problems:
- Data is from training cutoff (months/years old)
- Model may hallucinate a plausible-sounding rate
- No way to know if the answer is current
- Could cause real financial decisions based on stale data
```

### The Solution With ReAct

```
With ReAct (tool access):
─────────────────────────────────────────
Thought: I need the current EUR/USD rate. This is live data — I must use a tool.
Action: get_exchange_rate(EUR, USD)
Observation: {"rate": 1.0832, "timestamp": "2026-03-07T09:14:22Z"}

Thought: I have the current rate from a live source.
Answer: The current EUR/USD exchange rate is 1.0832 (as of 09:14 UTC today).

Result:
- Grounded in live data
- No hallucination
- Timestamp-accurate
- Trustworthy for real decisions
```

---

## Why ReAct Works

### The Core Mechanism

ReAct solves two fundamental LLM limitations simultaneously:

1. **Static knowledge** — LLMs have a training cutoff. ReAct connects them to live tools.
2. **Hallucination** — When the model doesn't know something, it may invent an answer. ReAct forces grounding in real observations.

The interleaved reasoning is critical — it's not just "call tools". The Thought step before each action ensures the model:
- Selects the right tool for the situation
- Constructs the right argument for that tool
- Interprets the observation correctly
- Adapts its plan when observations are unexpected

### Why Interleaving Beats Sequential

```
Sequential (wrong approach):
  1. Plan all actions upfront
  2. Execute all actions
  3. Reason about results

Problem: Plans made before seeing observations
         cannot adapt to unexpected results.

ReAct (correct approach):
  1. Think → Act → Observe
  2. Think → Act → Observe (adapted)
  3. Think → Answer

Benefit: Each observation informs the next action.
         The agent adjusts dynamically.
```

---

## ReAct vs Alternatives

| Approach | Strengths | Weaknesses | Best For |
|---|---|---|---|
| **Pure CoT** | No tool overhead, fast | Hallucination, stale data | Closed-domain reasoning |
| **Pure Tool Calling** | Efficient execution | No reasoning between calls | Simple, single-step lookups |
| **ReAct** | Adaptive, grounded, auditable | Higher latency, more tokens | Multi-step tasks with live data |
| **Plan-and-Execute** | Efficient for known workflows | Brittle to unexpected results | Predictable, stable tasks |
| **ReAct + Reflection** | Self-correcting | Highest cost | High-stakes accuracy |

---

## Implementation Patterns

### Pattern 1: Basic ReAct Agent

```python
import re
import anthropic

client = anthropic.Anthropic()

# ── Tool implementations ──────────────────────────────────────────
def web_search(query: str) -> str:
    """Search the web. Returns top results summary."""
    # Plug in your search API (Serper, Tavily, etc.)
    return f"[Search results for '{query}': ...results here...]"

def calculate(expression: str) -> str:
    """Safely evaluate a mathematical expression."""
    try:
        # Use a safe evaluator in production — never raw eval()
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"Error: {e}"

def get_current_time() -> str:
    """Return the current UTC time."""
    from datetime import datetime, timezone
    return datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")

TOOLS = {
    "search":           web_search,
    "calculate":        calculate,
    "get_current_time": lambda _: get_current_time(),
}

REACT_SYSTEM_PROMPT = """
You are a helpful assistant with access to the following tools:

- search(query: str): Search the web for current information
- calculate(expression: str): Evaluate a mathematical expression
- get_current_time(): Get the current UTC time

Always follow this exact format:

Thought: [Your reasoning about what to do next]
Action: tool_name(argument)

Wait for the Observation before continuing.
Once you have enough information:

Thought: I now have everything needed to answer.
Answer: [Your final response to the user]

Rules:
- Always reason before acting
- Never guess — use tools when you need current data
- If a tool fails, explain why and try an alternative approach
"""

# ── Agent loop ────────────────────────────────────────────────────
def react_agent(user_query: str, max_steps: int = 10) -> dict:
    messages = [{"role": "user", "content": user_query}]
    trace = []   # Full reasoning trace for debugging

    for step in range(max_steps):

        # Get model response (stop before Observation)
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=REACT_SYSTEM_PROMPT,
            messages=messages,
            stop_sequences=["Observation:"],
        )
        text = response.content[0].text.strip()
        trace.append({"step": step, "model_output": text})

        # Check for final answer
        if "Answer:" in text:
            answer = text.split("Answer:")[-1].strip()
            return {"answer": answer, "trace": trace, "steps": step + 1}

        # Parse tool call
        action_match = re.search(r'Action:\s*(\w+)\(([^)]*)\)', text)
        if not action_match:
            # Model deviated from format
            break

        tool_name = action_match.group(1)
        tool_arg  = action_match.group(2).strip('"\'')

        # Execute tool
        if tool_name not in TOOLS:
            observation = f"Error: Unknown tool '{tool_name}'. Available: {list(TOOLS.keys())}"
        else:
            try:
                observation = TOOLS[tool_name](tool_arg)
            except Exception as e:
                observation = f"Tool error: {str(e)}"

        trace.append({"step": step, "tool": tool_name, "arg": tool_arg, "observation": observation})

        # Feed observation back into context
        full_turn = f"{text}\nObservation: {observation}"
        messages.append({"role": "assistant", "content": text})
        messages.append({"role": "user",      "content": f"Observation: {observation}"})

    return {"answer": "Agent could not reach a conclusion.", "trace": trace, "steps": max_steps}


# Usage
result = react_agent("What is 15% of the current EUR/USD rate?")
print(f"Answer: {result['answer']}")
print(f"Steps taken: {result['steps']}")
```

---

### Pattern 2: Typed Tool Schema

Define tools with strict schemas for reliability:

```python
from dataclasses import dataclass
from typing import Callable, Any

@dataclass
class Tool:
    name: str
    description: str
    argument_description: str
    fn: Callable
    timeout_seconds: int = 10
    max_output_tokens: int = 500

TOOL_REGISTRY = [
    Tool(
        name="search",
        description="Search the web for current information, news, and facts",
        argument_description="A concise search query string",
        fn=web_search,
        timeout_seconds=15,
        max_output_tokens=800,
    ),
    Tool(
        name="calculate",
        description="Evaluate mathematical expressions. Supports +,-,*,/,**,sqrt,etc.",
        argument_description="A valid mathematical expression e.g. '(25 * 1.08) / 12'",
        fn=calculate,
        timeout_seconds=5,
        max_output_tokens=100,
    ),
    Tool(
        name="get_current_time",
        description="Get the current date and time in UTC",
        argument_description="No argument needed — pass empty string",
        fn=lambda _: get_current_time(),
        timeout_seconds=2,
        max_output_tokens=50,
    ),
]

def build_system_prompt(tools: list) -> str:
    tool_docs = "\n".join([
        f"- {t.name}({t.argument_description}): {t.description}"
        for t in tools
    ])
    return f"""You are a helpful assistant with access to these tools:

{tool_docs}

Format:
  Thought: [reasoning]
  Action: tool_name(argument)
  [wait for Observation]
  ...
  Answer: [final response]
"""
```

---

### Pattern 3: Streaming ReAct (Real-Time UX)

Show the model's thinking to users in real time:

```python
import sys

def react_agent_streaming(user_query: str) -> None:
    """Stream ReAct reasoning to the user in real time."""
    messages = [{"role": "user", "content": user_query}]

    print(f"\n🤔 Agent thinking about: {user_query}\n")
    print("─" * 50)

    for step in range(10):
        # Stream the model response
        full_text = ""
        with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=512,
            system=REACT_SYSTEM_PROMPT,
            messages=messages,
            stop_sequences=["Observation:"],
        ) as stream:
            for text_chunk in stream.text_stream:
                print(text_chunk, end="", flush=True)
                full_text += text_chunk

        print()  # newline after stream ends

        # Check for final answer
        if "Answer:" in full_text:
            break

        # Execute tool and show observation
        action_match = re.search(r'Action:\s*(\w+)\(([^)]*)\)', full_text)
        if action_match:
            tool_name = action_match.group(1)
            tool_arg  = action_match.group(2).strip('"\'')

            print(f"\n🔧 Calling {tool_name}({tool_arg})...")
            observation = TOOLS.get(tool_name, lambda x: "Tool not found")(tool_arg)
            print(f"📋 Observation: {observation}\n")
            print("─" * 50)

            messages.append({"role": "assistant", "content": full_text})
            messages.append({"role": "user", "content": f"Observation: {observation}"})
```

---

## AI-Specific Use Cases

### Use Case 1: Research Agent

```python
RESEARCH_SYSTEM = """
You are a research assistant. For every research task:

1. Break the question into sub-questions
2. Search for each sub-question independently
3. Synthesise findings into a coherent answer
4. Cite which searches produced which findings

Tools: search(query), calculate(expr)

Format: Thought / Action / Observation loop, then a detailed Answer.
"""

def research_agent(topic: str) -> str:
    response = react_agent(topic)
    return response["answer"]

# Example
answer = research_agent(
    "Compare the market cap of Apple and Microsoft as of today, "
    "and calculate the ratio between them."
)
```

---

### Use Case 2: Data Analysis Agent

```python
DATA_TOOLS = {
    "query_db":     lambda sql: run_sql(sql),
    "calculate":    lambda expr: str(eval(expr)),
    "plot":         lambda params: create_chart(params),
    "get_schema":   lambda table: get_table_schema(table),
}

DATA_SYSTEM = """
You are a data analyst with database access.

Tools:
- get_schema(table_name): Get column names and types for a table
- query_db(sql): Execute a SQL SELECT query
- calculate(expression): Perform calculations on query results
- plot(json_params): Create a chart from data

Always check the schema before writing queries.
Always verify row counts before drawing conclusions.
"""

# Example usage
result = react_agent(
    "What was the month-over-month revenue growth rate for Q4 2025?",
    # inject DATA_SYSTEM and DATA_TOOLS
)
```

---

### Use Case 3: Code Execution Agent

```python
import subprocess
import tempfile
import os

def safe_execute_python(code: str) -> str:
    """Execute Python code in a sandboxed subprocess."""
    with tempfile.NamedTemporaryFile(suffix=".py", mode="w", delete=False) as f:
        f.write(code)
        tmp_path = f.name

    try:
        result = subprocess.run(
            ["python3", tmp_path],
            capture_output=True,
            text=True,
            timeout=10,  # 10 second hard limit
        )
        output = result.stdout if result.returncode == 0 else result.stderr
        return output[:1000]  # Cap output length
    except subprocess.TimeoutExpired:
        return "Error: Code execution timed out (10s limit)"
    finally:
        os.unlink(tmp_path)

CODE_TOOLS = {
    "write_code":    lambda spec: generate_code(spec),
    "run_code":      safe_execute_python,
    "read_error":    lambda err: f"Error analysis: {err}",
}

CODE_SYSTEM = """
You are a coding assistant. To solve programming tasks:
1. Write the code
2. Run it and observe the output
3. If there's an error, analyse it and fix the code
4. Repeat until the code works correctly

Tools:
- write_code(specification): Generate Python code
- run_code(python_code): Execute Python code and return output/errors
"""
```

---

## Advanced Patterns

### Pattern: ReAct with Reflection

After completing a task, add a reflection step to improve future runs:

```text
After producing your Answer, add a REFLECTION section:

REFLECTION:
- Did I use the most efficient sequence of tool calls?
- Were any tool calls unnecessary?
- What would I do differently next time?
- Confidence in answer accuracy: [0-100%]
```

---

### Pattern: Multi-Agent ReAct

Orchestrate multiple specialised ReAct agents:

```python
class MultiAgentOrchestrator:
    def __init__(self):
        self.agents = {
            "researcher":  ReActAgent(system=RESEARCH_SYSTEM,  tools=RESEARCH_TOOLS),
            "analyst":     ReActAgent(system=ANALYSIS_SYSTEM,  tools=DATA_TOOLS),
            "writer":      ReActAgent(system=WRITING_SYSTEM,   tools=DOC_TOOLS),
        }

    def run(self, task: str) -> str:
        """Route task to appropriate agent or chain multiple agents."""

        # Step 1: Researcher gathers information
        research = self.agents["researcher"].run(f"Research: {task}")

        # Step 2: Analyst processes the findings
        analysis = self.agents["analyst"].run(
            f"Analyse this research and extract key metrics:\n{research}"
        )

        # Step 3: Writer produces final output
        report = self.agents["writer"].run(
            f"Write a concise report based on:\nResearch: {research}\nAnalysis: {analysis}"
        )

        return report
```

---

### Pattern: Adaptive Tool Selection

Let the model decide which tools to register based on the task:

```python
def adaptive_react(task: str, all_tools: dict) -> str:
    """
    First ask the model which tools it needs, then run ReAct
    with only those tools — reduces prompt length and confusion.
    """

    tool_descriptions = "\n".join([
        f"- {name}: {tool.description}"
        for name, tool in all_tools.items()
    ])

    selection_prompt = f"""
Given this task: "{task}"

Available tools:
{tool_descriptions}

Which tools are needed? List only tool names, comma-separated.
"""

    selection = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=100,
        messages=[{"role": "user", "content": selection_prompt}]
    )

    selected_names = [
        name.strip()
        for name in selection.content[0].text.split(",")
        if name.strip() in all_tools
    ]

    selected_tools = {name: all_tools[name] for name in selected_names}

    # Run ReAct with only the selected tools
    return react_agent(task, tools=selected_tools)
```

---

## Production Safeguards

These are non-negotiable for any ReAct agent in production:

### 1. Max Iterations

```python
MAX_STEPS = 10  # Never remove this limit

for step in range(MAX_STEPS):
    # ... agent loop
    pass

# If we reach here, agent failed to converge
return {"error": "Agent exceeded max steps", "partial_trace": trace}
```

---

### 2. Tool Allow-Listing

```python
# ❌ NEVER do this
TOOLS = {name: fn for name, fn in globals().items() if callable(fn)}

# ✅ Always explicitly register allowed tools
TOOLS = {
    "search":    safe_web_search,
    "calculate": safe_calculator,
    # Only what the agent NEEDS
}

def execute_tool(tool_name: str, arg: str) -> str:
    if tool_name not in TOOLS:
        return f"Error: Tool '{tool_name}' is not available."
    return TOOLS[tool_name](arg)
```

---

### 3. Observation Truncation

```python
MAX_OBSERVATION_TOKENS = 2000  # ~1500 words

def truncate_observation(raw_output: str) -> str:
    # Simple word-based truncation — use tokeniser for precision
    words = raw_output.split()
    if len(words) > MAX_OBSERVATION_TOKENS:
        truncated = " ".join(words[:MAX_OBSERVATION_TOKENS])
        return truncated + f"\n[... truncated — {len(words) - MAX_OBSERVATION_TOKENS} words omitted]"
    return raw_output
```

---

### 4. Per-Tool Timeouts

```python
import signal

class TimeoutError(Exception):
    pass

def with_timeout(fn, arg, seconds: int):
    def handler(signum, frame):
        raise TimeoutError(f"Tool timed out after {seconds}s")

    signal.signal(signal.SIGALRM, handler)
    signal.alarm(seconds)
    try:
        result = fn(arg)
        signal.alarm(0)
        return result
    except TimeoutError as e:
        return str(e)

TOOL_TIMEOUTS = {
    "search":           15,
    "calculate":         5,
    "get_current_time":  2,
    "query_db":         30,
}

def execute_with_timeout(tool_name: str, arg: str) -> str:
    fn = TOOLS[tool_name]
    timeout = TOOL_TIMEOUTS.get(tool_name, 10)
    return with_timeout(fn, arg, timeout)
```

---

### 5. Loop Detection

```python
def react_agent_safe(user_query: str) -> dict:
    messages = [{"role": "user", "content": user_query}]
    seen_actions = set()

    for step in range(10):
        response = get_model_response(messages)
        text = response.content[0].text

        if "Answer:" in text:
            return {"answer": text.split("Answer:")[-1].strip()}

        action_match = re.search(r'Action:\s*(\w+)\(([^)]*)\)', text)
        if action_match:
            action_key = f"{action_match.group(1)}:{action_match.group(2)}"

            # Detect loop
            if action_key in seen_actions:
                return {
                    "error": f"Loop detected — repeated action: {action_key}",
                    "steps": step
                }
            seen_actions.add(action_key)

        # ... continue loop
```

---

### 6. Human-in-the-Loop for Destructive Actions

```python
DESTRUCTIVE_TOOLS = {"send_email", "write_to_db", "delete_record", "make_payment"}

def execute_tool_safe(tool_name: str, arg: str) -> str:
    if tool_name in DESTRUCTIVE_TOOLS:
        # Require explicit confirmation
        print(f"\n⚠️  Agent wants to execute: {tool_name}({arg})")
        confirm = input("Approve? (yes/no): ").strip().lower()
        if confirm != "yes":
            return f"Action {tool_name} was rejected by the user."

    return TOOLS[tool_name](arg)
```

---

### Safeguard Summary

| Safeguard | Implementation | Why |
|---|---|---|
| Max iterations | Hard `for` loop limit (10–20) | Prevents infinite loops |
| Tool allow-list | Explicit `TOOLS` dict | Principle of least privilege |
| Observation truncation | Cap at ~2,000 tokens | Prevents context overflow |
| Per-tool timeout | `signal.alarm` or `asyncio.wait_for` | Prevents hanging agents |
| Loop detection | Track seen `(tool, arg)` pairs | Stops repetitive failures |
| Human-in-the-loop | Confirm destructive actions | Prevents irreversible mistakes |

---

## Combining with Other Patterns

### ReAct + CoT

Add structured reasoning within each Thought step:

```text
Thought: Let me reason step by step about what to do.
  Step 1: The user wants today's EUR/USD rate — this is live data.
  Step 2: My training data won't have today's rate.
  Step 3: I must use the get_exchange_rate tool.
  Therefore: I will call get_exchange_rate(EUR/USD).
Action: get_exchange_rate(EUR/USD)
```

### ReAct + Few-Shot

Provide 1–2 complete worked ReAct traces as examples in the prompt:

```text
# This dramatically improves tool selection and format adherence

Example trace:
  User: What is 20% of the S&P 500's current value?
  Thought: I need the current S&P 500 value — live data, need a tool.
  Action: get_stock_index(SP500)
  Observation: {"index":"S&P500","value":5842.31}
  Thought: Now calculate 20%.
  Action: calculate(5842.31 * 0.20)
  Observation: 1168.462
  Answer: 20% of the S&P 500's current value of 5,842 is approximately 1,168.

Now solve this new task: [task]
```

---

## Production Best Practices

### Observability — Log Everything

```python
import json
from datetime import datetime

def react_agent_with_logging(query: str, session_id: str) -> dict:
    start_time = datetime.utcnow()
    result = react_agent(query)
    end_time = datetime.utcnow()

    log_entry = {
        "session_id":   session_id,
        "query":        query,
        "answer":       result["answer"],
        "steps":        result["steps"],
        "duration_ms":  (end_time - start_time).total_seconds() * 1000,
        "trace":        result["trace"],
        "timestamp":    start_time.isoformat(),
    }

    # Send to your observability platform (Datadog, Langfuse, etc.)
    log_to_platform(log_entry)
    return result
```

### Cost Estimation

```
ReAct cost breakdown (per query):
  System prompt:   ~300-500 tokens (fixed per call)
  User query:      ~20-100 tokens
  Per step:
    Thought:       ~50-150 tokens output
    Action:        ~20-50 tokens output
    Observation:   ~100-500 tokens input (tool result)

Typical 3-step ReAct query:
  Input:  ~1,200 tokens
  Output: ~500 tokens
  Cost (Claude Sonnet at $3/$15 per M):  ~$0.004 per query

At 100K queries/day: ~$400/day
Compare to simple prompt: ~$50/day
Overhead: ~8x — justify with accuracy improvement metrics
```

### Latency Management

```python
# For latency-sensitive applications, run tools in parallel when possible
import asyncio

async def react_parallel_tools(tool_calls: list) -> list:
    """Execute multiple independent tool calls concurrently."""
    tasks = [
        asyncio.create_task(async_tool(name, arg))
        for name, arg in tool_calls
    ]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

---

## Interview Cheat Sheet

**Q: What is ReAct?**
A: A prompting pattern that interleaves Thought (reasoning), Action (tool call), and Observation (tool result) in a loop. Introduced by Yao et al. (2022). Solves the hallucination and stale knowledge problems of pure CoT.

**Q: What problem does ReAct solve that CoT cannot?**
A: CoT reasons only from training data — it can't access live information and may hallucinate facts. ReAct grounds every factual claim in real tool observations, making it accurate on tasks requiring current or private data.

**Q: What are the essential production safeguards for a ReAct agent?**
A: Max iterations (hard loop limit), tool allow-listing (never expose unneeded tools), observation truncation (cap tool output to ~2,000 tokens), per-tool timeouts, loop detection (track seen action pairs), and human-in-the-loop for destructive actions.

**Q: How do you prevent a ReAct agent from looping infinitely?**
A: Two mechanisms: (1) a hard `max_steps` limit in the agent loop, and (2) tracking the set of seen `(tool, argument)` pairs — if the same pair repeats, abort the loop.

**Q: What's the difference between ReAct and a function-calling API?**
A: Function calling is a structured protocol where the model outputs a tool call in a structured format (JSON). ReAct is a prompting pattern where the model generates free-text Thought/Action/Observation. In practice, modern APIs (like Anthropic's tool use) implement the structured version of what ReAct described as a prompting pattern.

**Q: How do you combine ReAct with Few-Shot?**
A: Provide 1–2 complete Thought/Action/Observation/Answer traces as examples in the prompt before presenting the new task. This dramatically improves the agent's tool selection accuracy and format adherence.

**Q: How do you handle tool failures in ReAct?**
A: Return a descriptive error string as the Observation (e.g. `"Error: timeout after 10s"`). The model will read this, reason about it, and typically try an alternative approach or tool. Never raise an exception that breaks the loop.

**Q: How do you estimate the cost of a ReAct agent?**
A: Count tokens: fixed system prompt (~400 tokens) + query + (per step: ~200 output tokens + tool observation as input). A typical 3-step query costs ~8x more than a simple prompt. Justify this with accuracy improvement data.

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
*← [Few-Shot Prompting Guide](02-few-shot-pattern-guide.md)*
