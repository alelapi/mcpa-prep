---
title: Rapid-Fire Flashcards
---

# Rapid-Fire Flashcards

Short cards for the last few days before the exam. Use them to find gaps, not to learn topics for the first
time. Every section ends with a link back to its domain.

**How to use them:**

1. Read the question and answer it **out loud** before you expand the card.
2. Keep a list of the cards you miss.
3. Go back to the chapter behind every miss, then retry only those cards the next day.
4. The day before the exam, do one pass of the [numbers, codes and names](#numbers-codes-names-to-memorise)
   and the [legacy vs. modern](#legacy-vs-2026-07-28) tables.

---

## Domain 1: MCP Fundamentals

??? question "What problem does MCP solve, in one line?"
    The **M×N integration problem**: M AI applications × N systems needing custom connectors becomes **M + N**
    implementations with a shared protocol.

??? question "Which protocol does the MCP specification name as its inspiration?"
    The **Language Server Protocol (LSP)**. "USB-C for AI applications" is the documentation's analogy, not the
    spec's stated inspiration.

??? question "What three things does MCP standardise?"
    Sharing **context** with models, exposing **tools** and capabilities, and building **composable integrations**
    and workflows.

??? question "What are the three participants, and how many servers does one client talk to?"
    **Host** (the AI application), **client** (a connector inside the host), **server** (exposes capabilities).
    Each client talks to **exactly one** server.

??? question "Why one client per server?"
    **Isolation**: servers can't see each other's traffic, and the host keeps a clean boundary per server.

??? question "Which primitive is controlled by whom?"
    **Tools**: the model. **Resources**: the application. **Prompts**: the user.

??? question "A prompt is user-controlled. Who writes its content?"
    The **server**. "User-controlled" means the user decides **when** it's used.

??? question "Which client features exist, and what is their status?"
    **Elicitation**: active. **Sampling** and **roots**: deprecated.

??? question "What are the two layers of MCP?"
    **Data layer**: JSON-RPC messages, primitives, notifications. **Transport layer**: framing, delivery,
    authorisation (stdio, Streamable HTTP).

??? question "Which parts of the spec MUST every implementation support?"
    The **base protocol**, **versioning and compatibility**, and the **message patterns**. Everything else is optional.

??? question "Name four things MCP deliberately doesn't cover."
    How the host **uses the LLM** or manages context; **enforcement** of security; **UI patterns**; the host's own
    **model connection**. It also doesn't replace existing APIs.

??? question "What are the four design principles?"
    Servers **easy to build**; servers **composable**; servers **can't see the whole conversation or into other servers**;
    features added **progressively**.

??? question "Where does composition of several servers happen?"
    In the **host**. Servers never talk to each other.

??? question "What does a version string like `2026-07-28` mean?"
    The date of the **last backward-incompatible** change. Compatible changes keep the version.

??? question "What are the revision states?"
    **Draft** (in progress), **Current** (ready; may get compatible changes), **Final** (frozen).

??? question "What does 'stateless' mean for MCP?"
    Every request carries everything needed to process it. No handshake, no session, and an open connection (even
    stdio) is **not** a conversation.

??? question "Is a custom transport allowed?"
    Yes, if it preserves the **JSON-RPC format**, the **message patterns** and the **per-request metadata model**.
    Byte-stream transports SHOULD reuse stdio framing.

Review: [Domain 1](../domain-1/index.md)

---

## Domain 2: Architecture & Components

??? question "What is the source of truth for message structure?"
    **`schema.ts`**. `schema.json` is generated from it.

??? question "Who sends requests, responses and notifications?"
    **Requests**: clients only. **Responses**: servers only. **Notifications**: servers, plus `notifications/cancelled`
    from clients on stdio.

??? question "What are MCP's rules for request ids?"
    String or integer, **never `null`**, unique among in-flight requests. An MRTR retry uses a **new** id.

??? question "What are `resultType` absent vs. unrecognised?"
    **Absent**: treat as `"complete"` (legacy server). **Unrecognised**: treat as **invalid**.

??? question "Which `_meta` keys are required on every request?"
    `io.modelcontextprotocol/protocolVersion` and `io.modelcontextprotocol/clientCapabilities`. `clientInfo` is a SHOULD.

??? question "Missing a required `_meta` key: what error?"
    **`-32602`** Invalid params (**HTTP 400**).

??? question "Which `_meta` prefixes are reserved for MCP?"
    Any prefix whose **second label** is `modelcontextprotocol` or `mcp`, e.g. `io.modelcontextprotocol/`, `dev.mcp/`.
    `com.example.mcp/` is **not** reserved.

??? question "Can you make security decisions based on `clientInfo` or `serverInfo`?"
    **No.** Both are self-reported. Use them for display, logging and debugging only.

??? question "Which unprefixed `_meta` keys carry trace context?"
    `traceparent`, `tracestate`, `baggage` (W3C formats, for OpenTelemetry).

??? question "Default JSON Schema dialect?"
    **2020-12** when `$schema` is absent. Support for 2020-12 is mandatory.

??? question "Rule for `$ref` pointing to a network URI?"
    **MUST NOT** auto-dereference. Opt-in fetching must be off by default. Reject schemas with unresolved external refs.

??? question "Recommended `inputSchema` for a tool with no parameters?"
    `{"type": "object", "additionalProperties": false}`.

??? question "If a tool declares `outputSchema`, what must its result contain?"
    `structuredContent` that **conforms** to it (any JSON value). It SHOULD also include the JSON as a text block.

??? question "Tool annotation defaults?"
    `readOnlyHint: false`, `destructiveHint: true`, `idempotentHint: false`, `openWorldHint: true`. All **untrusted**
    unless the server is trusted.

??? question "`resource_link` vs. embedded `resource`?"
    **Link**: a URI to fetch if wanted (not guaranteed to be in `resources/list`). **Embedded**: contents inline.

??? question "What does the `audience` content annotation do?"
    Says who content is for: `["user"]`, `["assistant"]` or both. Hosts can keep user-only content out of the model's context.

??? question "What schema does form elicitation allow?"
    A **flat object of primitives**: string, number/integer, boolean, single- or multi-select enums. No nested objects.

??? question "What format do prompt arguments use?"
    A simple list of `{ name, description?, required? }`, **not** JSON Schema.

??? question "What does the model send to a server, and what does the server see?"
    The model sends nothing to servers. The **host** sends `tools/call` with the name, arguments and `_meta`. The
    server never sees the conversation.

??? question "What is `server/discover`'s `instructions` field?"
    Optional server-written guidance **for the model**. Trust it only as much as the server.

??? question "What are the stdio framing rules?"
    Newline-delimited JSON, **no embedded newlines**, **nothing but MCP on stdout**, logs to **stderr**.

??? question "How is a stdio server shut down?"
    **Close stdin**, wait, then SIGTERM → SIGKILL (POSIX) if needed.

??? question "Streamable HTTP in three facts?"
    **One endpoint**; **every message is a POST**; reply is **JSON or a request-scoped SSE stream**.

??? question "Which HTTP headers mirror the body, and when?"
    `MCP-Protocol-Version` (every POST), `Mcp-Method` (all requests), `Mcp-Name` (`tools/call`, `resources/read`, `prompts/get`).

??? question "What does `x-mcp-header` do, and on which types?"
    Mirrors a tool argument into `Mcp-Param-{Name}`. Only `string`, `integer`, `boolean` (**not `number`**). An invalid
    tool is excluded from the client's list.

??? question "What goes in `Mcp-Param-*` for a value with non-ASCII characters?"
    The Base64 sentinel: `=?base64?...?=`.

??? question "What are the three DNS rebinding defences for local HTTP servers?"
    **Validate `Origin`** (403 if invalid), **bind to 127.0.0.1**, **authenticate**.

Review: [Domain 2](../domain-2/index.md)

---

## Domain 3: Interactions & Execution

??? question "What are the three message patterns?"
    **Request/response**, **Multi Round-Trip Requests**, **subscribe/notify**.

??? question "What does a server do in `2026-07-28` when it needs user input mid-request?"
    Returns an **`InputRequiredResult`** (`resultType: "input_required"`) with `inputRequests` and/or `requestState`.

??? question "Which requests may return `InputRequiredResult`?"
    **`tools/call`**, **`resources/read`**, **`prompts/get`**. Nothing else.

??? question "What may `inputRequests` contain?"
    `elicitation/create`, and the deprecated `sampling/createMessage` and `roots/list`.

??? question "What are the client's MUSTs on an MRTR retry?"
    **New id**; same method and arguments; `inputResponses` with matching keys; **echo `requestState` exactly** without
    inspecting it; don't reuse it for other requests.

??? question "What are the server's MUSTs on `requestState`?"
    Treat it as **attacker-controlled**; **HMAC/AEAD** when it affects authorisation or business logic; **reject** on failure.

??? question "What should `requestState` be bound to, to limit replay?"
    The **authenticated principal**, a **short TTL**, and the **originating request**. Single use still needs server-side enforcement.

??? question "The retry lacks required input. What does the server return?"
    A **new `InputRequiredResult`**, not an error.

??? question "Can MRTR retry results be cached?"
    **No.**

??? question "Form vs. URL elicitation?"
    **Form**: structured, non-sensitive data through the client. **URL**: sensitive interactions out of band (secrets,
    payments, third-party OAuth).

??? question "What does `elicitation: {}` mean?"
    **Form mode only.** A missing `mode` on a request also means form.

??? question "Elicitation actions?"
    **accept** (submitted), **decline** (explicit no), **cancel** (dismissed without choosing).

??? question "What does `accept` mean for URL mode?"
    The user **consented to open** the URL, not that the interaction finished.

??? question "What are the client MUSTs for URL elicitation?"
    Don't pre-fetch; don't open without consent; show the full URL; open it where the client and LLM **can't inspect** it.

??? question "How do you prevent the URL-elicitation account takeover?"
    The server verifies that the user **completing** the flow is the user who **started** it.

??? question "Protocol error vs. tool execution error?"
    **Protocol error**: JSON-RPC `error` about the request. **Tool execution error**: result with **`isError: true`** about
    the operation, for the model to self-correct.

??? question "An input validation error (e.g. a date in the past) is which kind?"
    **Tool execution error** (`isError: true`).

??? question "Unknown tool, per the spec?"
    **`-32602`** Invalid params.

??? question "Resource not found in `2026-07-28`?"
    **`-32602`**. Accept legacy `-32002`. Never return empty `contents`.

??? question "What does `-32602` cover?"
    Missing `_meta`, unknown tool, invalid prompt name or missing prompt args, resource not found, invalid cursor,
    invalid log level.

??? question "`completion/complete` on a server without `completions`?"
    **`-32601`**.

??? question "Which results must carry cache hints?"
    `server/discover`, `tools/list`, `prompts/list`, `resources/list`, `resources/templates/list`, `resources/read`.

??? question "What do `ttlMs` = 0, absent and negative mean?"
    All mean **stale**; treat as `0`. Servers must send `ttlMs >= 0`.

??? question "`cacheScope` public vs. private?"
    **public**: shareable across users and gateways. **private**: same authorisation context only; a different token means a different cache.

??? question "Can `cacheScope` replace access control?"
    **No.** Servers MUST enforce per-primitive access control.

??? question "Is `ttlMs` a polling interval?"
    **No.** Check freshness when you need the data. If you do poll, use jitter and backoff.

??? question "What are the pagination rules?"
    Opaque cursors; the server picks page size; **missing `nextCursor` = end**; **empty-string cursor is valid**; invalid
    cursor → `-32602`.

??? question "Is completion for tool arguments?"
    **No.** Prompt arguments and resource template parameters only. Max **100** values.

??? question "What replaced `resources/subscribe` and the GET stream?"
    **`subscriptions/listen`** with a `notifications` filter.

??? question "What is the first message on a listen stream?"
    `notifications/subscriptions/acknowledged`, listing the **subset** the server will honour.

??? question "What is the value of `io.modelcontextprotocol/subscriptionId`?"
    The **JSON-RPC id of the `subscriptions/listen` request**.

??? question "Do progress notifications go on the listen stream?"
    **No.** `progress` and log messages stay on their **own request's** stream.

??? question "What are the progress rules?"
    Client sends `progressToken` (unique among active requests); `progress` **MUST increase** even without `total`;
    notifications stop at completion.

??? question "Cancellation on stdio vs. Streamable HTTP?"
    **stdio**: `notifications/cancelled` with `requestId`. **HTTP**: **close the response stream**.

??? question "When may a server send `notifications/cancelled`?"
    **Only** to end a `subscriptions/listen` stream.

??? question "What are the timeout rules?"
    Time out every request; MAY reset the clock on progress; **always enforce a maximum**.

??? question "A Streamable HTTP response stream breaks. What now?"
    The request is lost. **Re-issue with a new id**, and watch idempotency.

??? question "`tools/list` may vary by...?"
    **Authorisation** on the request, yes. **Connection** or earlier requests, **no**.

??? question "How do servers keep state across calls?"
    **Explicit handles** returned by tools and passed back as arguments, validated against the caller on every call.

??? question "Deprecated ≠ removed. Earliest removal for sampling, roots and logging?"
    First revision **on or after 2027-07-28**. Deprecated features still work.

??? question "Sampling: what is required, and what are hints?"
    **`maxTokens`** is required and must be respected. **Hints** are substrings; the client may map them to other providers
    and makes the final choice.

??? question "Are roots a security boundary?"
    **No.** They're advisory, and `file://` only. Real restriction comes from the OS or a sandbox.

??? question "How do clients get log messages now?"
    Per request with `io.modelcontextprotocol/logLevel`. No messages without it. Better: **stderr** and **OpenTelemetry**.

Review: [Domain 3](../domain-3/index.md)

---

## Domain 4: Security & Governance

??? question "Can MCP enforce consent?"
    **No.** The **host** must build consent and authorisation flows.

??? question "Why are the model's tool calls only proposals?"
    The model is **steerable** by any text in its context, including attacker-controlled text.

??? question "What must a host do before a tool call, to catch exfiltration?"
    **Show the tool inputs** to the user and let a human deny it.

??? question "What are the three ingredients of the dangerous combination?"
    **Private data** + **untrusted content** + **a way to send data out**.

??? question "Is authorisation required?"
    It's **optional**. HTTP **SHOULD** use it; stdio **SHOULD NOT** (credentials from the environment).

??? question "What OAuth role does the MCP server play?"
    **Resource server.** The MCP client is the OAuth client.

??? question "What does the server's 401 carry?"
    `WWW-Authenticate` with `resource_metadata` (and optionally `scope`).

??? question "What must Protected Resource Metadata include?"
    `authorization_servers`, with at least one AS. MCP servers MUST implement PRM.

??? question "What check follows fetching AS metadata?"
    Its **`issuer` must equal** the issuer used to build the URL.

??? question "What is the client registration priority?"
    Pre-registered → **CIMD** → DCR (deprecated) → ask the user.

??? question "What is a CIMD `client_id`?"
    An **HTTPS URL with a path** serving the client's metadata JSON; the document's `client_id` must match the URL exactly.

??? question "What are the PKCE rules?"
    **MUST** use it, `S256` when capable; **refuse to proceed** if `code_challenge_methods_supported` is absent.

??? question "What are the `resource` parameter rules?"
    **MUST** be sent in the authorisation **and** token requests, set to the server's **canonical URI**, even if the AS ignores it.

??? question "What stops mix-up attacks?"
    **Validating `iss`** (RFC 9207) against the recorded issuer. PKCE doesn't.

??? question "Where does the token go?"
    `Authorization: Bearer` on **every** request. **Never** in the query string.

??? question "What must the server validate about a token?"
    That it's valid and issued **for this server** (audience). It must never accept or pass through other tokens.

??? question "Invalid token vs. insufficient scope?"
    **401** vs. **403** with `error="insufficient_scope"` and `scope`.

??? question "What does the client do on step-up?"
    Re-authorises with the **union** of old and new scopes; retries a limited number of times.

??? question "Should MCP servers ask for `offline_access`?"
    **No** (SHOULD NOT). Clients MAY request it if the AS lists it.

??? question "Machine-to-machine vs. corporate employees: which extensions?"
    **OAuth Client Credentials** (JWT assertions recommended) vs. **Enterprise-Managed Authorization** (IdP issues an ID-JAG).

??? question "What are the confused deputy conditions?"
    Proxy with a **static client ID**, **dynamic registration**, a third-party **consent cookie**, and **no per-client consent**.

??? question "What is the confused deputy fix?"
    **Per-client consent** in the proxy before the third-party flow; exact `redirect_uri` match; `state` set only after consent.

??? question "What is token passthrough?"
    Accepting a token not issued to you and forwarding it upstream. **Forbidden.** Use separate upstream tokens.

??? question "SSRF in MCP: where, and what defences?"
    Clients fetching server-supplied OAuth URLs. HTTPS, **block private and link-local** (`169.254.0.0/16`), validate
    redirects, egress proxies, beware DNS TOCTOU.

??? question "How do you mitigate state handle hijacking?"
    Random handles, **bound to the authenticated user** from the token; possession is never authentication.

??? question "What does one-click local server install require?"
    A consent dialog showing the **exact, untruncated command**, marked as dangerous, with explicit approval.

??? question "How must a client open an OAuth authorisation URL?"
    Only `http`/`https` schemes, and **never via a shell**.

??? question "What are tool poisoning and a rug pull?"
    **Poisoning**: hidden instructions in tool metadata. **Rug pull**: a definition changing after approval. Bind approvals to definitions.

??? question "What does Registry namespace verification prove?"
    **Who published**, not that the code is safe.

??? question "What must logs never contain?"
    Credentials or secrets, PII, or internal details that aid attacks.

??? question "What is the authoritative identity for audit?"
    **Validated token claims** (`sub`, `client_id`, scopes), not `clientInfo`.

??? question "How are tool execution errors made visible in OpenTelemetry?"
    `error.type = tool_error` when `isError: true`.

Review: [Domain 4](../domain-4/index.md)

---

## Domain 5: Use Cases & Ecosystem

??? question "When did MCP join the Agentic AI Foundation, and with which projects?"
    **9 December 2025**, under the Linux Foundation, alongside **goose** (Block) and **AGENTS.md** (OpenAI).

??? question "What are MCP's licences?"
    **Apache 2.0** for code and specs; **CC-BY 4.0** for documentation.

??? question "What is the governance hierarchy?"
    **Lead Maintainers** → **Core Maintainers** → **Maintainers** → Contributors. Membership is for **individuals**, not companies.

??? question "Interest Group vs. Working Group?"
    **IG**: discusses problems (non-binding). **WG**: builds solutions (binding).

??? question "What are the SEP types?"
    **Standards Track**, **Informational**, **Process**, **Extensions Track**.

??? question "What does a SEP need to be accepted, and to reach Final?"
    **Accepted**: a sponsor and a prototype implementation. **Final**: a reference implementation and, for observable
    behaviour, a **conformance test**.

??? question "Dormant vs. rejected?"
    **Dormant** = no sponsor within 6 months; can be revived. Not a rejection.

??? question "Which SDKs are in which tier?"
    **Tier 1**: TypeScript, Python, C#, Go, Rust. **Tier 2**: Java, Ruby. **Tier 3**: Swift, PHP, Kotlin.

??? question "What are the Tier 1 requirements?"
    **100%** conformance, new features **before** spec release, triage in **2 business days**, critical fixes in **7 days**.

??? question "What does the MCP Registry store?"
    **Metadata** (`server.json`), not code. It's in preview, with verified reverse-DNS namespaces.

??? question "Who should consume the Registry?"
    **Aggregators and subregistries**, not hosts directly. Private servers need your own registry.

??? question "How is npm vs. PyPI package ownership verified?"
    **npm**: `mcpName` in `package.json`. **PyPI**: `mcp-name: <name>` in the README.

??? question "What are the three MCP Inspector modes?"
    **Web** (default), **CLI** (CI and scripts), **TUI**.

??? question "What are extension identifiers, and how are extensions negotiated?"
    `vendor-prefix/name`, official ones under `io.modelcontextprotocol/`; negotiated via the **`extensions`** capability
    map; opt-in; fall back or reject when unsupported.

??? question "What does Tasks give you, and which methods?"
    Durable `taskId` (`resultType: "task"`); **`tasks/get`** polling, **`tasks/update`** for input, **`tasks/cancel`** (cooperative).

??? question "How do MCP Apps render?"
    A `ui://` resource in a **sandboxed iframe**, talking over **postMessage**. Always provide a text fallback.

??? question "What are the Skills integrity rules?"
    Verify **SHA-256 digests**; approval is bound to all files, so any change revokes it; skill content is untrusted.

??? question "What are progressive discovery and code mode?"
    **Progressive discovery**: search tools, load schemas on demand (catalog → inspect → execute), above ~1–5% of context.
    **Code mode**: the model writes a script in a sandbox, a host broker executes the calls, and only the final output returns.

??? question "Why should servers return tools in deterministic order?"
    Client caching, and **prompt cache** hits at the model provider.

??? question "What is portable across hosts, and what isn't?"
    **Portable**: messages, primitives, capabilities, transports, auth, errors. **Not**: config files, UI, optional features,
    tool selection, approval policy.

Review: [Domain 5](../domain-5/index.md)

---

## Numbers, codes & names to memorise {#numbers-codes-names-to-memorise}

| Item | Value |
| --- | --- |
| `-32700` / `-32600` / `-32601` / `-32602` / `-32603` | Parse error / Invalid Request / Method not found / Invalid params / Internal error |
| **`-32020`** | HeaderMismatch (HTTP 400) |
| **`-32021`** | MissingRequiredClientCapability (HTTP 400, `data.requiredCapabilities`) |
| **`-32022`** | UnsupportedProtocolVersion (HTTP 400, `data.supported`) |
| `-32002` | Retired resource-not-found: accept it, never emit it |
| `-32042` | Retired "URL elicitation required" |
| `-32020`…`-32099` / `-32000`…`-32019` | Reserved for the spec / legacy implementation-defined |
| HTTP 202 / 401 / 403 / 404 / 405 | Notification accepted / auth / Origin or scope / unknown method / legacy GET or DELETE |
| `resultType` values | `complete`, `input_required` (+ extension values such as `task`) |
| Cacheable fields | `ttlMs` (milliseconds), `cacheScope` (`public` or `private`) |
| Completion limit | 100 values |
| Deprecation window | ≥ 12 months (90 days expedited); earliest removal on or after 2027-07-28 |
| RFC 9728 / 8414 / 8707 / 9207 / 7591 / 6570 | PRM / AS metadata / resource indicators / `iss` / DCR / URI templates |
| Extension ids | `io.modelcontextprotocol/tasks`, `/ui`, `/skills`, `/oauth-client-credentials`, `/enterprise-managed-authorization` |
| SEPs | 2575 stateless · 2322 MRTR · 2567 sessionless handles · 2577 deprecations · 2596 feature lifecycle · 2663 Tasks extension |
| Progressive discovery threshold | ~1–5% of the context window |
| Tier 1 SLAs | 100% conformance · triage 2 business days · critical fix 7 days |

## Legacy vs. 2026-07-28 {#legacy-vs-2026-07-28}

| Legacy (`2025-11-25` and earlier) | Modern (`2026-07-28`) |
| --- | --- |
| `initialize` + `notifications/initialized` | Per-request `_meta`; optional `server/discover` |
| Sessions, `Mcp-Session-Id` | Stateless; explicit handles |
| Server sends `elicitation/create` as a request | `InputRequiredResult` + client retry |
| No `resultType` | `resultType` on every result |
| GET SSE stream, `resources/subscribe` | `subscriptions/listen` |
| `ping` | Removed |
| `logging/setLevel` | Per-request `logLevel` in `_meta` |
| `notifications/roots/list_changed` | Removed |
| Resumable SSE (`Last-Event-ID`) | Not resumable; new id |
| `notifications/elicitation/complete`, `elicitationId` | Removed; retry and `requestState` |
| Tasks in core (`tasks/result`, `tasks/list`) | Tasks extension (`tasks/get`, `tasks/update`) |
| Resource not found `-32002` | `-32602` |
| Dynamic Client Registration normal | DCR deprecated; CIMD preferred |
