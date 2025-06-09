# Bridging Protocols: Translating Between General Tool Use and MCP

## Table of Contents
- [Bridging Protocols: Translating Between General Tool Use and MCP](#bridging-protocols-translating-between-general-tool-use-and-mcp)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [MCP-to-Framework Tool Use Proxy](#mcp-to-framework-tool-use-proxy)
    - [Purpose](#purpose)
    - [Flow](#flow)
    - [Key Field Mappings](#key-field-mappings)
    - [Example](#example)
  - [Framework Tool Use-to-MCP Proxy](#framework-tool-use-to-mcp-proxy)
  - [🔁 Multi-Turn Flow Comparison: General Tool Use vs MCP](#-multi-turn-flow-comparison-general-tool-use-vs-mcp)
    - [Example Goal: "What are my recent orders?"](#example-goal-what-are-my-recent-orders)
    - [🔧 Tool Use Only (Manual)](#-tool-use-only-manual)
    - [🤖 With MCP (Structured)](#-with-mcp-structured)
    - [🤝 A Behavioral Agreement](#-a-behavioral-agreement)
    - [Purpose](#purpose-1)
    - [Flow](#flow-1)
    - [Key Field Mappings](#key-field-mappings-1)
    - [Example](#example-1)
  - [🔄 Minimal Compatibility: Bridging without Full Fidelity](#-minimal-compatibility-bridging-without-full-fidelity)
  - [🧩 Semantic Kernel Integration Example](#-semantic-kernel-integration-example)
  - [🧭 Real-World Applications](#-real-world-applications)
    - [🌐 MindsDB as a Cross-Language MCP Gateway](#-mindsdb-as-a-cross-language-mcp-gateway)
    - [🛠 Proxy Use Across Vendor Environments](#-proxy-use-across-vendor-environments)
  - [Structural Similarities and Differences Between General Tool Use (e.g., OpenAI) and MCP](#structural-similarities-and-differences-between-general-tool-use-eg-openai-and-mcp)
    - [Shared Intent](#shared-intent)
    - [Common Structures](#common-structures)
    - [Differences in Approach](#differences-in-approach)
  - [Design Considerations](#design-considerations)
  - [🌐 Looking Ahead: Toward Protocol Convergence](#-looking-ahead-toward-protocol-convergence)
  - [Conclusion](#conclusion)
  - [📊 Language and Orchestrator Support Matrix](#-language-and-orchestrator-support-matrix)
  - [🧪 Proxy Implementation Notes](#-proxy-implementation-notes)
    - [🔄 Streaming Responses](#-streaming-responses)
    - [🧠 Memory Management](#-memory-management)
    - [🧱 Batching and Concurrency](#-batching-and-concurrency)
    - [🚨 Errors and Timeouts](#-errors-and-timeouts)
    - [🔐 Security and Input Validation](#-security-and-input-validation)
  - [🧵 Multi-Turn Patterns and Behavioral Alignment](#-multi-turn-patterns-and-behavioral-alignment)
    - [Tool Use: Stateless Request-Response](#tool-use-stateless-request-response)
    - [MCP: Ephemeral Memory and Event Chaining](#mcp-ephemeral-memory-and-event-chaining)
    - [Behavioral Alignment](#behavioral-alignment)
- [🧩 Feature Matrix: What MCP Adds Beyond General Tool Use](#-feature-matrix-what-mcp-adds-beyond-general-tool-use)

## Overview

As the AI ecosystem evolves, interoperability between different tool invocation standards becomes crucial. Two major paradigms have emerged:

- **OpenAI-style Tool Use** (commonly called Tool Calls), now broadly adopted across LLM providers.
- **Model Context Protocol (MCP)**, a clean, model-agnostic standard for tool invocation.

The **Model Context Protocol (MCP)** is an open standard for structuring LLM interaction with tools, memory, and conversation context. It uses typed events like `tool_use_requested` and `state_delta` to enable predictable multi-turn reasoning across sessions and systems.

Although differing slightly in JSON structure and execution philosophy, both systems share the same fundamental intent: 

> Enable an LLM to request external tool functionality during inference.

To harmonize these systems and unlock broader capabilities, we propose two complementary proxy designs:

1. **MCP-to-Framework Tool Use Proxy**
2. **Framework Tool Use-to-MCP Proxy**

These proxies translate requests and responses between standards, enabling seamless cross-compatibility.

> 🧭 Terminology Note  
> In this document, **Tool Use** refers to any structured mechanism by which an LLM interacts with external tools — such as those in Semantic Kernel, LangChain, or LangGraph.  
> When we use terms like `tool_calls`, this reflects OpenAI's schema specifically. Our focus is on the broader class of tool orchestration patterns, not vendor-specific APIs.

---

## MCP-to-Framework Tool Use Proxy

### Purpose

Allow existing MCP services to be accessible to clients (LLMs, runtimes, frameworks) that expect OpenAI-style tool use behavior (e.g., LangChain-style tool messages).

### Flow

1. Receive a standard MCP `tool_use_requested` event.
2. Transform it into a synthetic tool call structure used by LangChain, Semantic Kernel, or similar frameworks.
3. Simulate or dispatch appropriate function/tool execution.
4. Transform the result back into a synthetic tool response compatible with general tool orchestration schemas.

### Key Field Mappings

| MCP Field | Tool Use Field (general schema) | Notes |
|:---|:---|:---|
| `type: tool_use_requested` | `tool_calls` array (general schema) | Wrapped inside OpenAI schema |
| `tool` | `function.name` | |
| `args` | `function.arguments` (stringified JSON) | |
| `content` (from `tool_use_succeeded`) | `tool_response.output` (general schema) | Content parsed into output |

### Example

**MCP Request:**
```json
{
  "type": "tool_use_requested",
  "tool": "get_weather",
  "args": {
    "location": "Paris"
  }
}
```

**Equivalent Tool Use Representation (example schema format):**
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

## Framework Tool Use-to-MCP Proxy

## 🔁 Multi-Turn Flow Comparison: General Tool Use vs MCP

To better understand the difference MCP makes, let's look at a simplified multi-turn interaction in both styles — raw tool use (as in LangChain, Semantic Kernel, etc) versus MCP.

### Example Goal: "What are my recent orders?"

### 🔧 Tool Use Only (Manual)
1. **User**: "What are my recent orders?"
2. **LLM**: Calls `get_orders({ user_id: "123" })`
3. **Developer**: Executes the tool, gets JSON result
4. **Developer**: Injects back a system message:
   ```
   Tool result: { "orders": ["Order A", "Order B"] }
   ```
5. **LLM**: Attempts to continue reasoning based on that raw JSON

All memory, formatting, and flow control are up to the developer. The LLM doesn't have a structured understanding of the tool result unless you explicitly train or scaffold it.

### 🤖 With MCP (Structured)
1. **User message** is appended to a thread
2. **Model** emits:
   ```json
   {
     "type": "tool_use_requested",
     "tool": "get_orders",
     "args": { "user_id": "123" }
   }
   ```
3. **Developer** runs tool and responds with:
   ```json
   {
     "type": "tool_use_succeeded",
     "tool_call_id": "xyz",
     "content": {
       "prompt": "Fetched 2 orders for user 123: Order A and Order B."
     }
   }
   ```
4. **Model** continues the thread with full understanding of what the tool accomplished, framed in natural language it was trained to interpret

### 🤝 A Behavioral Agreement

MCP isn’t just a data structure — it’s a **conversation protocol**. The model has been trained to:

- Interpret `tool_use_requested` as a planning step
- Expect `tool_use_succeeded` or `tool_use_failed` afterward
- Read `prompt` fields as reliable summaries
- Use `state_delta` updates as persistent context

Because the model has been aligned with this structure during training, it behaves more predictably and coherently when you speak this protocol.

### Purpose

Allow general tool use requests (e.g., LangChain, Semantic Kernel, or similar tool call formats) to be transparently fulfilled by backends built using MCP.

### Flow

1. Receive a tool use request (such as a tool call array used in LangChain or Semantic Kernel).
2. Parse `function.name` and `function.arguments`.
3. Construct a corresponding MCP `tool_use_requested` event.
4. Forward it to the MCP service.
5. Translate the MCP `tool_use_succeeded` response into a tool response (e.g., OpenAI `tool_response`).

### Key Field Mappings

| Tool Use Field (general schema) | MCP Field | Notes |
|:---|:---|:---|
| `tool_calls[x].function.name` | `tool` | |
| `tool_calls[x].function.arguments` (parsed) | `args` | |
| MCP `content` (from `tool_use_succeeded`) | `tool_response.output` (general schema) | Content mapped back |

### Example

**Tool Use Input (example schema format):**
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
  "type": "tool_use_requested",
  "tool": "calculate_sum",
  "args": {
    "a": 5,
    "b": 7
  }
}
```

**MCP Response:**
```json
{
  "type": "tool_use_succeeded",
  "tool_call_id": "toolcall-456",
  "content": {
    "prompt": "12"
  }
}
```

**Converted Tool Response (example schema format):**
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

## 🔄 Minimal Compatibility: Bridging without Full Fidelity

Although MCP and general tool use patterns (including OpenAI-style `tool_calls`) differ in expressive power, they share a core structure that allows for lightweight interoperability:

| Shared Element        | Tool Use (e.g., LangChain, SK, vendor formats)      | MCP                          |
|-----------------------|-------------------------------|------------------------------|
| Tool name             | `function.name`               | `tool`                      |
| Arguments             | JSON string (inside chat)     | Native JSON object           |
| Tool result           | Injected `tool_response`      | Returned in `content` field  |

Because of this shared foundation:
- An **MCP server can fulfill a tool use request (general schema)** by translating the `tool_calls` array into a `tool_use_requested` event and formatting the response back.
- A **tool use handler** can respond to a simple `tool_use_requested` by pretending it's just another general tool call.

This enables **lightweight bridging** without full support for:
- Memory updates (`state_delta`)
- Retry semantics
- Prompt summarization
- Multi-turn planning metadata

> ✅ **TL;DR**: You don’t need full MCP fidelity to bridge tool use and MCP systems. Minimal fields can get them talking — and proxies can progressively layer on richer behaviors.



## 🧩 Semantic Kernel Integration Example

Recent developments in the Semantic Kernel (SK) ecosystem provide early guidance on integrating MCP-compatible tools within its plugin-based model. A blog post by Microsoft illustrates how to wrap MCP services inside SK-compatible function-call plugins, enabling developers to:

- Define plugins using the familiar `IChatFunction` interface.
- Delegate tool logic to a backend proxy that speaks MCP.
- Preserve structured tool semantics while embedding them in SK orchestration.

This pattern is a strong example of how frameworks originally designed for stateless function calls can adopt richer, protocol-aware interactions — without rewriting core logic.

🔗 See [Integrating Model Context Protocol Tools with Semantic Kernel](https://devblogs.microsoft.com/semantic-kernel/integrating-model-context-protocol-tools-with-semantic-kernel-a-step-by-step-guide/)

By wrapping MCP-compatible backends inside SK plugins, developers can prototype multi-turn, memory-aligned workflows within C# or .NET runtimes — extending the reach of protocol-level reasoning into general-purpose orchestration tools.

## 🧭 Real-World Applications

As MCP adoption increases, several platforms are building native support for its protocols — and some are serving as bridges for non-MCP environments.

### 🌐 MindsDB as a Cross-Language MCP Gateway

[MindsDB](https://mindsdb.com) has introduced MCP server capabilities that allow any LLM or orchestration system — including those not written in Python — to interact with tools and data sources using the MCP structure.

Through its support for SQL, REST, and vector-aware backends, MindsDB enables:

- **Framework Tool Use-to-MCP bridging** using SQL queries from C#, Java, Rust, or Go.
- Federated tool execution with native context summarization and prompt response formatting.
- Hosting of complex tool pipelines that emit MCP-standard events and states, even if the frontend model only supports tool use (e.g., OpenAI `tool_calls`).

This effectively turns MindsDB into a **vendor-neutral MCP router**, unlocking structured, multi-turn reasoning in any language environment.

### 🛠 Proxy Use Across Vendor Environments

These proxy mechanisms make it feasible to bridge tool invocation across LLM environments that adhere to different standards. For example:

- A hosted assistant emitting tool use requests (via standard schema) can interact with an MCP backend hosted on another platform.
- A Claude model using MCP structure can leverage a local tool implemented using OpenAI-compatible tool handlers.
- MindsDB or other MCP-compliant orchestrators can serve as an intermediary between stateless OpenAI tools and structured multi-turn MCP-compatible threads.

This layered interoperability promotes a “best tool for the job” mindset — where models and tools can be independently selected and composed regardless of original vendor alignment.

## Structural Similarities and Differences Between General Tool Use (e.g., OpenAI) and MCP

While OpenAI Tool Use and MCP appear different at first glance, their structures are strikingly similar:

### Shared Intent

Both systems structure inference into:

1. **Model generates an intent to call a tool**.
2. **External service processes the request**.
3. **Model resumes generation with external information**.

### Common Structures

| Aspect | Tool Use (e.g., LangChain, SK, LangGraph) | MCP |
|:---|:---|:---|
| **Trigger** | LLM emits `tool_calls` mid-generation (example schema) | Client sends `tool_use_requested` event |
| **Arguments** | JSON-stringified inside `function.arguments` | Native JSON object in `args` |
| **Response Handling** | `tool_response` fed back into conversation (example schema) | `content` returned via event |
| **Purpose** | Enhance LLM output during generation | Extend LLM reasoning with external services |

### Differences in Approach

| Dimension | Tool Use (e.g., LangChain, OpenAI, SK) | MCP |
|:---|:---|:---|
| **Integration Tightness** | Tightly woven into token generation | Looser, external service model |
| **Serialization** | Double-encoded JSON inside chat flow | Native JSON event payloads |
| **Error Handling** | Model sometimes infers error from content | Explicit `isError: true` field in content |
| **Flexibility** | Static or orchestrator-specified tools | Dynamic tool discovery and invocation |

Despite these differences, the overlap in purpose and operational structure is profound, making proxy adaptation not just possible but natural.

---

## Design Considerations

- **Error Handling:** MCP's `isError: true` must be mapped carefully into OpenAI tool failure semantics.
- **Timeouts and Streaming:** Need to accommodate slower MCP responses or future MCP streaming extensions.
- **Batching:** Some frameworks allow multiple tool calls per turn; proxies should be able to batch or sequence MCP calls.
- **Security:** Proxies must validate inputs carefully, particularly when passing arbitrary arguments.

---

## 🌐 Looking Ahead: Toward Protocol Convergence

As LLM vendors evolve toward more structured and event-driven orchestration models, the distinction between embedded tool calls and external protocols like MCP may blur.

These proxies serve a vital transitional purpose: helping models, tools, and orchestration engines speak a common language — even before that language becomes standardized.

In the future, native interoperability between LLMs, orchestrators, and tools could rely on shared schema families like MCP, enabling seamless multi-turn, tool-augmented reasoning out of the box.

---

## Conclusion

These proxies do not just convert JSON — they align model behavior patterns across otherwise incompatible inference protocols.

Proposing these two proxy mechanisms — MCP-to-ToolCall and ToolCall-to-MCP — would create a powerful interoperability layer across the AI landscape.

By translating between MCP and OpenAI Tool Call standards, we can:

- Unlock access to a broader universe of tools.
- Future-proof LLM applications.
- Foster an open, modular, and composable AI ecosystem.

## 📊 Language and Orchestrator Support Matrix

As developers choose platforms for LLM integration, the support for MCP and general tool use patterns (including OpenAI-style `tool_calls`) varies across ecosystems. This matrix outlines current maturity and community support for both mechanisms across key languages and orchestration frameworks.

| Language / Framework | Tool Use Support (General LLM Orchestration) | MCP Support | Notes |
|----------------------|----------------------------------------------|-------------|-------|
| Python               | ✅ Mature (OpenAI SDK, LangChain) | ✅ Early (Anthropic SDK, LangGraph) | Python leads in both paradigms. LangGraph is designed for MCP workflows. |
| JavaScript / TypeScript | ✅ Stable (LangChainJS, OpenAI SDK) | ⚠️ Limited (manual or via proxy) | MCP support is emerging but not first-class. |
| C# / .NET            | ⚠️ Partial (via REST, OpenAI SDKs) | ⚠️ Proxy-only via REST | No native SDKs yet. Proxies or services like MindsDB are vital. |
| Rust                 | ⚠️ Low (manual HTTP or FFI) | ⚠️ Experimental only | No direct MCP or Tool Use SDKs. Developers implement directly or via bindings. |

| Orchestrator / Runtime | Tool Use (General LLM Orchestration) | MCP | Highlights |
|------------------------|--------------------------------------|-----|------------|
| OpenAI Assistants API  | ✅ Native (OpenAI schema) | ❌ | Deeply integrated tool use model. |
| Anthropic Claude (via Messages API) | ❌ | ✅ Native MCP support | Claude is aligned with the MCP standard. |
| LangChain              | ✅ Yes    | ⚠️ Partial via LangGraph | Tool Use primary, but MCP patterns emerging. |
| Semantic Kernel        | ✅ Partial (Tool Call handlers) | ❌ | No MCP structure yet; proxies required. |
| MindsDB                | ✅ via SQL proxy | ✅ Native MCP | Bridges languages using database-first workflows. |

## 🧪 Proxy Implementation Notes

Bridging Tool Use and MCP protocols involves more than just field remapping. The following design concerns emerge in real-world usage:

### 🔄 Streaming Responses

MCP supports future extensions for streaming via event sequences. To support models that stream tool responses, proxies should:
- Accumulate partial `tool_use_succeeded` updates before synthesizing a final ToolCall response.
- Preserve order and completion markers across message boundaries.

### 🧠 Memory Management

MCP includes the ability to transmit `state_delta` to represent memory updates. Tool Use does not expose this natively, so proxies must:
- Capture memory-relevant updates from tool outcomes (when possible).
- Optionally inject them as synthetic system messages or update a backing memory store.

### 🧱 Batching and Concurrency

- OpenAI Tool Use supports multiple tool_calls in one turn.
- MCP prefers atomic, single tool_use_requested messages.

Proxies must:
- Batch tool requests from MCP → ToolCall if needed.
- Serialize ToolCall → MCP flows to avoid race conditions when stateful memory is involved.

### 🚨 Errors and Timeouts

- MCP explicitly declares `isError: true` and often a `status` or `reason` string.
- OpenAI Tool Use may require converting this into a failure output or skipping the tool response entirely.

Proxies must:
- Normalize error formats and timeouts into expected fallback patterns.
- Avoid confusing the LLM with undecoded internal error messages.

### 🔐 Security and Input Validation

- MCP requests can contain arbitrary structured arguments.
- Tool Use function signatures may be stricter or user-defined.

Proxies should:
- Sanitize and validate all incoming tool requests.
- Enforce schema correctness before dispatching execution.

---

## 🧵 Multi-Turn Patterns and Behavioral Alignment

A critical insight when comparing MCP to Tool Use is that models are not merely parsing structured data — they are exhibiting behavior aligned with training-time expectations.

### Tool Use: Stateless Request-Response
OpenAI's Tool Use is fundamentally transactional. Each tool_call represents a single-turn detour where:
- The model emits a function call mid-generation.
- The external function runs and returns a single response.
- The model resumes, with no built-in structure for memory, retries, or interpretation hints.

Any persistent state or multi-step reasoning must be managed by the host application.

### MCP: Ephemeral Memory and Event Chaining
MCP introduces **ephemeral memory** and **threaded event structure**:
- Events like `tool_use_requested`, `tool_use_succeeded`, and `state_delta` imply a temporal thread.
- Models trained on MCP sequences expect the host to respond with structured updates.
- This creates an illusion of "memory" — where models infer and maintain continuity through structured replies.

### Behavioral Alignment
The power of MCP comes not just from its schema, but from the model's **training alignment with that schema**.

In practice:
- LLMs respond more predictably to well-formed MCP sequences.
- They assume consistent phases (planning → tool use → result → continuation).
- Prompt fields are interpreted naturally as state summaries, not as new user messages.

Thus, multi-turn flows in MCP behave less like stateless function calls and more like **conversational state machines** — where each event nudges the model along a predictable path.

This behavioral expectation is what makes proxying non-trivial — and also what makes MCP worth aligning with, even from tool-use-native systems.

# 🧩 Feature Matrix: What MCP Adds Beyond General Tool Use

While general tool use (including OpenAI-style `tool_calls`) provides a minimal, stateless interface for calling external functions, MCP builds upon this with several structured features. This table outlines specific capabilities MCP introduces and whether they are natively supported in OpenAI-style Tool Use (e.g., LangChain, OpenAI, SK).

| Feature                        | Tool Use (e.g., LangChain, OpenAI, SK) | MCP   | Notes |
|-------------------------------|--------------------------|-------|-------|
| Retry Logic                   | ❌ Manual only           | ✅ Native | Retries are a formal phase in MCP, allowing LLMs to request re-attempts explicitly. |
| Prompt Summarization          | ❌                       | ✅     | Tool results are summarized into the `prompt` field to guide follow-up reasoning. |
| Stateful Memory (`state_delta`) | ❌                      | ✅     | Models can issue memory diffs; the orchestrator is expected to persist them. |
| Typed Tool Phases             | ❌                       | ✅     | MCP defines event types like `tool_use_requested`, `tool_use_failed`, and `tool_use_succeeded`. |
| Vector or File Awareness      | ❌ Manual                | ✅     | MCP threads may refer to file IDs or vector queries directly. |

These additions enable structured, multi-turn, memory-aware interactions that move beyond what stateless function calls allow — especially when models are aligned with these behaviors during training.

In short, MCP doesn’t just extend the JSON — it extends the **thinking model** that the LLM applies across sessions.

> 📖 See: [Model Context Protocol Overview – Anthropic](https://www.anthropic.com/news/model-context-protocol)