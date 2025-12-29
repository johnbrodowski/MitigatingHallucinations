# Mitigating Hallucinations: Intent‑Driven Tool Use

## Abstract

As large language models (LLMs) are increasingly embedded into automated systems, agent frameworks, and production tooling, a persistent failure mode has emerged: *unwanted helpfulness*. Models hallucinate API calls, invent parameters, retry failed actions creatively, or drift beyond the user’s original request. These behaviors are acceptable in conversational contexts but dangerous in deterministic or destructive environments.

This paper introduces **Intent‑Driven Tool Use**, a simple and universal design pattern for constraining LLM behavior by adding an explicit `intent` signal to every tool invocation and, optionally, to normal chat. The pattern requires no model fine‑tuning, no external classifiers, and no architectural changes to existing APIs. Instead, it formalizes behavioral boundaries at the interface layer, enabling deterministic execution, safe exploration, guided assistance, or creative ideation — *explicitly and on demand*.

---

## 1. The Problem: Helpful by Default

Modern LLMs are optimized for conversational usefulness. When a request is ambiguous or incomplete, they:

* Fill in missing parameters
* Infer user intent without confirmation
* Retry failed actions using alternatives
* Provide suggestions beyond the scope of the request

In automated systems, this manifests as:

* Hallucinated API calls
* Invented arguments or file paths
* Infinite “let me try another way” loops
* Silent deviation from the original instruction

While schema validation helps, it does not address *behavioral drift*. The model may still choose to explore, explain, or innovate when none of those are desired.

---

## 2. Core Idea: Make Intent Explicit

The solution is intentionally minimal: **add a single, required `intent` parameter to every tool call**.

The `intent` parameter does not describe *what* to do — it describes *how the model is allowed to behave while doing it*.

### 2.1 Intent Taxonomy

The following four intents cover the vast majority of operational needs:

* **DIRECT** — Execute exactly as requested. No commentary, no alternatives, no retries.
* **EXPLORATORY** — Gather information or simulate outcomes. Do not take action.
* **INNOVATIVE** — Propose alternative or creative solutions. Clearly mark speculation.
* **ASSISTIVE** — Explain what will happen, highlight risks, and ask for confirmation if needed.

This taxonomy is intentionally small. Its power comes from enforcement, not granularity.

---

## 3. Behavioral Rules by Intent

| Intent      | Allowed Behavior           | Forbidden Behavior             |
| ----------- | -------------------------- | ------------------------------ |
| DIRECT      | Execute exactly            | Guessing, retrying, explaining |
| EXPLORATORY | Inspect, simulate, analyze | Mutating state                 |
| INNOVATIVE  | Suggest alternatives       | Silent execution               |
| ASSISTIVE   | Explain and confirm        | Acting without consent         |

The model is no longer responsible for inferring the correct mode — it is declared explicitly.

---

## 4. OpenAI Function Calling Example

### 4.1 Tool Definition

```python
tools = [
    {
        "type": "function",
        "name": "execute_command",
        "description": "Execute a shell command",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "Shell command to execute"
                },
                "intent": {
                    "type": "string",
                    "enum": ["DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE"],
                    "description": "Behavioral execution mode"
                }
            },
            "required": ["command", "intent"]
        }
    }
]
```

### 4.2 System Prompt Enforcement

```text
When calling tools, you MUST include an intent.

DIRECT:
- Execute exactly as requested
- No suggestions or explanations
- If information is missing, respond: CANNOT_PROCEED: <reason>

EXPLORATORY:
- Investigate only
- Do not mutate state

INNOVATIVE:
- Propose alternatives
- Label speculation

ASSISTIVE:
- Explain effects
- Warn before destructive actions

Never override user intent.
```

### 4.3 Execution Layer (Kill Switch)

```python
def handle_tool_call(call):
    intent = call["arguments"]["intent"]
    command = call["arguments"]["command"]

    if intent == "DIRECT":
        if not command:
            return "CANNOT_PROCEED: missing command"
        return run(command)

    if intent == "EXPLORATORY":
        return simulate(command)

    if intent == "ASSISTIVE":
        return explain_and_confirm(command)

    if intent == "INNOVATIVE":
        return suggest_alternatives(command)
```

This logic functions as a **circuit breaker**: execution is impossible unless the declared intent permits it.

---

## 5. Anthropic Claude Example

```python
tools = [
    {
        "name": "file_operations",
        "description": "Read, write, or delete files",
        "input_schema": {
            "type": "object",
            "properties": {
                "operation": {"type": "string", "enum": ["read", "write", "delete"]},
                "path": {"type": "string"},
                "intent": {
                    "type": "string",
                    "enum": ["DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE"]
                }
            },
            "required": ["operation", "path", "intent"]
        }
    }
]
```

Claude is explicitly instructed to halt after DIRECT execution and never retry with alternatives unless intent allows it.

---

## 6. Gemini and Grok Compatibility

Because intent is application‑level metadata, the pattern is portable across providers. Any API that supports structured tool schemas can enforce intent deterministically in the executor layer.

The model suggests actions; your system decides whether they are allowed.

---

## 7. Intent in Normal Chat

Intent is not limited to tools.

### Example

```
[INTENT: ASSISTIVE]
Migrate the production database
```

The model is constrained to explanation and confirmation.

```
[INTENT: DIRECT]
List all running containers
```

The model must respond concisely and exactly.

This dramatically reduces conversational drift in operational chat interfaces.

---

## 8. Why This Works

* **Deterministic** — Behavior is constrained by declaration, not inference
* **Minimal** — One parameter, no classifiers, no fine‑tuning
* **Universal** — Works with any LLM that supports tool calls
* **Composable** — Integrates cleanly with RAG, agents, and automation

Most importantly, it acknowledges a core truth: *hallucinations are often a policy mismatch, not a knowledge failure*.

---

## 9. Conclusion

Intent‑Driven Tool Use reframes hallucination mitigation as a control‑surface problem rather than a modeling problem. By explicitly declaring behavioral intent at the boundary between language and action, we can preserve creativity where it is valuable — and enforce determinism where it is required.

The result is safer automation, cleaner agents, and models that finally stop “helping” when you didn’t ask them to.

---

## License

MIT — use it, fork it, ship it.
