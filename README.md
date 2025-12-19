# Mitigating Hallucinations: Intent-Driven Tool Use

A simple, universal pattern for reducing AI hallucinations by adding explicit intent signals to tool calls.

## The Problem

LLMs default to "helpful" behavior - they fill gaps, suggest alternatives, and reframe problems. This causes:
- Hallucinated API calls
- Invented parameters
- "Let me try another way" loops
- Responses that drift from the actual request

## The Solution

Add a single `intent` parameter to your tool definitions. That's it.

```
INTENT OPTIONS:
- DIRECT: Execute exactly as requested. No alternatives, no commentary.
- EXPLORATORY: Gather options. Don't commit to a solution.
- INNOVATIVE: Propose novel approaches. Label speculation clearly.
- ASSISTIVE: Explain what would happen. Clarify before acting.
```

## How It Works

**Without intent:**
```
User: "Delete all logs"
AI: *deletes logs, then suggests setting up log rotation, offers to configure cleanup scripts*
```

**With intent=DIRECT:**
```
User: "Delete all logs" [intent: DIRECT]
AI: *deletes logs, stops*
```

**With intent=ASSISTIVE:**
```
User: "Delete all logs" [intent: ASSISTIVE]
AI: "This will permanently remove all files in /var/log/*. Confirm?"
```

---

## Implementation Examples

### OpenAI Function Calling

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "execute_command",
            "description": "Execute a shell command",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {
                        "type": "string",
                        "description": "The command to execute"
                    },
                    "intent": {
                        "type": "string",
                        "enum": ["DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE"],
                        "description": "Execution mode: DIRECT=execute exactly, EXPLORATORY=analyze options, INNOVATIVE=suggest alternatives, ASSISTIVE=explain before executing"
                    }
                },
                "required": ["command", "intent"]
            }
        }
    }
]

# In your system prompt:
system_prompt = """
When calling tools, the 'intent' parameter defines your behavior:
- DIRECT: Call the tool exactly as requested. No additional suggestions.
- EXPLORATORY: Use the tool to gather information. Present options.
- INNOVATIVE: Propose creative solutions using this tool.
- ASSISTIVE: Explain what the tool will do. Ask for confirmation if destructive.

Do not override user intent. If information is missing and intent is DIRECT, respond with "CANNOT_PROCEED: missing required parameter X" instead of guessing.
"""
```

### Anthropic Claude

```python
tools = [
    {
        "name": "file_operations",
        "description": "Read, write, or delete files",
        "input_schema": {
            "type": "object",
            "properties": {
                "operation": {
                    "type": "string",
                    "enum": ["read", "write", "delete"],
                    "description": "File operation to perform"
                },
                "path": {
                    "type": "string",
                    "description": "File path"
                },
                "intent": {
                    "type": "string",
                    "enum": ["DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE"],
                    "description": "DIRECT: execute immediately | EXPLORATORY: show what would happen | INNOVATIVE: suggest alternatives | ASSISTIVE: guide user through decision"
                }
            },
            "required": ["operation", "path", "intent"]
        }
    }
]

# System prompt addition:
"""
Tool Intent Rules:
- DIRECT mode: Execute the tool call. Stop immediately after. No explanations unless the tool fails.
- EXPLORATORY mode: Use tools to gather data. Present findings without taking action.
- INNOVATIVE mode: You may propose alternative tool uses if they better solve the problem.
- ASSISTIVE mode: Explain tool effects before use. Highlight risks.

If you cannot fulfill a DIRECT intent request exactly, respond: "CANNOT_PROCEED: [reason]" and stop.
"""
```

### Google Gemini

```python
tools = [
    {
        "function_declarations": [
            {
                "name": "database_query",
                "description": "Execute a database query",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "SQL query to execute"
                        },
                        "intent": {
                            "type": "string",
                            "enum": ["DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE"],
                            "description": "Behavioral mode: DIRECT=execute only, EXPLORATORY=explain results, INNOVATIVE=suggest optimizations, ASSISTIVE=validate before running"
                        }
                    },
                    "required": ["query", "intent"]
                }
            }
        ]
    }
]

# System instruction:
"""
You have access to tools with an 'intent' parameter. This parameter is mandatory and controls your behavior:

DIRECT: Execute the tool exactly as specified. Output only the result. Do not add commentary, suggestions, or alternatives.

EXPLORATORY: Use the tool to investigate. Present what you found. Do not make decisions or take further action.

INNOVATIVE: You may suggest better approaches using this tool. Clearly mark speculative ideas.

ASSISTIVE: Explain the tool's effect in plain language. For destructive operations, warn the user.

Never guess parameters. Never retry with alternatives unless intent is INNOVATIVE. If a DIRECT operation cannot be completed, output "CANNOT_PROCEED: [specific reason]" and stop.
"""
```

### xAI Grok

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "api_call",
            "description": "Make an HTTP API request",
            "parameters": {
                "type": "object",
                "properties": {
                    "endpoint": {"type": "string"},
                    "method": {"type": "string", "enum": ["GET", "POST", "PUT", "DELETE"]},
                    "body": {"type": "object"},
                    "intent": {
                        "type": "string",
                        "enum": ["DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE"],
                        "description": "DIRECT: make the call | EXPLORATORY: describe what the call would do | INNOVATIVE: suggest alternative endpoints | ASSISTIVE: explain risks and confirm"
                    }
                },
                "required": ["endpoint", "method", "intent"]
            }
        }
    }
]

# System context:
"""
Tool calls include an 'intent' parameter that defines operational boundaries:

- DIRECT: Make the call. Return the response. Do not interpret, summarize, or editorialize.
- EXPLORATORY: Describe what this API call would return. Do not actually make the call unless explicitly confirmed.
- INNOVATIVE: Propose alternative API calls that might better achieve the goal.
- ASSISTIVE: Explain what will happen in user-friendly terms. Highlight side effects.

Hard rule: In DIRECT mode, if you lack required information, output "CANNOT_PROCEED: missing [parameter]" - do not infer, do not substitute, do not assume.
"""
```

---

## Usage Patterns

### Pattern 1: Strict Automation
```python
user_message = "Deploy to production"
intent = "DIRECT"  # No safety rails, just execute
```

### Pattern 2: Safe Exploration
```python
user_message = "What would happen if I deleted this table?"
intent = "EXPLORATORY"  # Investigate but don't act
```

### Pattern 3: Guided Learning
```python
user_message = "Help me optimize this query"
intent = "INNOVATIVE"  # AI can suggest creative solutions
```

### Pattern 4: User Hand-Holding
```python
user_message = "Migrate the database"
intent = "ASSISTIVE"  # Explain steps, ask for confirmation
```

---

## System Prompt Template

Add this to your base system prompt:

```
You are operating in an intent-driven execution environment.

Every tool call includes an 'intent' parameter that defines your behavior:

DIRECT:
- Execute exactly what is requested
- Output only the result
- No suggestions, alternatives, or commentary
- If information is missing: respond "CANNOT_PROCEED: [reason]" and stop
- No "let me try another way" - halt on failure

EXPLORATORY:
- Use tools to gather information
- Present options clearly
- Do not commit to a solution
- Label all assumptions

INNOVATIVE:
- Propose novel approaches
- Clearly distinguish facts from speculation
- May suggest alternative tools or methods

ASSISTIVE:
- Explain what will happen before acting
- Highlight risks and side effects
- Ask clarifying questions if needed
- Confirm destructive operations

CRITICAL RULES:
1. Never override the declared intent
2. Never guess missing parameters in DIRECT mode
3. Never add "helpful" commentary in DIRECT mode
4. Stop immediately when you've fulfilled the request
5. Treat missing information as a blocker, not a prompt to be creative
```

---

## Why This Works

**Simple**: One parameter. Four values. Clear rules.

**Universal**: Works with any LLM API that supports tool calling.

**Deterministic**: Intent defines behavior boundaries. No drift.

**Practical**: Solves the real problem - unwanted "helpfulness" that causes hallucinations.

---

## License

MIT - do whatever you want with this.