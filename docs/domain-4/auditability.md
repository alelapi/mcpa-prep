# Auditability & Observability

*Competency: **Auditability & Observability**.*

**Observability** answers *what is happening right now, and why is it slow or failing?* **Auditability** answers
*who did what, when, on whose authority, and can we prove it?* AI agents make both harder: actions are taken by a
model on a user's behalf, across several servers, in response to text nobody reviewed. This chapter covers what
MCP provides, what it deliberately leaves to standard tooling, and what implementors must record.

---

## 1. Why AI tool use needs an audit trail

A traditional application action has a clear chain: *user clicked button → handler ran → database row changed*. An
MCP action has a longer, less obvious one:

```text
user prompt → model reasoning (maybe influenced by a tool result from server B)
            → tool call proposed → host policy / user approval
            → client request → server A → upstream API call
            → result → model → next action
```

When something goes wrong (data deleted, an email sent to the wrong person, a secret leaked), you need to reconstruct
that chain:

- **What** was called, with **which arguments**?
- **Which server** and **which version** handled it?
- **Who** was the authenticated user, and **which client** acted for them?
- **Did a human approve** it, or was it auto-approved, and under which policy?
- **What came back**, and did it influence later calls?
- **What authority** (token scopes) did the call carry?

The specification is explicit that clients **SHOULD log tool usage for audit purposes**, and that servers and proxies must
produce records that make incidents investigable.

---

## 2. What the specification requires or recommends

Collected across the spec and security best practices:

| Party | Recommendation | Why |
| --- | --- | --- |
| **Client** | **Log tool usage** for audit purposes (SHOULD) | Accountability for model-initiated actions |
| **Host** | **Activity logs** of all tool executions with results (documented pattern) | User-visible review |
| **Both** | **Log cancellation reasons** (SHOULD); UIs indicate cancellation | Debugging interrupted work |
| **Server** | **Log scope elevation events**: scope requested, subset granted, **with correlation IDs** | Audit privilege changes |
| **Proxy** | **Log all stdio transport usage** (process spawning) | Detect escalation |
| **Client** | **Log suspicious authorization URLs** | Security monitoring |
| **Schema validators** (opt-in network `$ref`) | **Log dereferenced URIs** | SSRF forensics |
| **Client** (tool rejections) | **Log a warning** when excluding an invalid `x-mcp-header` tool: tool name and reason | Explain missing tools |
| **Enterprise** | Enterprise-Managed Authorization gives an **auditable authorization trail** via the IdP | Compliance |

And what logs **must not** contain. From the (deprecated) Logging feature's security rules, which apply to logging in general:

- **credentials or secrets**,
- **personally identifying information**,
- **internal system details that could aid attacks**.

---

## 3. Identity: the foundation of accountability

An audit record is only as good as its **identity** fields, and MCP offers identity values that look useful but aren't
trustworthy.

| Identity source | Trustworthy for audit? | Notes |
| --- | --- | --- |
| `io.modelcontextprotocol/clientInfo` | **No** (as proof) | **Self-reported**; log it for debugging, never as the actor of record |
| `io.modelcontextprotocol/serverInfo` | **No** (as proof) | Self-reported; record it, but trust comes from configuration and source |
| User-provided claims ("I am joe@…") | **No** | Servers **MUST NOT** rely on unverified identification |
| **Validated access token** (`sub`, `client_id`, scopes, audience) | **Yes** | The authoritative identity for remote servers |
| Enterprise IdP assertion / ID-JAG subject | **Yes** | Stable subject for account linking and audit |
| stdio process owner | Implicit | The local user; the host is the audit point |

For remote servers, the spec's guidance on user identity is explicit: **derive it from credentials acquired via MCP authorization**
when possible, such as the **`sub` claim**. Bind audit records, state handles and elicitation state to **that** identity.

### Token passthrough destroys the audit trail

The security best practices list **accountability** as a core reason passthrough is forbidden. If a server forwards the client's token:

- the MCP server **can't distinguish** which MCP client made a call, because it sees an opaque upstream token;
- the **upstream API's logs** show a request apparently from a different identity than the MCP server that actually sent it.

Both make incident investigation harder. Separate tokens per hop keep each system's log truthful about who called it.

---

## 4. Observability without the Logging feature

The MCP **Logging** utility (`notifications/message`) is **deprecated** as of `2026-07-28`. The migration path doubles as the
observability guidance:

- **stdio servers: log to `stderr`.** The server **MAY** write any UTF-8 logging to stderr; the client **MAY** capture, forward or ignore
  it, and **SHOULD NOT** assume stderr output means an error. Never write logs to **stdout**, which is reserved for MCP messages.
- **Everything else: OpenTelemetry.**

SEP-2577's reasoning: standard mechanisms (stderr, OpenTelemetry) are *"mature, widely adopted, and better suited to logging than an
application-protocol channel."*

If an existing implementation still uses Logging: the client sets `io.modelcontextprotocol/logLevel` **per request**; the server **MUST NOT**
emit `notifications/message` for requests without it; messages are request-scoped and never on `subscriptions/listen` streams.

---

## 5. Distributed tracing with OpenTelemetry

### The problem

One user action can span: the host's agent loop, a model API call, an MCP client, an HTTP request, an MCP server, and one or more upstream
services. Without correlation, each produces isolated logs and nobody can tell which upstream call belonged to which user prompt.

**Distributed tracing** solves this by propagating a **trace context** (a trace ID plus a parent span ID) across every hop, so all the spans
form one tree.

### How MCP carries trace context

The spec reserves three `_meta` keys, an explicit exception to the prefix rule, for compatibility with OpenTelemetry:

| Key | Format |
| --- | --- |
| `traceparent` | W3C Trace Context |
| `tracestate` | W3C Trace Context |
| `baggage` | W3C Baggage |

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "location": "New York" },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "traceparent": "00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01"
    }
  }
}
```

A `traceparent` value has four parts: version `00`, the 32-hex **trace ID** shared by every span in the operation, the 16-hex **parent span ID**
(the caller's span), and **flags** (`01` = sampled).

### Why not just use HTTP headers?

HTTP already has `traceparent` headers. But:

- **stdio has no headers**; `_meta` works on every transport.
- One MCP request may use **several** HTTP requests (for example retries after a broken stream), and one HTTP connection can carry several MCP
  messages. The **MCP-level** context is independent of the transport's.

### The OpenTelemetry semantic conventions for MCP

OpenTelemetry publishes semantic conventions for MCP, currently in **Development** status. Their key points:

- Instrumentations **SHOULD** inject context into **`params._meta`** when creating a request, and receivers **extract** it as the **remote parent**.
  Trace keys are written **unprefixed** (per SEP-414).
- **Span name** **SHOULD** be `{mcp.method.name} {target}`, e.g. `tools/call get_weather`.
- Client spans are kind **CLIENT**; server spans are kind **SERVER**. The MCP client span is the parent of the server span **regardless of
  transport**, with span links recording transport context.
- Selected attributes:

| Attribute | Meaning |
| --- | --- |
| `mcp.method.name` | Method, e.g. `tools/call` (**required**) |
| `gen_ai.tool.name` | Tool name, when the operation concerns a tool |
| `gen_ai.prompt.name` | Prompt name |
| `mcp.resource.uri` | Resource URI (opt-in in span names, to avoid high cardinality) |
| `jsonrpc.request.id` | Request id |
| `mcp.protocol.version` | Protocol version |
| `error.type` | JSON-RPC error code, or **`tool_error`** when a tool result has **`isError: true`** |
| `gen_ai.tool.call.arguments` / `.result` | **Opt-in**, because they may contain sensitive data |

- Metrics include `mcp.client.operation.duration` and `mcp.server.operation.duration` (histograms, seconds).

!!! note "Conventions lag the redesign"
    These conventions predate `2026-07-28` in places. For example they still describe an `mcp.session.id` attribute for protocol sessions, which no
    longer exist in the current protocol. Treat them as evolving guidance.

Note the `error.type = tool_error` convention. It makes **tool execution errors** visible in metrics even though they are successful JSON-RPC
responses, so a dashboard doesn't show a tool failing 40% of the time as 100% success.

---

## 6. What a good MCP audit record contains

Combining the above, a host and server can log for each tool call:

```json
{
  "timestamp": "2026-09-15T14:03:22.418Z",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "span_id": "00f067aa0ba902b7",
  "actor": { "sub": "user_8f21", "client_id": "https://app.example.com/oauth/client.json" },
  "server": { "name": "crm-server", "version": "3.1.0", "origin": "https://mcp.crm.example" },
  "method": "tools/call",
  "tool": "update_contact",
  "tool_definition_hash": "sha256:9c1e…",
  "arguments": { "contact_id": "c_2291", "fields": ["phone"] },
  "approval": { "mode": "user_confirmed", "at": "2026-09-15T14:03:20.002Z" },
  "scopes": ["contacts:write"],
  "result": { "resultType": "complete", "isError": false },
  "duration_ms": 412
}
```

Field by field:

- `trace_id` / `span_id`: correlates with model calls and upstream services.
- `actor`: **validated** identity from the token, not `clientInfo`.
- `server` / `tool_definition_hash`: shows exactly which server and which definition ran, which supports rug-pull detection.
- `arguments`: **redacted or minimized** as required. Record enough to investigate, never secrets or unnecessary PII.
- `approval`: proves **human in the loop**, or records the auto-approval policy used.
- `scopes`: the authority the call carried.
- `result` status: distinguishes protocol errors, tool execution errors and success.

### Balancing auditability and privacy

Full arguments and results are the most useful for investigation and the most dangerous to store. Practical controls:

- **Redact** known secret patterns and PII before writing.
- Make argument and result capture **opt-in** or policy-controlled (as the OpenTelemetry conventions do).
- **Restrict** log access, set **retention** limits, and protect logs as the sensitive data they are.

---

## 7. Operational observability

Beyond audit, operators need to see health:

| Signal | Why it matters |
| --- | --- |
| Operation duration by method and tool | Slow tools, timeouts |
| Error rate by `error.type`, **including `tool_error`** | Failing upstreams, bad schemas, model misuse |
| `-32022` / `-32021` / `-32020` counts | Version skew, capability mismatches, broken intermediaries |
| `input_required` rate and retry completion | Elicitation friction; abandoned flows |
| 401 / 403 rates, step-up events | Token expiry, scope design problems |
| Cancellations and broken streams | Timeouts too short, proxy buffering (`X-Accel-Buffering`) |
| `subscriptions/listen` reconnects | Idle connections being dropped (keep-alives) |
| Rate-limit hits per tool and principal | Runaway agent loops |

### Debugging tools

- **MCP Inspector**: the official interactive tool for testing servers, in web, CLI and TUI forms. It shows raw requests and responses, and handles
  OAuth and protocol eras.
- **stderr capture** in hosts: most "server won't start" problems are visible there.
- **Traces** spanning host → server → upstream, via `traceparent` in `_meta`.

---

## 8. Governance: turning logs into control

Auditability supports **governance** decisions the exam's "AI governance & security roles" care about:

- **Which servers are allowed?** Allowlists by source or registry namespace, enforced by hosts or enterprise policy.
- **Which tools need approval?** Risk tiers backed by evidence from usage logs.
- **Who has access?** Enterprise-Managed Authorization for central policy, central revocation and a single trail.
- **Did a change alter behavior?** Definition hashes and version pinning.
- **Are deprecated features still in use?** Inventory which servers depend on sampling, roots or logging before their removal window.

---

## 9. Recap

- Auditability = *who did what, on whose authority*. Observability = *what's happening and why*. AI tool use needs both, because actions are
  model-initiated across multiple systems.
- Clients **SHOULD log tool usage**; servers **log scope elevation with correlation IDs**; proxies **log stdio spawning**; log suspicious URLs and
  cancellation reasons.
- **Never log** credentials, secrets, PII or attack-aiding internals; redact, restrict, retain minimally.
- Audit identity = **validated token claims** (`sub`, `client_id`, scopes), **not** `clientInfo`/`serverInfo` or user claims.
- **Token passthrough breaks accountability** on both the MCP server and upstream sides.
- Logging is deprecated: use **stderr** (stdio, never stdout) and **OpenTelemetry**.
- Trace context rides in **`_meta`**: **`traceparent`**, **`tracestate`**, **`baggage`** (unprefixed, W3C formats), and works over stdio too.
- OTel MCP conventions (Development): span `{mcp.method.name} {target}`, `mcp.method.name`, `gen_ai.tool.name`, **`error.type = tool_error`** for
  `isError`, opt-in arguments and results.
- A good record: trace ids, validated actor, server and definition hash, minimized arguments, **approval evidence**, scopes, result status.
