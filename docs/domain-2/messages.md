# JSON-RPC Messages & `_meta`

*Supports the competency **Schemas & Structured Data** and the exam focus on **message flow**.*

Every MCP interaction is a sequence of JSON-RPC messages. If you can read a raw message and say
what kind it is, who sent it and what it is doing, most message-flow questions become easy.

---

## 1. Why JSON-RPC

MCP needed a message format that is:

- **language-neutral** (servers are written in every language),
- **transport-independent** (the same messages over stdio and HTTP),
- **human-readable** (debugging a protocol by reading its traffic is invaluable),
- **simple** and **already widely implemented**.

**JSON-RPC 2.0** fits all of these. It is a tiny specification (a few pages) defining how to encode
a remote procedure call as JSON. The Language Server Protocol uses it for the same reasons.

JSON-RPC defines **requests**, **responses** and **notifications**. MCP adopts all three, and adds
rules on top.

All MCP messages **MUST** follow JSON-RPC 2.0 and **MUST** be UTF-8 encoded.

---

## 2. Requests

A request asks the receiver to perform an operation and expects exactly one response.

```typescript
{
  jsonrpc: "2.0";
  id: string | number;
  method: string;
  params?: { [key: string]: unknown };
}
```

A real one:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "location": "New York" },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": { "name": "ExampleClient", "version": "1.0.0" }
    }
  }
}
```

### MCP's rules for requests

1. The `id` **MUST** be a string or an integer.
2. Unlike base JSON-RPC, the `id` **MUST NOT be `null`**.
3. The `id` **MUST NOT** match any other request the sender has issued and **not yet received a
   response for**.
4. **In MCP, requests are sent only by clients.** Servers **MUST NOT** initiate JSON-RPC requests.
5. Every request **MUST** carry the required `_meta` fields (section 6).

Rule 3 is subtle. IDs need only be unique among *in-flight* requests. Once a request is answered its
ID could, in principle, be reused, though most implementations just increment a counter.

!!! note "Why null IDs are forbidden"
    In base JSON-RPC, a `null` id is legal but ambiguous: error responses to unparseable requests also
    use a missing/null id. Forbidding `null` on requests means an id-less error can only ever refer to a
    malformed message, never to a real request.

!!! tip "MRTR retries need a new id"
    A **Multi Round-Trip** retry is a brand-new request, so it **MUST** use a different `id` from the
    original, even though the method and arguments are the same.

---

## 3. Responses

A response answers a request. It carries the **same `id`** and exactly one of `result` or `error`.

### Result responses

```typescript
{
  jsonrpc: "2.0";
  id: string | number;
  result: {
    resultType: string;
    [key: string]: unknown;
  };
}
```

- **MUST** include the same `id` as the request.
- **MUST** include a `result` field, which may follow any JSON object structure.
- The `result` **MUST** include a **`resultType`** field. <span class="new-2026">2026</span>

### `resultType`: polymorphic results

In `2026-07-28` a request can end in more than one way, so the result declares its own shape:

| `resultType` | Meaning | Shape |
| --- | --- | --- |
| `"complete"` | Finished successfully | The method's normal result (`tools`, `content`, `contents`, …) |
| `"input_required"` | Needs more input first | `InputRequiredResult`: `inputRequests` and/or `requestState` |
| Extension-defined (e.g. `"task"`) | Defined by a negotiated extension | Per extension |

The rules:

- The set of valid values is the core set **plus** values from **extensions advertised via
  capabilities**.
- A `resultType` the client **does not recognize MUST be considered invalid**.
- An **absent** `resultType` (from a server implementing an earlier protocol version) **MUST be treated
  as `"complete"`**.

Think about why the last two rules differ. An **absent** value can only come from an older server that
predates the field, and its results were always complete, so treating them as complete is safe. An
**unrecognized** value means the server believes it is saying something specific that the client cannot
interpret. Guessing would be dangerous.

### Error responses

```typescript
{
  jsonrpc: "2.0";
  id?: string | number;
  error: {
    code: number;
    message: string;
    data?: unknown;
  };
}
```

- **MUST** include the same `id` as the request, **except** when the request was so malformed its id could
  not be read.
- **MUST** include `error` with an integer `code` and a `message`.
- **MAY** include `data` with additional detail of any type.

The codes, and when to use an error response versus a result with `isError: true`, are covered in
[Error Handling](../domain-3/error-handling.md).

---

## 4. Notifications

A notification is a one-way message. It has **no `id`**, and the receiver **MUST NOT** respond.

```typescript
{
  jsonrpc: "2.0";
  method: string;
  params?: { [key: string]: unknown };
}
```

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": { "progressToken": "abc123", "progress": 50, "total": 100 }
}
```

Notifications in the core protocol:

| Notification | Sender | Where it travels |
| --- | --- | --- |
| `notifications/progress` | Server | On the response stream of the request it relates to |
| `notifications/message` *(deprecated logging)* | Server | On the response stream of the request that set `logLevel` |
| `notifications/subscriptions/acknowledged` | Server | First message on a `subscriptions/listen` stream |
| `notifications/tools/list_changed` | Server | On a `subscriptions/listen` stream |
| `notifications/prompts/list_changed` | Server | On a `subscriptions/listen` stream |
| `notifications/resources/list_changed` | Server | On a `subscriptions/listen` stream |
| `notifications/resources/updated` | Server | On a `subscriptions/listen` stream |
| `notifications/cancelled` | Client (stdio); server only to end a listen stream | stdio channel |

!!! tip "How to classify a raw message in two seconds"
    - Has `method` **and** `id` → **request** (so the sender is a **client**)
    - Has `id` and `result` → **result response**
    - Has `id` (or none, if malformed) and `error` → **error response**
    - Has `method` and **no `id`** → **notification**

---

## 5. Direction: who sends what

The message patterns page states it flatly:

> Servers **MUST NOT** initiate JSON-RPC requests, and clients do not send JSON-RPC responses.

| Message type | Client sends | Server sends |
| --- | --- | --- |
| Request | ✅ | ❌ |
| Result / error response | ❌ | ✅ |
| Notification | ✅ (only `notifications/cancelled`, stdio) | ✅ |

This is one of the most important facts about `2026-07-28`, and it simplifies every transport: messages
only ever flow as client-request → server-response, plus server notifications.

---

## 6. `_meta`

`_meta` is a reserved property on `params` and `result` objects for **protocol-level metadata**. Arguments
describe *what to do*; `_meta` describes *the request itself*.

### Key naming rules

A `_meta` key has an optional **prefix** and a **name**: `prefix/name`.

**Prefix**

- If present: dot-separated labels followed by `/`.
- Labels start with a letter, end with a letter or digit, and may contain hyphens.
- Implementations **SHOULD** use **reverse DNS** (`com.example/`, not `example.com/`).
- Any prefix whose **second label** is `modelcontextprotocol` or `mcp` is **reserved for MCP**.

| Prefix | Reserved for MCP? | Why |
| --- | --- | --- |
| `io.modelcontextprotocol/` | Yes | second label `modelcontextprotocol` |
| `dev.mcp/` | Yes | second label `mcp` |
| `org.modelcontextprotocol.api/` | Yes | second label `modelcontextprotocol` |
| `com.mcp.tools/` | Yes | second label `mcp` |
| `com.example.mcp/` | **No** | second label is `example` |

**Name**

- Unless empty, begins and ends with an alphanumeric character.
- May contain hyphens, underscores, dots and alphanumerics in between.

Implementations **MUST NOT** make assumptions about values at reserved keys they don't understand.

### Reserved keys

| Key | Purpose |
| --- | --- |
| `progressToken` | Opts a request into progress notifications |
| `io.modelcontextprotocol/protocolVersion` | Protocol version for the request <span class="new-2026">2026</span> |
| `io.modelcontextprotocol/clientInfo` | Client name and version <span class="new-2026">2026</span> |
| `io.modelcontextprotocol/clientCapabilities` | Client capabilities for the request <span class="new-2026">2026</span> |
| `io.modelcontextprotocol/logLevel` | Minimum log level for the request (deprecated logging) <span class="new-2026">2026</span> |
| `io.modelcontextprotocol/subscriptionId` | Correlates a notification with its subscription <span class="new-2026">2026</span> |
| `traceparent`, `tracestate`, `baggage` | OpenTelemetry trace context (an explicit exception to the prefix rule) <span class="new-2026">2026</span> |

Official extensions add more keys under `io.modelcontextprotocol/`; third-party extensions use their own
vendor prefix.

### Per-request fields (client → server)

| Key | Type | Required |
| --- | --- | --- |
| `io.modelcontextprotocol/protocolVersion` | `string` | **Yes** |
| `io.modelcontextprotocol/clientCapabilities` | `ClientCapabilities` | **Yes** |
| `io.modelcontextprotocol/clientInfo` | `Implementation` | No (but clients **SHOULD** include it) |
| `io.modelcontextprotocol/logLevel` | `LoggingLevel` | No |

**Missing a required field** → the request is malformed → the server **MUST** reject it with **`-32602`
(Invalid params)**, and on HTTP with **`400 Bad Request`**.

**Needing an undeclared capability** → the server **MUST** return
**`MissingRequiredClientCapabilityError` (`-32021`)**, with `data.requiredCapabilities` listing what is
missing, and on HTTP **`400 Bad Request`**.

### Per-response fields (server → client)

| Key | Type | Required |
| --- | --- | --- |
| `io.modelcontextprotocol/serverInfo` | `Implementation` | No (but servers **SHOULD** include it on every result) |

On notifications delivered via a `subscriptions/listen` stream, the server **MUST** include
`io.modelcontextprotocol/subscriptionId`.

!!! danger "clientInfo and serverInfo are unverified"
    Both are **self-reported** and not verified by the protocol. They exist for **display, logging and
    debugging**. Implementations **SHOULD NOT** change behavior based on them and **SHOULD NOT** use them for
    **security decisions**. A client claiming `{"name": "TrustedCorpAgent"}` proves nothing. Identity for
    security comes from **authorization**.

### OpenTelemetry trace context

`traceparent`, `tracestate` and `baggage` are reserved for OpenTelemetry and must follow W3C Trace Context
and W3C Baggage formats. They let a trace flow from the host, through the MCP call, into the server and
beyond, so one distributed trace covers the whole operation. See
[Auditability & Observability](../domain-4/auditability.md).

```json
"_meta": { "traceparent": "00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01" }
```

---

## 7. Other general fields

### `icons`

Tools, resources, prompts and implementations may carry `icons`, an array of `{ src, mimeType?, sizes?,
theme? }`. `src` is an HTTPS URL or a `data:` URI.

The requirements are mostly **security** rules, because icons are fetched and rendered by the client from
server-supplied URLs:

- Treat icon metadata and bytes as **untrusted**.
- Accept only **`https:`** or **`data:`** URIs; **reject** `javascript:`, `file:`, `ftp:`, `ws:` and local
  app schemes; disallow scheme changes and cross-origin redirects.
- Fetch **without credentials** (no cookies, no `Authorization` header).
- Verify icons come from the **same origin** as the server.
- Beware **SVG with embedded JavaScript**; validate content type by magic bytes, not by the declared MIME type.
- Defend against oversized images and excessive GIF frames.
- Clients that render icons **MUST** support `image/png` and `image/jpeg`, and **SHOULD** support
  `image/svg+xml` and `image/webp`.

### `title` vs. `name`

Tools, resources and prompts have a `name` (programmatic identifier) and an optional `title`
(human-readable display name). Use `name` in protocol calls and `title` in UI.

---

## 8. A complete exchange, annotated

A client lists tools, then calls one, over stdio. Each line is one message.

```jsonc
// C→S  REQUEST (method + id)
{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{}}}}

// S→C  RESULT (id 1, resultType complete, cache hints)
{"jsonrpc":"2.0","id":1,"result":{"resultType":"complete","tools":[{"name":"get_weather","inputSchema":{"type":"object","properties":{"location":{"type":"string"}},"required":["location"]}}],"ttlMs":300000,"cacheScope":"public","_meta":{"io.modelcontextprotocol/serverInfo":{"name":"weather","version":"1.0.0"}}}}

// C→S  REQUEST (progressToken opts into progress)
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_weather","arguments":{"location":"Oslo"},"_meta":{"progressToken":"p-2","io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{}}}}

// S→C  NOTIFICATION (no id), scoped to request 2 via its token
{"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"p-2","progress":1,"total":2}}

// S→C  RESULT (id 2)
{"jsonrpc":"2.0","id":2,"result":{"resultType":"complete","content":[{"type":"text","text":"Oslo: 4°C, snow"}],"isError":false}}

// C→S  REQUEST with a missing required _meta field
{"jsonrpc":"2.0","id":3,"method":"tools/list","params":{}}

// S→C  ERROR (malformed: missing protocolVersion/clientCapabilities)
{"jsonrpc":"2.0","id":3,"error":{"code":-32602,"message":"Missing required _meta field: io.modelcontextprotocol/protocolVersion"}}
```

[Lab 1](../labs/01-raw-json-rpc.md) has you type messages like these into a real server by hand.

---

## 9. Recap

- MCP uses **JSON-RPC 2.0**, UTF-8 encoded: **requests**, **responses** (result or error), **notifications**.
- Request `id`s are string or integer, **never `null`**, unique among in-flight requests; an **MRTR retry
  uses a new id**.
- **Only clients send requests**; servers send responses and notifications.
- Every result has **`resultType`**: `complete`, `input_required` or an extension value. **Absent → complete;
  unrecognized → invalid.**
- Error responses carry integer `code`, `message`, optional `data`, and the request `id` unless unreadable.
- Notifications have **no id** and get **no response**.
- `_meta` keys: optional reverse-DNS prefix; prefixes whose **second label is `modelcontextprotocol` or `mcp`**
  are reserved.
- Required on every request: **`protocolVersion`** and **`clientCapabilities`**. Missing → **`-32602`** /
  **HTTP 400**. Undeclared capability needed → **`-32021`**.
- `clientInfo` and `serverInfo` are **self-reported**; never use them for security decisions.
- `traceparent` / `tracestate` / `baggage` carry OpenTelemetry context.
- Icons are **untrusted**: HTTPS or `data:` only, no credentials, same origin, beware SVG scripts.
