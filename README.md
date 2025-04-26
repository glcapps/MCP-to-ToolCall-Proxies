# Bridging the Gap: Proposing MCP-to-ToolCall and ToolCall-to-MCP Proxies

## Overview

As the AI ecosystem evolves, interoperability between different tool invocation standards becomes crucial. Two major paradigms have emerged:

- **OpenAI-style Tool Use** (commonly called Tool Calls), now broadly adopted across LLM providers.
- **Model Context Protocol (MCP)**, a clean, model-agnostic standard for tool invocation.

Although differing slightly in JSON structure and execution philosophy, both systems share the same fundamental intent: 

> Enable an LLM to request external tool functionality during inference.

To harmonize these systems and unlock broader capabilities, we propose two complementary proxy designs:

1. **MCP-to-ToolCall Proxy**
2. **ToolCall-to-MCP Proxy**

These proxies translate requests and responses between standards, enabling seamless cross-compatibility.

---

## MCP-to-ToolCall Proxy

### Purpose

Allow existing MCP services to be accessible to clients (LLMs, runtimes, frameworks) that expect OpenAI-style Tool Call behavior.

### Flow

1. Receive a standard MCP `tools/call` request.
2. Transform it into a synthetic OpenAI `tool_calls` array.
3. Simulate or dispatch appropriate function/tool execution.
4. Transform the result back into a synthetic `tool_response`.

### Key JSON Mappings

| MCP Field | ToolCall Field | Notes |
|:---|:---|:---|
| `method: tools/call` | `tool_calls` array | Wrapped inside OpenAI schema |
| `params.name` | `function.name` | |
| `params.arguments` | `function.arguments` (stringified JSON) | |
| `content` | `tool_response.output` | Content parsed into output |

### Example

**MCP Request:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "Paris"
    }
  }
}
```

**Equivalent OpenAI ToolCall Representation:**
```json
{
  "tool_calls": [
    {
      "id": "toolcall-123",
      "type": "function",
      "function": {
        "name": "get_weather",
        "arguments": "{\"location\": \"Paris\"}"
      }
    }
  ]
}
```

---

## ToolCall-to-MCP Proxy

### Purpose

Allow OpenAI-style Tool Calls to be transparently fulfilled by backends built using MCP.

### Flow

1. Receive a Tool Call request (via OpenAI-compatible LLM output).
2. Parse `function.name` and `function.arguments`.
3. Construct a corresponding MCP `tools/call` request.
4. Forward it to the MCP service.
5. Translate MCP `content` response into ToolCall `tool_response`.

### Key JSON Mappings

| ToolCall Field | MCP Field | Notes |
|:---|:---|:---|
| `tool_calls[x].function.name` | `params.name` | |
| `tool_calls[x].function.arguments` (parsed) | `params.arguments` | |
| MCP `content` | `tool_response.output` | Content mapped back |

### Example

**ToolCall Input:**
```json
{
  "tool_calls": [
    {
      "id": "toolcall-456",
      "type": "function",
      "function": {
        "name": "calculate_sum",
        "arguments": "{\"a\": 5, \"b\": 7}"
      }
    }
  ]
}
```

**Generated MCP Request:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "calculate_sum",
    "arguments": {
      "a": 5,
      "b": 7
    }
  }
}
```

**MCP Response:**
```json
{
  "content": [
    {
      "type": "text",
      "text": "12"
    }
  ]
}
```

**Converted Tool Response:**
```json
{
  "tool_responses": [
    {
      "tool_call_id": "toolcall-456",
      "output": {
        "result": "12"
      }
    }
  ]
}
```

---

## Structural Similarities and Differences Between OpenAI Tool Use and MCP

While OpenAI Tool Use and MCP appear different at first glance, their structures are strikingly similar:

### Shared Intent

Both systems structure inference into:

1. **Model generates an intent to call a tool**.
2. **External service processes the request**.
3. **Model resumes generation with external information**.

### Common Structures

| Aspect | OpenAI Tool Use | MCP |
|:---|:---|:---|
| **Trigger** | LLM emits `tool_calls` mid-generation | Client sends `tools/call` request |
| **Arguments** | JSON-stringified inside `function.arguments` | Native JSON object in `params.arguments` |
| **Response Handling** | `tool_response` fed back into conversation | `content` returned via RPC-style call |
| **Purpose** | Enhance LLM output during generation | Extend LLM reasoning with external services |

### Differences in Approach

| Dimension | OpenAI Tool Use | MCP |
|:---|:---|:---|
| **Integration Tightness** | Tightly woven into token generation | Looser, external service model |
| **Serialization** | Double-encoded JSON inside chat flow | Native JSON RPC payloads |
| **Error Handling** | Model sometimes infers error from content | Explicit `isError: true` field in content |
| **Flexibility** | Static or session-specified tools | Dynamic tool discovery and invocation |

Despite these differences, the overlap in purpose and operational structure is profound, making proxy adaptation not just possible but natural.

---

## Design Considerations

- **Error Handling:** MCP's `isError: true` must be mapped carefully into OpenAI tool failure semantics.
- **Timeouts and Streaming:** Need to accommodate slower MCP responses or future MCP streaming extensions.
- **Batching:** OpenAI allows multiple tool calls per turn; proxies should be able to batch or sequence MCP calls.
- **Security:** Proxies must validate inputs carefully, particularly when passing arbitrary arguments.

---

## Conclusion

Proposing these two proxy mechanisms — MCP-to-ToolCall and ToolCall-to-MCP — would create a powerful interoperability layer across the AI landscape.

By translating between MCP and OpenAI Tool Call standards, we can:

- Unlock access to a broader universe of tools.
- Future-proof LLM applications.
- Foster an open, modular, and composable AI ecosystem.
