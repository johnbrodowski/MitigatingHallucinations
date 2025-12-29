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

## 4. OpenAI Function Calling Example (C# and Python)

This section demonstrates intent-driven tool use using both **C#** (common in production agent backends) and **Python** (common for rapid prototyping and orchestration).

### 4.1 Tool Definition (Conceptual)

The intent pattern is provider-agnostic. The tool schema conceptually looks like this:

* A required `intent` field
* Strict enforcement in the executor, not the model

---

### 4.2 C# Example (OpenAI Tool Calling)

```csharp
var tools = new[]
{
    new
    {
        type = "function",
        name = "execute_command",
        description = "Execute a shell command",
        parameters = new
        {
            type = "object",
            properties = new
            {
                command = new
                {
                    type = "string",
                    description = "Shell command to execute"
                },
                intent = new
                {
                    type = "string",
                    @enum = new[] { "DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE" },
                    description = "Behavioral execution mode"
                }
            },
            required = new[] { "command", "intent" }
        }
    }
};
```

### 4.3 C# Execution Layer (Kill Switch / Circuit Breaker)

```csharp
public string HandleToolCall(string command, string intent)
{
    return intent switch
    {
        "DIRECT" => ExecuteDirect(command),
        "EXPLORATORY" => Simulate(command),
        "ASSISTIVE" => ExplainAndConfirm(command),
        "INNOVATIVE" => SuggestAlternatives(command),
        _ => "CANNOT_PROCEED: unknown intent"
    };
}

private string ExecuteDirect(string command)
{
    if (string.IsNullOrWhiteSpace(command))
        return "CANNOT_PROCEED: missing command";

    return RunShell(command);
}
```

In **DIRECT** mode, execution halts immediately on missing information. No retries. No inference. This is the circuit breaker.

---

### 4.4 Python Example (Reference Implementation)

```python
def handle_tool_call(command: str, intent: str):
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

    return "CANNOT_PROCEED: unknown intent"
```

---

## 5. Anthropic Claude Example (C#)

Claude supports structured tool schemas similar to OpenAI. The same intent enforcement applies.

```csharp
var tools = new[]
{
    new
    {
        name = "file_operations",
        description = "Read, write, or delete files",
        input_schema = new
        {
            type = "object",
            properties = new
            {
                operation = new { type = "string", @enum = new[] { "read", "write", "delete" } },
                path = new { type = "string" },
                intent = new
                {
                    type = "string",
                    @enum = new[] { "DIRECT", "EXPLORATORY", "INNOVATIVE", "ASSISTIVE" }
                }
            },
            required = new[] { "operation", "path", "intent" }
        }
    }
};
```

DIRECT intent means: perform exactly one file operation and stop. If the path is missing or ambiguous, execution fails hard.

---

## 6. Gemini and Grok Compatibility (C# Perspective)

Because `intent` is application-level metadata, the same executor logic can be reused across providers.

### Unified Executor Interface (C#)

```csharp
public interface IIntentExecutor
{
    string Execute(string payload, string intent);
}
```

Each provider adapter only maps model output to this interface. The behavioral guarantees live entirely in your code, not the model.

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
