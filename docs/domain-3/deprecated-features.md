# Deprecated: Sampling, Roots & Logging

*Competency: **Protocol Primitives**.*

`2026-07-28` deprecated three long-standing features. **Deprecated is not removed**: they remain in the
specification, still work, and still appear in exam questions. You need to know how they work, **why**
they were deprecated, and **what replaces them**.

---

## 1. What "deprecated" means precisely

Under the [feature lifecycle and deprecation policy](https://modelcontextprotocol.io/community/feature-lifecycle)
adopted in this revision (SEP-2596):

- A **Deprecated** feature **remains part of the specification** and remains **fully functional**.
- **New implementations SHOULD NOT adopt it.**
- **Existing implementations SHOULD migrate** before its earliest removal.
- It stays for **at least twelve months** before becoming **eligible** for removal (or **ninety days** under the
  expedited-removal exception). The window is measured **from the revision that first marks the feature
  Deprecated**, not from when the deprecation SEP reaches Final — which is why HTTP+SSE (deprecated
  `2025-03-26`, long past the floor) only gets a three-month transition grace period after SEP-2596 is Final.
- **Eligible** is a floor, not a date. Actual removal is a **Core Maintainer decision** made during release
  preparation, and may be later.
- Every deprecated feature is listed in the public
  [deprecated features registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated) with its SEP,
  migration path and earliest removal.

| Feature | Deprecated in | Migration path | Earliest removal |
| --- | --- | --- | --- |
| **Sampling** | `2026-07-28` (SEP-2577) | Integrate directly with **LLM provider APIs** | First revision on or after **2027-07-28** |
| **Roots** | `2026-07-28` (SEP-2577) | Pass directories/files via **tool parameters**, **resource URIs** or **server configuration** | First revision on or after **2027-07-28** |
| **Logging** | `2026-07-28` (SEP-2577) | Log to **`stderr`** (stdio); use **OpenTelemetry** | First revision on or after **2027-07-28** |
| `includeContext: "thisServer"` / `"allServers"` | `2025-11-25`, reclassified by SEP-2596 | Omit it or use `"none"` | No later than Sampling |
| **Dynamic Client Registration** | `2026-07-28` (PR #2858) | **Client ID Metadata Documents** | First revision on or after 2027-07-28 |
| **HTTP+SSE transport** | `2025-03-26`, reclassified by SEP-2596 | **Streamable HTTP** | Three months after SEP-2596 reaches Final |

!!! danger "Exam trap"
    - "Sampling **has been removed** from MCP" → **wrong**.
    - "Servers **can no longer** request sampling" → **wrong**. They can, via MRTR.
    - "New implementations **should not** adopt sampling" → **right**.

---

## Part 1: Sampling

### 2. What sampling does

> The Model Context Protocol provides a standardized way for servers to request LLM sampling ("completions" or
> "generations") from language models via clients. This flow allows clients to maintain control over model access,
> selection, and permissions while enabling servers to leverage AI capabilities — with no server API keys necessary.

The appeal was clear. A server wanting AI help (summarizing, classifying, picking the best of 47 flights) could borrow
the **client's** model instead of bringing its own API key, and the **client** kept control of which model, cost and
permissions.

### 3. How it works in `2026-07-28`

Like all server-to-client requests, sampling now goes through **MRTR**. The server returns an `InputRequiredResult`
containing a **`sampling/createMessage`** request; the client runs it through its LLM (with human review) and retries the
original request with the **`CreateMessageResult`** in `inputResponses`.

**Capability** (in each request's client capabilities):

```json
{ "sampling": {} }                    // basic
{ "sampling": { "tools": {} } }       // tool use inside sampling
{ "sampling": { "context": {} } }     // context inclusion (itself deprecated)
```

**Input request:**

```json
{
  "method": "sampling/createMessage",
  "params": {
    "messages": [ { "role": "user", "content": { "type": "text", "text": "What is the capital of France?" } } ],
    "modelPreferences": {
      "hints": [ { "name": "claude-3-sonnet" } ],
      "costPriority": 0.3,
      "intelligencePriority": 0.8,
      "speedPriority": 0.5
    },
    "temperature": 0.1,
    "systemPrompt": "You are a helpful assistant.",
    "maxTokens": 100
  }
}
```

**Client result (inside `inputResponses`):**

```json
{
  "role": "assistant",
  "content": { "type": "text", "text": "The capital of France is Paris." },
  "model": "claude-3-sonnet-20240307",
  "stopReason": "endTurn"
}
```

### 4. Model selection: preferences, not names

A server cannot simply demand a model by name. The client may not have it, or may prefer an equivalent from another
provider. So MCP uses **abstract priorities plus hints**:

**Priorities** (each `0`–`1`):

| Priority | Higher value prefers |
| --- | --- |
| `costPriority` | **Cheaper** models |
| `speedPriority` | **Faster** (lower-latency) models |
| `intelligencePriority` | **More capable** models |

**Hints**: an ordered list of `{ name }` treated as **substrings** that match model names flexibly
(`"claude-3-sonnet"`, then fallback `"claude"`). Clients **MAY map hints to equivalent models from other providers**. A
client with only Gemini might map a Sonnet hint to a comparable Gemini model. **Hints are advisory; the client makes the
final choice.**

### 5. Parameters and their obligations

| Parameter | Client obligation |
| --- | --- |
| `maxTokens` | **Required.** The client **MUST respect** it |
| `temperature`, `stopSequences`, `metadata` | Client **MAY modify or ignore** (the model may not support them) |
| `systemPrompt` | Client **MAY modify or ignore** without telling the server |
| `includeContext` | `"none"` (default), `"thisServer"`, `"allServers"`; client **MAY modify or ignore**. The last two are **deprecated** |

`includeContext` is instructive. `"allServers"` asked the client to include context from **every connected server** in a
completion requested by **one** server, which runs directly against the design principle that servers must not "see into"
other servers. Its deprecation predates sampling's.

### 6. Result fields

- `role` (`"assistant"`)
- `content`: a single block, or an array (e.g. multiple tool uses)
- `model`: the model actually used
- `stopReason`: `"endTurn"`, `"stopSequence"`, `"maxTokens"`, `"toolUse"`, or an implementation-defined value

### 7. Tools in sampling

Since `2025-11-25`, a server can let the client's LLM **use tools during sampling**, by sending `tools` and optional
`toolChoice`. The tool definitions are **scoped to that request**; they don't need to be tools the server exposes.

- Clients **MUST declare `sampling.tools`** to receive tool-enabled sampling requests; servers **MUST NOT** send them otherwise.
- `toolChoice.mode`: **`auto`** (model decides; default), **`required`** (must use at least one tool), **`none`** (must not
  use tools).

The **multi-turn tool loop**:

1. Server sends messages + tools; the LLM answers with **`tool_use`** blocks (`stopReason: "toolUse"`).
2. Server **executes** the tools itself.
3. Server sends a new sampling request with the history plus **`tool_result`** blocks.
4. Repeat until a final answer. Servers **might cap iterations**, and can pass `toolChoice: {mode: "none"}` on the last one to
   force an answer.

Two message constraints (for compatibility with provider APIs that use dedicated tool roles):

- A user message containing `tool_result` **MUST contain only tool results**, no mixed text.
- Every assistant message with `tool_use` **MUST be followed** by a user message of `tool_result`s, **one per `tool_use` id**
  (`toolUseId` matches `id`), before any other message.

Servers **MUST** honor that pairing; both sides **SHOULD** implement **iteration limits**.

### 8. Human in the loop

> For trust & safety and security, there **SHOULD** always be a human in the loop with the ability to deny sampling requests.

Applications **SHOULD** make requests easy to review, let users **view and edit prompts before sending**, and **present
generated responses for review before delivery** back to the server. That gives two checkpoints: one on the way to the LLM,
one on the way back.

Security considerations: approval controls, content validation on both sides, respecting model hints, **rate limiting**, and
handling sensitive data appropriately. If an error occurs or the user declines, the client **does not need to reply** (MRTR:
the server isn't waiting).

### 9. Why sampling was deprecated

SEP-2577 states the reasons:

- **Complex to implement.** Correct sampling needs human-in-the-loop approval, model selection logic, security
  considerations and, since SEP-1577, tool-loop support. That complexity contributed to low client adoption.
- **Low adoption.** Few clients support it, despite it being available since the November 2024 specification.
- **Direct alternatives.** Servers that need LLM capabilities can integrate directly with LLM provider APIs, gaining full
  control over model selection, parameters and streaming.

**Migration:** *integrate directly with LLM provider APIs.*

---

## Part 2: Roots

### 10. What roots do

> Roots inform servers about the directories and files the client considers relevant, so that servers can focus their
> operations accordingly. They are **informational guidance rather than an access-control mechanism**. The protocol does not
> enforce that servers stay within roots.

Typically a host exposes the folders the user has open. A filesystem or git server then knows where to work.

### 11. How it works in `2026-07-28`

**Capability:** `{"roots": {}}` in each request's client capabilities.

**Input request** (inside `inputRequests`):

```json
{ "method": "roots/list" }
```

**Client result** (inside `inputResponses`):

```json
{ "roots": [ { "uri": "file:///home/user/projects/myproject", "name": "My Project" } ] }
```

A root has:

- **`uri`**: **MUST be a `file://` URI** in the current spec
- **`name`**: optional display name

Changes: `notifications/roots/list_changed` was **removed**. Servers see updated roots **the next time they request them**.

### 12. Roots are not a security boundary

This is the examinable point. The client-concepts guide:

> Roots serve as a coordination mechanism between clients and servers, not a security boundary. The specification requires
> that servers "SHOULD respect root boundaries," and not that they "MUST enforce" them, because **servers run code the client
> cannot control**.

Real restriction must come from the **operating system**: file permissions, sandboxing, containers. Roots prevent **accidents**
by well-behaved servers; they don't stop **malicious** ones.

Security requirements still apply. Clients **MUST** only expose roots with appropriate permissions, **validate root URIs to
prevent path traversal**, implement access controls, and monitor accessibility. Servers **SHOULD** handle unavailable roots,
respect boundaries, and validate paths against them. Clients **SHOULD** prompt for consent before exposing roots.

### 13. Why roots were deprecated

From SEP-2577:

- **Low adoption.** Few clients implement roots, and few servers rely on them.
- **Vague semantics.** Roots are informational; servers aren't required to respect them, which reduces their usefulness.
- **Overlapping alternatives.** Working-directory context can be given through tool parameters, resource URIs, server
  configuration or environment variables, all of which are more explicit.

**Migration:** *pass directories or files via tool parameters, resource URIs, or server configuration.*

---

## Part 3: Logging

### 14. What logging does

Servers send **structured log messages** to clients, with **syslog severity levels** (RFC 5424):

| Level | Use |
| --- | --- |
| `debug` | Detailed debugging (function entry/exit) |
| `info` | General progress |
| `notice` | Normal but significant (config changes) |
| `warning` | Warning conditions (deprecated feature use) |
| `error` | Operation failures |
| `critical` | System component failures |
| `alert` | Immediate action needed (data corruption) |
| `emergency` | System unusable |

### 15. How it works in `2026-07-28`

**Capability** (server): `{"logging": {}}`.

**Per-request level.** `logging/setLevel` was **removed**. A client wanting logs for a request includes
**`io.modelcontextprotocol/logLevel`** in that request's `_meta`:

```json
{ "_meta": { "io.modelcontextprotocol/logLevel": "warning" } }
```

- The server **MUST NOT emit `notifications/message` for a request that did not include this field.**
- When present, the server **MAY** send messages **at or above** that level, **on that request's response stream**, before the
  final response.
- Log notifications are **request-scoped**: **MUST NOT** be delivered on a `subscriptions/listen` stream or any other stream.
- An **unrecognized level** → **`-32602`**.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/message",
  "params": { "level": "error", "logger": "database", "data": { "error": "Connection failed", "details": { "host": "localhost", "port": 5432 } } }
}
```

### 16. Security

Log messages **MUST NOT contain**:

- credentials or secrets,
- personal identifying information,
- internal system details that could aid attacks.

Servers **SHOULD** rate limit, include relevant context, use consistent logger names, and remove sensitive information.
Clients **MAY** display, filter or persist logs.

### 17. Why logging was deprecated

From SEP-2577:

- **Overlapping infrastructure.** Standard mechanisms (`stderr` for stdio, **OpenTelemetry** for structured observability)
  are mature, widely adopted, and better suited to logging than an application-protocol channel. MCP now reserves
  `traceparent`, `tracestate` and `baggage` in `_meta` for exactly this.
- **Low value relative to complexity.** Log message types, severity levels and `logging/setLevel` added implementation
  surface for every client and server.

**Migration:** *log to `stderr` for stdio transports; use OpenTelemetry for observability.* See
[Auditability & Observability](../domain-4/auditability.md).

---

## 4. The common thread

SEP-2577 frames all three deprecations with one principle: *"The MCP specification aims to remain minimal and focused.
Features that see low adoption, overlap with existing alternatives, or impose disproportionate implementation burden
relative to their value are candidates for removal."*

A useful way to see the pattern (this framing is the course's, not the SEP's): all three were ways for a **server to reach
back into the client**.

| Feature | Server reaches into the client's… | Replacement |
| --- | --- | --- |
| Sampling | **model** | The server's own provider integration |
| Roots | **workspace** | Explicit arguments, URIs, configuration |
| Logging | **log view** | stderr, OpenTelemetry |

The SEP also notes that during the deprecation period **wire-level behavior is unchanged**: no types are removed, capability
negotiation is unchanged, and no existing implementation breaks. The deprecation is a signal to stop building on these features.

**Elicitation is not deprecated.** Asking **the user** for input or consent has no substitute outside the host, which is the
only party with a UI.

---

## Recap

- **Deprecated ≠ removed.** Still in the spec and functional; new implementations **SHOULD NOT** adopt; ≥ **12 months** before
  removal eligibility; earliest removal for all three: first revision on or after **2027-07-28**.
- **Sampling:** `sampling/createMessage` via **MRTR**; `modelPreferences` (**cost/speed/intelligence priorities** + substring
  **hints**; client decides); **`maxTokens` required and must be respected**; tools via `sampling.tools` with
  `auto`/`required`/`none` and strict `tool_use` / `tool_result` pairing; **two human review checkpoints**. Migrate to **direct
  provider APIs**. `includeContext` `thisServer`/`allServers` deprecated.
- **Roots:** `roots/list` via MRTR; **`file://` only**; **advisory, not a security boundary**; `roots/list_changed` removed.
  Migrate to **tool parameters, resource URIs, server configuration**.
- **Logging:** syslog levels; **per-request `io.modelcontextprotocol/logLevel`** (`logging/setLevel` removed); **no messages
  without it**; **request-scoped** only; no secrets or PII. Migrate to **stderr** and **OpenTelemetry**.
- **Elicitation remains active.**
