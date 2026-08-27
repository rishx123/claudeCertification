Yes — focus only on **how the context/data moves from one agent to another**.

## Overall flow

```text
Research Agents
      │
      ▼
research_findings
      │
      ▼
Synthesis Prompt
      │
      ▼
Task Tool
      │
      ▼
Synthesis Subagent
      │
      ▼
intermediate_output
(claims + sources + conflicts)
      │
      ▼
Next Agent
      │
      ▼
Final output
```

---

## Step 1: Research findings are collected

You first have:

```python
research_findings = [
    {
        "finding": "...",
        "source_url": "...",
        "retrieved_at": "...",
        "confidence": "high"
    }
]
```

Think of this as the **output from previous research agents**.

```text
Research Agent 1 ──┐
                   ├──► research_findings
Research Agent 2 ──┘
```

The important thing is that the finding is **not passed alone**.

It carries its metadata:

```text
Finding
   +
Source URL
   +
Retrieval date
   +
Confidence
```

So the context remains traceable.

---

# Step 2: The findings are inserted into the synthesis prompt

This line is important:

```python
{json.dumps(research_findings, indent=2)}
```

It takes the entire `research_findings` object and converts it into text/JSON that can be placed inside the prompt.

So:

```text
research_findings
        │
        ▼
json.dumps()
        │
        ▼
synthesis_prompt
```

The synthesis agent receives something conceptually like:

```text
You are a synthesis agent.

FINDINGS TO SYNTHESIZE:

[
  {
    "finding": "...",
    "source_url": "...",
    "retrieved_at": "...",
    "confidence": "high"
  }
]

Requirements:
...
```

So **the context is explicitly embedded into the prompt**.

This is important because the new subagent does not magically know what the previous agents found.

---

# Step 3: The entire prompt is passed through the Task tool

```python
task_call = {
    "type": "tool_use",
    "name": "Task",
    "input": {
        "description": "Synthesize research findings into structured report",
        "prompt": synthesis_prompt
    }
}
```

The flow is:

```text
research_findings
       ↓
synthesis_prompt
       ↓
Task tool call
       ↓
prompt given to Synthesis Subagent
```

The `prompt` is the actual **context transfer mechanism** here.

```text
Coordinator
    │
    │ passes synthesis_prompt
    ▼
Task Tool
    │
    │ gives prompt + embedded findings
    ▼
Synthesis Subagent
```

---

# Step 4: The synthesis agent produces `intermediate_output`

The synthesis agent processes the findings and returns:

```python
intermediate_output = {
    "claims": [...],
    "sources": [...],
    "conflicts": []
}
```

Now notice something important.

Instead of doing this:

```text
Research findings
       ↓
Synthesis Agent
       ↓
"Here are the conclusions"
```

it produces **structured output with relationships preserved**:

```text
Claim
   │
   └── source_id
           │
           ▼
        Source
```

For example:

```python
{
    "text": "Subagents do not inherit coordinator conversation history.",
    "source_id": "src_001"
}
```

And:

```python
{
    "source_id": "src_001",
    "url": "...",
    "title": "...",
    "retrieved_at": "..."
}
```

So the relationship is:

```text
Claim c001
     │
     │ source_id = src_001
     ▼
Source src_001
     │
     ├── URL
     ├── Title
     └── Retrieved date
```

---

# Step 5: Pass the WHOLE object to the next agent

This comment is the key concept:

```python
# Never extract just "claims" and drop "sources"
```

❌ Bad:

```python
next_agent_prompt = intermediate_output["claims"]
```

Now the next agent receives:

```text
"Subagents do not inherit coordinator history."
```

But it doesn't know:

```text
Where did this claim come from?
When was it retrieved?
How confident are we?
```

So you lose **provenance/context**.

---

✅ Good:

```text
Next Agent receives:

{
    claims: [...],
    sources: [...],
    conflicts: [...]
}
```

Flow:

```text
Synthesis Agent
      │
      ▼
intermediate_output
      │
      │ ENTIRE OBJECT
      ▼
Next Agent
```

Therefore:

> **Context is not just the claim; context includes the claim's source, metadata, confidence, and conflicts.**

---

# Step 6: Conflicting information is also preserved

Suppose two agents find:

```text
Source A:
"Task calls in the same response run in parallel."

Source B:
"Task calls are always sequential."
```

Instead of silently choosing one, the system passes:

```python
conflicts = [
    {
        "claim_a": {...},
        "claim_b": {...},
        "resolution": "unresolved"
    }
]
```

So the next agent receives:

```text
Claims
Sources
Conflicts
```

This prevents a later agent from incorrectly assuming that every claim is universally accepted.

---

# Step 7: Resuming a session

The final part is slightly different.

```python
resume_context = """
Resuming from previous analysis session.

CHANGES SINCE LAST SESSION:
- auth/middleware.py has been modified
- requirements.txt updated
- TODO in payment_service.py remains
"""
```

This context is given to a **new or resumed agent session**.

The flow is:

```text
Previous Session
      │
      ▼
Something changed while agent was away
      │
      ▼
resume_context
      │
      ▼
New / Resumed Agent
```

The agent is told:

```text
What happened before
+
What changed
+
What is still unfinished
```

So instead of giving it the entire project again:

```text
Analyze EVERYTHING again ❌
```

you give it a **delta**:

```text
Previous context
+
Changes since last session
        ↓
Continue intelligently
```

---

## The complete context-passing flow

```text
┌──────────────────┐
│ Research Agents  │
└────────┬─────────┘
         │ findings + source metadata
         ▼
┌────────────────────────┐
│ research_findings      │
│ finding                │
│ source_url             │
│ retrieved_at           │
│ confidence             │
└────────┬───────────────┘
         │
         │ json.dumps()
         ▼
┌────────────────────────┐
│ synthesis_prompt       │
│ Contains FULL findings │
└────────┬───────────────┘
         │
         │ Task tool
         ▼
┌────────────────────────┐
│ Synthesis Subagent     │
└────────┬───────────────┘
         │
         ▼
┌─────────────────────────────┐
│ intermediate_output         │
│                             │
│ claims ────────┐            │
│ sources ◄──────┘            │
│ conflicts                   │
└────────┬────────────────────┘
         │
         │ PASS WHOLE OBJECT
         ▼
┌──────────────────┐
│ Next Agent       │
└──────────────────┘
```

### The main concept you should remember

**Agents do not automatically share all context.** If Agent 2 needs Agent 1's knowledge, the coordinator/application must **explicitly pass that information**, usually through the prompt or a structured tool result.

And the important best practice here is:

> **Pass not just the conclusion, but the conclusion together with its provenance — sources, timestamps, confidence, and conflicts.**
