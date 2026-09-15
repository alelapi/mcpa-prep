# Glossary

Terms and method names you should recognize on sight. Items marked
<span class="new-2026">2026</span> are new or changed in `2026-07-28`.

## Participants and architecture

**Host**
: The AI application the user interacts with (Claude Desktop, VS Code, an agent). Creates and
  manages clients, enforces consent and security policy, and coordinates the LLM.

**Client**
: A protocol component inside the host that talks to **exactly one** server.

**Server**
: A program that exposes tools, resources and prompts. Local (usually stdio) or remote (usually
  Streamable HTTP).

**Local server**
: A server running on the user's machine, typically launched by the client as a subprocess over stdio.

**Remote server**
: A server reached over the network, typically via Streamable HTTP, usually serving many clients.

**Data layer**
: The JSON-RPC protocol: discovery, primitives, notifications.

**Transport layer**
: How messages move: connection setup, framing, authorization. stdio or Streamable HTTP.

## Base protocol

**JSON-RPC 2.0**
: The message format. Requests, responses (result or error) and notifications.

**Request**
: A message with a non-null `id` that expects a response. In MCP only **clients** send requests.

**Notification**
: A message with **no `id`**. The receiver **MUST NOT** respond.

**`_meta`**
: Reserved metadata field on params and results. Holds protocol-level keys under the
  `io.modelcontextprotocol/` prefix.

**Stateless** <span class="new-2026">2026</span>
: Each request carries everything needed to process it. No state may be inferred from earlier
  requests on the same connection.

**`io.modelcontextprotocol/protocolVersion`** <span class="new-2026">2026</span>
: Required on every request. The protocol version the request uses.

**`io.modelcontextprotocol/clientCapabilities`** <span class="new-2026">2026</span>
: Required on every request. What the client supports for this request.

**`io.modelcontextprotocol/clientInfo`** / **`serverInfo`** <span class="new-2026">2026</span>
: Self-reported, unverified identity. For display, logging and debugging, **not** security decisions.

**`resultType`** <span class="new-2026">2026</span>
: Required on every result. `"complete"` or `"input_required"` (extensions may add more). Missing
  means `"complete"`; unrecognized means invalid.

**Capabilities**
: Declared feature support. Client capabilities go in each request's `_meta`; server capabilities
  are returned by `server/discover`.

**Extension**
: Optional, opt-in addition to the core protocol, negotiated via the `extensions` capability map.
  Identifiers look like `io.modelcontextprotocol/tasks`.

**Protocol version**
: A date string `YYYY-MM-DD`, the last date backward-incompatible changes were made.

**Modern / Legacy / Dual-era** <span class="new-2026">2026</span>
: Modern = `2026-07-28`+ (per-request metadata). Legacy = `2025-11-25` and earlier (`initialize`
  handshake). Dual-era = supports both.

## Methods

| Method | Direction | Purpose |
| --- | --- | --- |
| `server/discover` <span class="new-2026">2026</span> | C→S | Supported versions, capabilities, identity, instructions. Servers MUST implement |
| `tools/list` | C→S | List tools (paginated, cacheable) |
| `tools/call` | C→S | Invoke a tool (MRTR-capable) |
| `resources/list` | C→S | List direct resources |
| `resources/templates/list` | C→S | List resource templates |
| `resources/read` | C→S | Read resource contents (MRTR-capable, cacheable) |
| `prompts/list` | C→S | List prompts |
| `prompts/get` | C→S | Get a prompt with arguments filled in (MRTR-capable) |
| `completion/complete` | C→S | Argument autocompletion for prompts and resource templates |
| `subscriptions/listen` <span class="new-2026">2026</span> | C→S | Open a long-lived stream of opted-in change notifications |
| `elicitation/create` | inside `inputRequests` | Ask the user for input (form or URL mode) |
| `sampling/createMessage` | inside `inputRequests` | *Deprecated.* Ask the client's LLM for a completion |
| `roots/list` | inside `inputRequests` | *Deprecated.* Ask the client for filesystem roots |

| Notification | Direction | Purpose |
| --- | --- | --- |
| `notifications/subscriptions/acknowledged` <span class="new-2026">2026</span> | S→C | First message on a listen stream; which filters are honored |
| `notifications/tools/list_changed` | S→C | Tool list changed (on a listen stream) |
| `notifications/prompts/list_changed` | S→C | Prompt list changed |
| `notifications/resources/list_changed` | S→C | Resource list changed |
| `notifications/resources/updated` | S→C | A subscribed resource changed |
| `notifications/progress` | S→C | Progress on an in-flight request (request-scoped) |
| `notifications/message` | S→C | *Deprecated.* Log message (request-scoped, only if `logLevel` was set) |
| `notifications/cancelled` | C→S (stdio) | Cancel an in-flight request. Servers send it only to end a listen stream |

## Interaction patterns

**Request and response**
: Client sends a request, server returns a result or error, optionally preceded by request-scoped notifications.

**Multi Round-Trip Requests (MRTR)** <span class="new-2026">2026</span>
: Server returns `InputRequiredResult`; client gathers input and retries the original request.

**`InputRequiredResult`** <span class="new-2026">2026</span>
: Result with `resultType: "input_required"`, carrying `inputRequests` and/or `requestState`.

**`inputRequests`** / **`inputResponses`** <span class="new-2026">2026</span>
: Maps keyed by server-chosen strings. Requests (elicitation, sampling, roots) and the client's answers.

**`requestState`** <span class="new-2026">2026</span>
: Opaque server string the client must echo exactly. Attacker-controlled from the server's view.

**Subscribe and notify** <span class="new-2026">2026</span>
: `subscriptions/listen` opens a stream; notifications carry `io.modelcontextprotocol/subscriptionId`.

## Server features

**Tool**
: A function the model can invoke. **Model-controlled.**

**Resource**
: Data identified by a URI, provided as context. **Application-controlled.**

**Resource template**
: A parameterized resource URI (RFC 6570), e.g. `file:///{path}`.

**Prompt**
: A server-defined message template the user selects. **User-controlled.**

**`inputSchema`** / **`outputSchema`**
: JSON Schema (default 2020-12) for tool arguments and structured results.

**`structuredContent`**
: Machine-readable tool result (any JSON value), validated against `outputSchema` if present.

**`isError`**
: `true` on a tool result means a **tool execution error**, reported to the model so it can self-correct.

**Tool annotations**
: Hints like `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`. **Untrusted** unless the server is trusted.

**`x-mcp-header`** <span class="new-2026">2026</span>
: Schema annotation that mirrors a tool parameter into an `Mcp-Param-{Name}` HTTP header.

**Content annotations**
: `audience` (`user`/`assistant`), `priority` (0–1), `lastModified`.

**Resource link** vs. **embedded resource**
: A `resource_link` points to a URI; a `resource` content block includes the contents inline.

## Client features

**Elicitation**
: Server asks the user for input via the client. **Form mode** collects structured, non-sensitive data.
  **URL mode** sends the user to a URL for sensitive interactions that must not pass through the client.

**Elicitation actions**
: `accept` (submitted), `decline` (explicitly refused), `cancel` (dismissed without choosing).

**Sampling** *(deprecated)*
: Server asks the client's LLM for a completion.

**Roots** *(deprecated)*
: Client tells the server which `file://` directories are relevant. Advisory, not enforced.

## Utilities

**Caching** <span class="new-2026">2026</span>
: `ttlMs` (freshness hint in ms) and `cacheScope` (`public` or `private`) on cacheable results.

**Pagination**
: Opaque `cursor` / `nextCursor`. Server chooses page size. An empty string cursor is still a cursor.

**Progress**
: Client puts `progressToken` in `_meta`; server sends increasing `notifications/progress`.

**Cancellation**
: stdio: `notifications/cancelled`. Streamable HTTP: close the SSE response stream.

**Completion**
: `completion/complete` with `ref/prompt` or `ref/resource`; up to 100 values.

## Transports

**stdio**
: Client launches server as subprocess. Newline-delimited JSON-RPC on stdin/stdout. Logs to stderr.

**Streamable HTTP**
: One MCP endpoint. Every client message is a POST. Reply is JSON or a request-scoped SSE stream.

**`MCP-Protocol-Version`**, **`Mcp-Method`**, **`Mcp-Name`** <span class="new-2026">2026</span>
: Required HTTP headers mirroring body fields so intermediaries can route without parsing JSON.

**HTTP+SSE** *(deprecated)*
: The `2024-11-05` two-endpoint transport, replaced by Streamable HTTP.

**DNS rebinding**
: Attack in which a website reaches a local server. Mitigated by validating `Origin` and binding to `127.0.0.1`.

## Authorization

**Resource server**
: A protected MCP server, in OAuth terms.

**Authorization server (AS)**
: Issues access tokens. May be separate from the MCP server.

**Protected Resource Metadata (PRM)**
: RFC 9728 document at `/.well-known/oauth-protected-resource`, listing `authorization_servers`. MCP servers MUST implement it.

**`WWW-Authenticate`**
: Header on a `401`/`403` carrying `resource_metadata` and optionally `scope` / `error="insufficient_scope"`.

**PKCE**
: Proof Key for Code Exchange. **MUST**, with `S256`. Clients must refuse to proceed if the AS doesn't advertise support.

**Resource indicator (`resource` parameter)**
: RFC 8707. **MUST** be sent in authorization and token requests, set to the MCP server's canonical URI.

**Audience validation**
: Server **MUST** only accept tokens issued for itself.

**Token passthrough**
: Forbidden anti-pattern: forwarding a client's token to an upstream API.

**Client ID Metadata Document (CIMD)**
: Client uses an HTTPS URL as its `client_id`, pointing to a JSON metadata document. The preferred registration method.

**Dynamic Client Registration (DCR)** *(deprecated)*
: RFC 7591 `POST /register` to obtain a client ID.

**Step-up authorization**
: Re-authorizing with the union of previous and newly challenged scopes after `insufficient_scope`.

**`iss` validation**
: RFC 9207 check that the authorization response came from the expected AS, preventing mix-up attacks.

## Security

**Confused deputy**
: A privileged intermediary (an MCP proxy) is tricked into using its authority for an attacker.

**SSRF**
: Server-side request forgery. A malicious server supplies URLs that make a client fetch internal resources.

**State handle hijacking** <span class="new-2026">2026</span>
: Guessing or stealing another user's explicit state handle. Mitigated by binding handles to the authenticated user.

**Tool poisoning**
: Hidden instructions in tool descriptions or metadata, aimed at the model.

**Rug pull**
: A server silently changes a tool's definition after the user approved it.

## Ecosystem

**AAIF**
: Agentic AI Foundation, a directed fund under the Linux Foundation, formed 9 December 2025. MCP is a founding project.

**SEP**
: Specification Enhancement Proposal. Types: Standards Track, Informational, Process, Extensions Track.

**MCP Registry**
: Official metadata registry for public servers (`server.json`), with namespace verification.

**MCP Inspector**
: Official developer tool for testing and debugging servers.

**Tier 1 SDKs**
: TypeScript, Python, C#, Go, Rust.
