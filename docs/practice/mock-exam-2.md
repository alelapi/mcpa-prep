---
title: Mock Exam 2
---

# Mock Exam 2

**60 questions · 90 minutes · target 85% (51/60)**

A second, independent exam with the same blueprint weights as [Mock Exam 1](mock-exam-1.md) and no repeated
questions. Keep it for your final week, and take it once, under real conditions: a timer, the course
closed, and answers written down before you open any of them.

Most questions have one correct answer. Questions marked **(Select TWO)** have exactly two. Score one point
only if you picked both. The [answer key](#answer-key) and a per-domain breakdown are at the bottom.

---

**1.** <span class="domain-tag">D2</span> The prose specification and `schema.ts` seem to disagree about
the shape of a message field. Which is authoritative for message structure?

- A. The prose specification, because it contains the normative MUST/SHOULD language
- B. The generated `schema.json`, because tooling uses it
- C. `schema.ts`, the TypeScript schema, which is the source of truth; `schema.json` is generated from it
- D. Whichever the Tier 1 SDKs implement

??? success "Answer"
    **C.** The spec states that `schema.ts` is the **source of truth** for all protocol messages and
    structures, and a JSON Schema is generated from it for tooling. When the prose and the schema appear to
    disagree about structure, the schema wins. SDKs implement the spec; they don't define it.
    [Schemas & Structured Data](../domain-2/schemas.md)

---

**2.** <span class="domain-tag">D3</span> A native iOS host implements URL-mode elicitation by loading the
server's URL inside an embedded `WKWebView` that the app can script and read. What is wrong?

- A. Nothing, as long as the user consented first
- B. URL-mode elicitation isn't supported on mobile
- C. The client must pre-fetch the page to check it first
- D. The URL must be opened in a way that doesn't let the client or the LLM inspect the page or the user's input; a system browser view such as `SFSafariViewController` is acceptable, `WKWebView` is not

??? success "Answer"
    **D.** URL mode exists so sensitive input never passes through the client. Clients MUST open the URL
    securely, so neither the client nor the LLM can inspect the page. Consent is also required, but it isn't
    enough on its own. Clients MUST NOT pre-fetch the URL.
    [MRTR & Elicitation](../domain-3/mrtr-elicitation.md)

---

**3.** <span class="domain-tag">D1</span> A user asks a travel assistant to plan a trip. The host uses a
`plan-vacation` prompt from a travel server, the user's calendar as a resource from a calendar server, and
tools from a weather server and an email server. No server knows the others exist. Which of MCP's stated
goals does this illustrate most directly?

- A. Building composable integrations and workflows
- B. Sharing contextual information with language models
- C. Replacing the model provider's API
- D. Enforcing security at the protocol level

??? success "Answer"
    **A.** The spec says MCP standardises how applications share context, expose tools, and **build composable
    integrations and workflows**. This scenario is composition: several focused servers combined by the
    **host**. Context sharing is one ingredient, but the combination of independent servers is the point.
    [Interoperability & Value](../domain-1/interoperability.md)

---

**4.** <span class="domain-tag">D4</span> A remote server identifies users by reading an `email` argument
that each tool call includes, for example `"email": "joe@example.com"`, and returns that user's records.
What is the flaw?

- A. Servers MUST NOT rely on client-provided identification without verification; user identity should come from authorization, such as the validated token's `sub` claim
- B. Email addresses must be sent in `_meta`, not arguments
- C. The server should read `clientInfo.name` instead
- D. There is no flaw if the tool is annotated `readOnlyHint: true`

??? success "Answer"
    **A.** Anything the client or model sends can be forged. The spec says servers MUST NOT rely on
    unverified identification, and for remote servers identity SHOULD come from credentials acquired via MCP
    authorization. `clientInfo` is self-reported too, and annotations don't create security.
    [Trust Boundaries](../domain-4/trust-boundaries.md)

---

**5.** <span class="domain-tag">D3</span> A server declares support for elicitation. A client's requests
declare `"elicitation": {}`. The server wants to send the user to a URL to enter an API key. What should it
do?

- A. Send a URL-mode elicitation, since the client declared elicitation
- B. Not send URL mode, because an empty `elicitation` object means form mode only; if it can't proceed without it, return `-32021` MissingRequiredClientCapability
- C. Ask for the API key in a form-mode elicitation instead
- D. Put the URL in a form-mode field as a clickable link

??? success "Answer"
    **B.** For backward compatibility, `elicitation: {}` equals `{ "form": {} }`. Servers MUST NOT send a mode
    the client didn't declare. Asking for an API key in form mode is forbidden, and servers SHOULD NOT put
    clickable URLs in form elicitations. When a required capability is missing, the server returns `-32021`.
    [MRTR & Elicitation](../domain-3/mrtr-elicitation.md)

---

**6.** <span class="domain-tag">D5</span> Who changes a SEP's status, for example from `draft` to
`in-review`?

- A. The SEP's author, by editing the file
- B. Any contributor who has reviewed it
- C. The Linux Foundation's legal team
- D. The SEP's sponsor, a Core Maintainer or Maintainer who champions it

??? success "Answer"
    **D.** A SEP needs a **sponsor**, and the sponsor updates its status. Authors don't change status
    themselves. A SEP that finds no sponsor within six months becomes `dormant`, which is not the same as
    rejected.
    [Roles, Responsibilities & Adoption](../domain-5/roles-adoption.md)

---

**7.** <span class="domain-tag">D4</span> An MCP proxy stores each user's consent to an MCP client in a
cookie. Which cookie design follows the security best practices?

- A. A cookie named `consented=true`, valid for all clients, so users aren't asked twice
- B. A `localStorage` entry readable by the consent page's JavaScript
- C. A `__Host-` prefixed cookie with `Secure`, `HttpOnly` and `SameSite=Lax`, cryptographically signed, and bound to the specific `client_id`
- D. A `SameSite=None` cookie so the third-party authorization server can read it

??? success "Answer"
    **C.** Consent must be recorded **per user and per `client_id`**. If cookies are used, they need the
    `__Host-` prefix, `Secure`, `HttpOnly`, `SameSite=Lax`, a signature (or server-side storage), and a
    binding to the client. A general "user has consented" flag is exactly what lets an attacker's newly
    registered client skip consent.
    [Permissions & Consent](../domain-4/permissions-consent.md)

---

**8.** <span class="domain-tag">D3</span> A document server wants the user to confirm before it returns a
sensitive file through `resources/read`. Can it use MRTR for this?

- A. No: only `tools/call` may return an `InputRequiredResult`
- B. No: resources are application-controlled, so they can't involve the user
- C. Yes: `resources/read` is one of the three requests that may return an `InputRequiredResult`, along with `tools/call` and `prompts/get`
- D. Yes, but only through a `resources/subscribe` request

??? success "Answer"
    **C.** `tools/call`, `resources/read` and `prompts/get` may all return `InputRequiredResult`, so a server can
    elicit confirmation before revealing a document. Every other request MUST NOT.
    `resources/subscribe` doesn't exist in `2026-07-28`.
    [Resources & Prompts](../domain-3/resources-prompts.md)

---

**9.** <span class="domain-tag">D1</span> Which statement about JSON-RPC notifications in MCP is correct?

- A. A notification has an `id` so the receiver can acknowledge it
- B. A notification has no `id`, and the receiver MUST NOT respond to it
- C. Servers must answer client notifications with an empty result
- D. Notifications are only used over Streamable HTTP

??? success "Answer"
    **B.** Requests carry an `id` and expect a response. Notifications are one-way: no `id`, and no response
    allowed. Both transports carry them. On HTTP an accepted notification POST gets `202 Accepted` with no
    body, which is an HTTP acknowledgment, not a JSON-RPC response.
    [Core Concepts](../domain-1/core-concepts.md)

---

**10.** <span class="domain-tag">D2</span> Which `_meta` keys are an explicit exception to the rule that
MCP's own keys use a reserved prefix such as `io.modelcontextprotocol/`?

- A. `progressToken` only
- B. `serverInfo` and `clientInfo`
- C. `traceparent`, `tracestate` and `baggage`, reserved for OpenTelemetry trace context
- D. `subscriptionId` and `logLevel`

??? success "Answer"
    **C.** The W3C Trace Context and Baggage keys are written **unprefixed** for OpenTelemetry compatibility.
    `clientInfo`, `serverInfo`, `subscriptionId` and `logLevel` all live under `io.modelcontextprotocol/`.
    (`progressToken` is also unprefixed, but it isn't the exception this question describes.)
    [JSON-RPC Messages & `_meta`](../domain-2/messages.md)

---

**11.** <span class="domain-tag">D4</span> A client validates that an OAuth metadata URL resolves to a
public IP address, then makes the HTTP request. An attacker's DNS answers with a public IP for the check and
`169.254.169.254` for the actual request. What is this weakness, and the recommended defence?

- A. A mix-up attack; validate `iss`
- B. Token passthrough; validate the audience
- C. Clickjacking; send `X-Frame-Options: DENY`
- D. A DNS time-of-check/time-of-use gap in SSRF protection; pin DNS resolution between the check and the request, and layer defences such as egress proxies

??? success "Answer"
    **D.** A domain can resolve to a safe IP when checked and an internal IP when used. The SSRF guidance says
    to pin the resolution between check and use, and not to rely on one layer: block private and link-local
    ranges, validate redirects, and use egress proxies.
    [Attacks & Mitigations](../domain-4/attacks-mitigations.md)

---

**12.** <span class="domain-tag">D3</span> A server's `tools/list` returns extra tools on a connection if an
earlier request on that same connection called `enable_admin_mode`. Another server returns extra tools
when the bearer token on the request carries an `admin` scope. Which is allowed?

- A. Only the second: lists MUST NOT vary per connection or as a side effect of other requests, but MAY vary by the authorization presented on the request
- B. Only the first: connection state is how lists are personalised
- C. Both
- D. Neither: `tools/list` must return the same list to every caller

??? success "Answer"
    **A.** Credentials are per-request input, so filtering by the token on this request is still stateless.
    Changing the list because of a previous request on the connection is connection state, which the
    stateless protocol forbids. A per-user list must also be cached as `private`.
    [Tools & the Invocation Lifecycle](../domain-3/tools.md)

---

**13.** <span class="domain-tag">D5</span> Where does the number in "SEP-2575" come from?

- A. The pull request number that introduced the SEP
- B. A sequential counter maintained by the Core Maintainers
- C. The date the SEP was accepted
- D. The GitHub issue number where it was first discussed

??? success "Answer"
    **A.** SEPs live as Markdown files in the `seps/` directory of the specification repository and are
    **numbered by their pull request**. PR #1850 becomes SEP-1850.
    [Roles, Responsibilities & Adoption](../domain-5/roles-adoption.md)

---

**14.** <span class="domain-tag">D1</span> Sentry runs an MCP server on its own infrastructure that users
reach over the internet. A colleague says, "That isn't really an MCP server; MCP servers run locally."
What's the correct response?

- A. Correct: MCP servers must be launched by the client as subprocesses
- B. Correct: remote endpoints are called MCP gateways
- C. Incorrect: "server" means the program serving context, which can be local (typically stdio) or remote (typically Streamable HTTP)
- D. Incorrect, but only because Sentry uses the HTTP+SSE transport

??? success "Answer"
    **C.** An MCP server is a program that exposes capabilities, wherever it runs. A filesystem server launched
    by a desktop app is local; Sentry's hosted server is remote. HTTP+SSE is deprecated, so D is wrong on its
    own terms.
    [Core Concepts](../domain-1/core-concepts.md)

---

**15.** <span class="domain-tag">D3</span> A client sends a `tools/call` whose `arguments` field is the
string `"city=Paris"` instead of an object. How should the server report this?

- A. As a tool execution error with `isError: true`, so the model can self-correct
- B. As a protocol error, because the request doesn't satisfy the `CallToolRequest` schema
- C. As an `InputRequiredResult` asking for the city
- D. As `-32020` HeaderMismatch

??? success "Answer"
    **B.** A request whose structure fails `CallToolRequest` is a **protocol error**. Tool execution errors are
    for problems with argument **values** the tool can explain (a past date, a value out of range). The line
    is between a malformed request envelope and a well-formed request whose content the tool rejects.
    [Error Handling](../domain-3/error-handling.md)

---

**16.** <span class="domain-tag">D4</span> A desktop client opens OAuth authorization URLs by running
`cmd.exe /c start <url>`. A malicious server supplies an authorization endpoint containing shell
metacharacters. What is the risk, and what do the security best practices require?

- A. Only a broken redirect; the client should retry
- B. Mix-up; validate `iss`
- C. Command injection leading to remote code execution; clients MUST NOT use shell commands to open URLs, and MUST allow only `http`/`https` schemes
- D. Nothing, as long as PKCE is used

??? success "Answer"
    **C.** Opening a server-supplied URL through a shell lets metacharacters run extra commands. Clients MUST NOT
    open URLs via shell commands (use non-shell platform APIs), MUST allow only `http://` and `https://`, and MUST
    reject schemes such as `javascript:` and `file:`. PKCE and `iss` checks protect the OAuth exchange, not the
    client's process.
    [Attacks & Mitigations](../domain-4/attacks-mitigations.md)

---

**17.** <span class="domain-tag">D2</span> A tool's `inputSchema` has a property
`"retries": {"type": "number", "x-mcp-header": "Retries"}`. What must a client do with this tool?

- A. Send `Mcp-Param-Retries` with the number formatted as a decimal
- B. Base64-encode the number with the sentinel format
- C. Reject the whole `tools/list` result and disconnect
- D. Treat the tool definition as invalid, exclude it from its tools list, and SHOULD log a warning, while keeping the server's other valid tools

??? success "Answer"
    **D.** `x-mcp-header` may only annotate `string`, `integer` or `boolean` properties, **not `number`**. A tool
    that violates the constraints is invalid: the client MUST exclude it, SHOULD log a warning, and continues
    using the rest.
    [Transports](../domain-2/transports.md)

---

**18.** <span class="domain-tag">D5</span> **(Select TWO)** Which of these are supported `registryType`
values for packages listed in the MCP Registry?

- A. `npm`
- B. `maven`
- C. `oci`
- D. `homebrew`
- E. `apt`

??? success "Answer"
    **A and C.** Supported package types are `npm` (registry.npmjs.org), `pypi` (pypi.org), `nuget`, `oci`
    (Docker/OCI images), `cargo` (crates.io) and `mcpb` (prebuilt MCP bundles). The registry stores metadata
    pointing to these package registries, not code.
    [Ecosystem & Portability](../domain-5/ecosystem-portability.md)

---

**19.** <span class="domain-tag">D3</span> A server offers a one-time voucher redemption. Its `requestState`
is AEAD-sealed and bound to the user, a 5-minute TTL and the originating request. The same user replays one
valid state twice within 5 minutes and redeems the voucher twice. What was missing?

- A. Nothing: the protections guarantee single use
- B. A longer TTL
- C. HMAC instead of AEAD
- D. Server-side single-use enforcement, such as recording a nonce as consumed; binding limits replay but doesn't guarantee single use

??? success "Answer"
    **D.** Principal, TTL and request binding stop cross-user, stale and cross-request reuse. They don't stop the
    same user replaying the same state within its TTL. For at-most-once operations, the server MUST enforce
    single use itself. Switching between HMAC and AEAD changes nothing about replay.
    [MRTR & Elicitation](../domain-3/mrtr-elicitation.md)

---

**20.** <span class="domain-tag">D4</span> A protected resource's metadata lists two authorization servers.
The client registered with the first one last month. Which statement is correct?

- A. The client chooses an AS; each is independent, so it MUST keep separate registration state and tokens per AS and MUST NOT assume credentials from one work with the other
- B. The client must use both and merge the tokens
- C. The client can reuse its `client_id` from the first AS with the second
- D. Listing two ASes is invalid; PRM must list exactly one

??? success "Answer"
    **A.** PRM MUST list at least one AS and may list several; the client chooses. Client identifiers are
    unique to the AS that issued them, so registrations and tokens are kept per AS. Pre-registered or persisted
    credentials are keyed by the AS issuer.
    [Authorization with OAuth 2.1](../domain-4/authorization.md)

---

**21.** <span class="domain-tag">D1</span> Which of these is **NOT** one of the properties MCP's
interoperability depends on?

- A. A precise specification with normative language and a schema
- B. Capability negotiation, so implementations with different feature sets work on common ground
- C. Versioning and compatibility rules, including cross-era behaviour
- D. A single mandatory reference implementation that every host and server must embed

??? success "Answer"
    **D.** Interoperability rests on a **precise spec**, **capability negotiation** and **versioning rules**, with
    conformance tests making claims verifiable. There is no mandatory shared implementation; independent SDKs
    in many languages interoperate because they follow the same spec.
    [Interoperability & Value](../domain-1/interoperability.md)

---

**22.** <span class="domain-tag">D3</span> A client calls `resources/read` for `file:///missing.txt`. The
server returns `{"resultType": "complete", "contents": []}`. What is wrong?

- A. Nothing: an empty array clearly means not found
- B. The server MUST NOT return empty `contents` for a non-existent resource; it should return JSON-RPC error `-32602`
- C. It should return `-32002`, the current not-found code
- D. It should return `isError: true`

??? success "Answer"
    **B.** An empty `contents` array is ambiguous, because it could mean the resource exists but is empty.
    Not found is **`-32602`** in `2026-07-28`. `-32002` is the retired code, which clients should still accept
    from older servers but implementations MUST NOT emit. `isError` belongs to tool results.
    [Resources & Prompts](../domain-3/resources-prompts.md)

---

**23.** <span class="domain-tag">D5</span> A server author publishes version `1.2.0` of their server's
metadata to the MCP Registry, then notices a typo in the description. What can they do?

- A. Publish a new version; published version metadata is immutable
- B. Edit `1.2.0` in place through the REST API
- C. Delete `1.2.0` permanently and republish it with the same version
- D. Ask an aggregator to fix the typo in its copy

??? success "Answer"
    **A.** Version metadata is **immutable** once published. Updates mean a new version. The `status` field can
    change to `deprecated` or `deleted`, but deleted servers are hidden from default listings, never
    permanently removed.
    [Ecosystem & Portability](../domain-5/ecosystem-portability.md)

---

**24.** <span class="domain-tag">D4</span> A local MCP server listens over HTTP on `127.0.0.1:9000` so
several desktop apps can share it. Which measure best prevents **other local processes** from using it
without permission?

- A. Setting `cacheScope: "private"` on all results
- B. Returning tools in deterministic order
- C. Requiring an authorization token, or using stdio or Unix domain sockets / restricted IPC instead of an open TCP port
- D. Declaring `readOnlyHint: true` on every tool

??? success "Answer"
    **C.** The best practices say local servers SHOULD prevent unauthorised use by other processes: use
    **stdio** to limit access to the launching client, or if HTTP is needed, **require an authorization token**
    or use restricted IPC. Loopback binding keeps remote hosts out but not local processes. The other options
    aren't access controls.
    [Attacks & Mitigations](../domain-4/attacks-mitigations.md)

---

**25.** <span class="domain-tag">D2</span> A tool returns a detailed 200-line report for the user and a
three-line summary for the model. How can the server tell the host which is which?

- A. Put the report in `structuredContent` and the summary in `content`
- B. Send the report as a `notifications/message`
- C. Mark the report `isError: true`
- D. Use content annotations: `audience: ["user"]` on the report and `audience: ["assistant"]` on the summary

??? success "Answer"
    **D.** The `audience` annotation says who content is for. A host can show `["user"]` content without adding
    it to the model's context, which saves tokens. `structuredContent` is for machine-readable data, logging is
    deprecated and request-scoped, and `isError` signals failure.
    [Schemas & Structured Data](../domain-2/schemas.md)

---

**26.** <span class="domain-tag">D3</span> Which statement describes how and when JSON-RPC error codes may
be allocated in `2026-07-28`?

- A. `-32020` to `-32099` is reserved for the MCP specification; implementations MUST NOT emit undefined codes there, `-32000` to `-32019` is legacy, and new application-defined codes SHOULD be allocated outside `-32768` to `-32000`
- B. Any code between `-32000` and `-32099` may be used freely for application errors
- C. Implementations should allocate new codes in `-32000` to `-32019`
- D. MCP forbids application-defined error codes entirely

??? success "Answer"
    **A.** MCP partitions the JSON-RPC server-error range: `-32020`…`-32099` for the spec only, with defined
    meanings; `-32000`…`-32019` legacy, with no new allocations. New error codes for other purposes should sit
    outside the whole reserved `-32768`…`-32000` range.
    [Error Handling](../domain-3/error-handling.md)

---

**27.** <span class="domain-tag">D1</span> A company wants MCP between two components inside one
application to run over a Unix domain socket instead of stdio or HTTP. What does the specification allow?

- A. Nothing: only stdio and Streamable HTTP are permitted
- B. A custom transport, provided it preserves the JSON-RPC message format, the message patterns and the per-request metadata model; over a byte stream it SHOULD reuse stdio's newline-delimited framing
- C. A custom transport, but it must also send `Mcp-Method` headers
- D. A custom transport, as long as it uses sessions to replace per-request metadata

??? success "Answer"
    **B.** Custom transports are allowed. They MUST preserve the message format, patterns and metadata model,
    and SHOULD document connection setup, framing and cancellation. Byte-stream transports SHOULD reuse stdio
    framing. Headers are a Streamable HTTP feature, and sessions don't exist.
    [Transports](../domain-2/transports.md)

---

**28.** <span class="domain-tag">D4</span> A server receives a request whose token lacks the `files:write`
scope, and writing also requires `files:versions`. How should its `403` challenge be built?

- A. Challenge `files:write` now and `files:versions` on the next attempt, to keep each prompt small
- B. Include every scope listed in `scopes_supported`
- C. Include all scopes required for this operation (`files:write` and `files:versions`) in one challenge, taking scope hierarchies into account
- D. Return `401` so the client starts over

??? success "Answer"
    **C.** Servers SHOULD include **all** scopes needed for the operation in **one** challenge. Challenging one at
    a time forces repeated authorisation round trips. Servers MUST also account for scope hierarchies when
    deciding if a token suffices. `401` is for missing or invalid tokens, not insufficient scope.
    [Authorization with OAuth 2.1](../domain-4/authorization.md)

---

**29.** <span class="domain-tag">D5</span> A server offers an MCP App through the `io.modelcontextprotocol/ui`
extension. Its HTML loads a charting library from an external CDN. What must the server do?

- A. Nothing: MCP Apps can load any external content
- B. Host the library under `https://` resource URIs in `resources/list`
- C. Ask the user through form elicitation to allow the CDN
- D. Declare the external origins in `_meta.ui.csp`, and still return meaningful text for hosts without the extension

??? success "Answer"
    **D.** MCP Apps render in a sandboxed iframe, and external origins must be declared in `_meta.ui.csp`. Like any
    extension, the server should degrade gracefully, returning meaningful **text** to clients that don't support
    the UI extension.
    [Extensions](../domain-5/extensions.md)

---

**30.** <span class="domain-tag">D3</span> A stdio client has two active `subscriptions/listen` streams
(ids 7 and 9) and several in-flight requests. The server process crashes and the client restarts it. Which
statements are correct?

- A. The client correlates stdio notifications by `io.modelcontextprotocol/subscriptionId`, and after the restart it MUST re-send `subscriptions/listen`, because the server holds no subscription state
- B. The new process automatically resumes subscriptions 7 and 9
- C. The client resumes the subscriptions with `Last-Event-ID`
- D. Subscription notifications on stdio are correlated by the order they arrive

??? success "Answer"
    **A.** On stdio, all messages share stdout, so clients MUST correlate by `subscriptionId` (the listen
    request's id). A restarted server has no memory of subscriptions, so the client re-sends `subscriptions/listen`.
    Streams aren't resumable.
    [Subscriptions, Progress & Cancellation](../domain-3/subscriptions-progress-cancellation.md)

---

**31.** <span class="domain-tag">D4</span> **(Select TWO)** Which of these are listed as common mistakes
in OAuth scope design for MCP servers?

- A. Publishing every possible scope in `scopes_supported`
- B. Emitting precise `insufficient_scope` challenges for privileged operations
- C. Using wildcard or omnibus scopes such as `*` or `full-access`
- D. Accepting down-scoped tokens that carry a subset of the requested scopes
- E. Logging scope elevation events with correlation IDs

??? success "Answer"
    **A and C.** The scope minimisation guidance lists as mistakes: publishing all scopes in `scopes_supported`,
    wildcard or omnibus scopes, bundling unrelated privileges, returning the full catalogue in every challenge,
    silently changing scope meanings, and trusting token scopes without server-side checks. Precise challenges,
    down-scoping tolerance and elevation logging are all **recommended**.
    [Permissions & Consent](../domain-4/permissions-consent.md)

---

**32.** <span class="domain-tag">D1</span> A developer writes a minimal MCP client for a script. Its
requests declare `"io.modelcontextprotocol/clientCapabilities": {}`. Is that valid?

- A. No: clients must declare at least `elicitation`
- B. No: clients must copy the server's capabilities
- C. Yes, but it must also send `initialize` first
- D. Yes: a minimal client can declare no capabilities at all; the field is required, but it may be an empty object

??? success "Answer"
    **D.** Capabilities make features progressive: a minimal server can implement only tools, and a minimal
    client can declare nothing. The `clientCapabilities` key is still **required** on every request, so an
    empty object is how "no optional features" is expressed. There's no `initialize` in `2026-07-28`.
    [Core Concepts](../domain-1/core-concepts.md)

---

**33.** <span class="domain-tag">D2</span> A host wants to stop a stdio server gracefully. What is the
recommended sequence?

- A. Close the server's stdin, wait for it to exit, and if it doesn't within a reasonable time, terminate it (on POSIX, `SIGTERM` escalating to `SIGKILL`)
- B. Send a JSON-RPC `shutdown` request, then `exit`
- C. Send `notifications/cancelled` with no request id
- D. Send `SIGKILL` immediately; stdio servers hold no state

??? success "Answer"
    **A.** Closing stdin is the primary, and only portable, graceful-shutdown signal; servers SHOULD exit promptly
    on end-of-file. The client escalates only if the server doesn't exit. MCP has no shutdown request, and
    `notifications/cancelled` cancels a specific request.
    [Transports](../domain-2/transports.md)

---

**34.** <span class="domain-tag">D3</span> A stdio client sends `notifications/cancelled` for request 42.
The server had already sent the response for 42 a moment earlier, and it arrives at the client after the
cancellation was sent. How should each side behave?

- A. The server must send a second response confirming the cancellation
- B. The client must treat the response as an error
- C. The server MAY ignore the cancellation for a completed request, and the client SHOULD ignore the late response
- D. The server must return `-32600` for the invalid cancellation

??? success "Answer"
    **C.** Cancellation races are normal, and both sides MUST handle them gracefully. Servers may ignore
    cancellations for unknown or completed requests, clients ignore responses that arrive after they
    cancelled, and invalid cancellations are ignored rather than answered with errors, since notifications
    get no response.
    [Subscriptions, Progress & Cancellation](../domain-3/subscriptions-progress-cancellation.md)

---

**35.** <span class="domain-tag">D5</span> A host has approved a skill from a server using the Skills over
MCP extension. The server later adds a new supporting file to the skill. What must the host do?

- A. Load the new file, since the skill was already approved
- B. Treat approval as revoked, because persisted approval is bound to the full set of file URIs and SHA-256 digests; any change, addition or removal requires fresh approval
- C. Ignore the manifest; digests only apply to `SKILL.md`
- D. Ask the server to re-sign the skill with its `serverInfo`

??? success "Answer"
    **B.** Hosts MUST bind persisted approval to the full set of URIs and digests, so any changed, added or removed
    file revokes it. Digests prove consistency with the manifest, not trustworthiness, and skill content remains
    untrusted. This is the Skills equivalent of rug-pull protection.
    [Extensions](../domain-5/extensions.md)

---

**36.** <span class="domain-tag">D4</span> A stdio server needs a GitHub token. How should it obtain
credentials?

- A. Run the full MCP OAuth flow over stdin/stdout
- B. From the environment, such as environment variables or configuration passed when the client launches it; implementations using stdio SHOULD NOT follow the MCP authorization specification
- C. Ask the user for the token with form-mode elicitation
- D. Read the host's bearer token from the `_meta` of each request

??? success "Answer"
    **B.** The authorization framework is for HTTP. A stdio server is a subprocess the client launched: there's no
    endpoint to protect and no redirect target, so credentials come from the environment. Asking for a token in a
    form elicitation is forbidden.
    [Authorization with OAuth 2.1](../domain-4/authorization.md)

---

**37.** <span class="domain-tag">D3</span> For what purpose may a **server** send `notifications/cancelled`?

- A. To cancel a client's `tools/call` that is taking too long
- B. To tell the client an elicitation expired
- C. Never: only clients send notifications
- D. Only to end a `subscriptions/listen` stream, referencing the listen request's id

??? success "Answer"
    **D.** Server-sent `notifications/cancelled` MUST reference a `subscriptions/listen` request and MUST NOT be used for
    anything else. Servers do send notifications (progress, list changes); they just never send requests.
    [Subscriptions, Progress & Cancellation](../domain-3/subscriptions-progress-cancellation.md)

---

**38.** <span class="domain-tag">D1</span> What is true of a specification revision in the **Current** state?

- A. It is frozen and can never change
- B. It is still in progress and not ready for use
- C. It is ready for use and may still receive backward-compatible changes
- D. It is deprecated and scheduled for removal

??? success "Answer"
    **C.** Revisions are **Draft** (in progress), **Current** (ready, may still get compatible changes) or **Final**
    (complete, frozen). Deprecation applies to individual features, not revisions.
    [Interoperability & Value](../domain-1/interoperability.md)

---

**39.** <span class="domain-tag">D5</span> A company is in its "scale" phase: many teams, many servers.
Which set of controls does the course's phased adoption path associate with that phase?

- A. Individual developers using local stdio servers with read-only tools
- B. No controls until incidents occur
- C. An internal registry or subregistry, Enterprise-Managed Authorization, OpenTelemetry tracing and scope minimisation
- D. Only an approved server list, with production credentials banned

??? success "Answer"
    **C.** The path goes Explore (local, read-only, approved list) → Pilot (OAuth via the corporate IdP, audit logging,
    confirmation policy) → **Scale** (internal registry, Enterprise-Managed Authorization, tracing, least-privilege
    scopes) → Operate (conformance testing, version pinning, deprecation tracking).
    [Roles, Responsibilities & Adoption](../domain-5/roles-adoption.md)

---

**40.** <span class="domain-tag">D4</span> An audit team wants every tool call logged with full arguments
and results "to be safe". What does the course's guidance on audit records say?

- A. Log everything verbatim; completeness beats privacy
- B. Record enough to investigate (actor from the validated token, tool, definition hash, approval evidence, result status) but redact secrets and PII, make argument and result capture policy-controlled, and restrict access and retention
- C. Log only `clientInfo`, which identifies the actor
- D. Don't log arguments at all; results are enough

??? success "Answer"
    **B.** Logs MUST NOT contain credentials, secrets, PII or details that aid attacks. Arguments and results are the
    most useful and the most dangerous data to store, so they are minimised and redacted (the OpenTelemetry
    conventions make them opt-in). Identity comes from the token, not `clientInfo`.
    [Auditability & Observability](../domain-4/auditability.md)

---

**41.** <span class="domain-tag">D2</span> A server's `server/discover` result includes
`"instructions": "Always call search_tickets before create_ticket."` Which statement is correct?

- A. It is optional natural-language guidance for the model, written by the server, and should be trusted only as much as the server itself
- B. It is a security policy the client must enforce
- C. It is only for human operators and must never be shown to the model
- D. It overrides the host's system prompt

??? success "Answer"
    **A.** `instructions` is server-authored text meant for LLMs, which hosts may add to context. Like tool
    descriptions, it enters the model's context, so a malicious server could use it for injection. It is neither a
    policy the protocol enforces nor something that overrides host instructions.
    [Versioning & Discovery](../domain-2/versioning-discovery.md)

---

**42.** <span class="domain-tag">D3</span> A server wraps a private wiki. Resource content is fetched by the
server with its own credentials. Which URI scheme guidance applies?

- A. Use `https://` URIs, because the content is ultimately on the web
- B. Use `https://` only when the client can fetch the resource directly itself; otherwise use another scheme (such as a custom one), even if the server downloads it from the internet
- C. Use `file://`, the only scheme allowed for text
- D. Any scheme is fine, since URIs are opaque

??? success "Answer"
    **B.** An `https://` URI tells the client it can fetch the resource itself. If access needs the server's
    credentials or processing, the client would get nothing or the wrong content, so a scheme that routes through
    `resources/read` is correct. Custom schemes MUST comply with RFC 3986.
    [Resources & Prompts](../domain-3/resources-prompts.md)

---

**43.** <span class="domain-tag">D1</span> Which party sends JSON-RPC **requests** in `2026-07-28` MCP?

- A. Only clients
- B. Both clients and servers
- C. Only servers
- D. Only hosts, bypassing clients

??? success "Answer"
    **A.** Servers MUST NOT initiate JSON-RPC requests, and clients don't send responses. When a server needs input,
    it returns an `InputRequiredResult`. The host acts through its clients; the client is the protocol component
    that sends requests.
    [Core Concepts](../domain-1/core-concepts.md)

---

**44.** <span class="domain-tag">D5</span> A client calls `tasks/cancel` on a running task. The next
`tasks/get` shows status `completed` with a result. Is that possible?

- A. No: after `tasks/cancel` the status must be `cancelled`
- B. No: `completed` tasks can revert to `working`
- C. No: `tasks/cancel` deletes the task
- D. Yes: cancellation is cooperative; it is acknowledged, but the task may still reach a different terminal state

??? success "Answer"
    **D.** `tasks/cancel` is **cooperative**. The server acknowledges it, but work may already have finished or may not
    be interruptible. Terminal states (`completed`, `failed`, `cancelled`) never change once reached.
    [Extensions](../domain-5/extensions.md)

---

**45.** <span class="domain-tag">D4</span> A native client uses a Client ID Metadata Document with a
`localhost` redirect URI. What limitation of CIMD does this expose, and what should the authorization
server do about it?

- A. CIMD can't be used by native apps; the AS must reject it
- B. CIMD proves control of a domain but not which local process listens on a `localhost` port; the AS should warn about localhost-only redirect URIs and clearly display the redirect URI hostname
- C. The `client_id` must be `localhost`
- D. None: CIMD fully authenticates the local process

??? success "Answer"
    **B.** An attacker can present a legitimate client's metadata URL and a `localhost` redirect of their own, and the
    user sees the legitimate client's name. The AS should add warnings for localhost-only redirects, show the redirect
    hostname clearly, and may require extra attestation.
    [Attacks & Mitigations](../domain-4/attacks-mitigations.md)

---

**46.** <span class="domain-tag">D3</span> A client caches a `resources/read` result marked
`cacheScope: "private"`. The user signs out and a different user signs in on the same shared machine, with
a different access token. The URI is the same. May the client serve the cached result?

- A. No: `private` results may be reused only within the same authorisation context, so a different access token requires a different cache
- B. Yes, because the URI and method match the cache key
- C. Yes, if `ttlMs` hasn't expired
- D. Only if the server also sent `listChanged`

??? success "Answer"
    **A.** `private` means the data must not be shared between callers. Caches MUST NOT be shared across authorisation
    contexts: a different token means a different cache. A matching key and a fresh TTL don't override that.
    [Caching, Pagination & Completion](../domain-3/utilities.md)

---

**47.** <span class="domain-tag">D2</span> A host renders server-provided tool icons. Which handling follows
the specification's guidance?

- A. Fetch any `src` URL, including `file:` and `javascript:`, with the user's cookies for authenticated CDNs
- B. Accept only `https:` or `data:` URIs, fetch without credentials, verify same origin as the server, and beware SVGs with embedded scripts
- C. Trust the declared `mimeType` and render SVG inline
- D. Icons are part of the protocol, so they are trusted once the server is connected

??? success "Answer"
    **B.** Icon metadata and bytes are **untrusted**. Only `https:` and `data:` are acceptable. Fetch without cookies or
    `Authorization` headers, check the origin, sniff content type from bytes rather than trusting `mimeType`, and guard
    against script-carrying SVGs and oversized images.
    [JSON-RPC Messages & `_meta`](../domain-2/messages.md)

---

**48.** <span class="domain-tag">D5</span> Which of these is an **anti-pattern** for designing an MCP server's tools?

- A. Focused tools with precise schemas and `additionalProperties: false`
- B. One mega-tool `execute(action, params)` covering every operation
- C. Returning tools in a deterministic order
- D. Returning an actionable `isError: true` message for invalid argument values

??? success "Answer"
    **B.** A single generic tool hides capabilities from the model, weakens validation and forces coarse permissions: a
    host can't approve "read" separately from "delete". The other options are all recommended practices.
    [Operational Use Cases](../domain-5/operational-use-cases.md)

---

**49.** <span class="domain-tag">D4</span> In Enterprise-Managed Authorization, at which step does the
organisation's policy (group membership, roles, conditional access) decide whether an employee may reach an
MCP server?

- A. When the MCP server calls `server/discover`
- B. When the user approves a consent screen on the MCP authorization server
- C. When the enterprise IdP exchanges the employee's ID token for an ID-JAG
- D. When the model selects a tool

??? success "Answer"
    **C.** The client exchanges the SSO identity at the IdP for an **ID-JAG**, and **the IdP evaluates policy** there.
    The client then exchanges the ID-JAG at the MCP authorization server for an access token without redirecting the
    user. Revocation is central too, taking effect across all clients.
    [Authorization with OAuth 2.1](../domain-4/authorization.md)

---

**50.** <span class="domain-tag">D3</span> A deprecated Roots-based filesystem server receives roots
`file:///home/ada/project`. A malicious version of the server reads `~/.ssh/id_rsa` anyway. Which statement
is correct?

- A. The protocol blocks this, because servers MUST enforce roots
- B. The client must revoke the server's `roots` capability
- C. Roots are enforced by `cacheScope`
- D. Roots are advisory coordination, not a security boundary; real restriction must come from the operating system, such as permissions, sandboxes or containers

??? success "Answer"
    **D.** Servers SHOULD respect roots, but the spec deliberately doesn't say MUST enforce, because servers run code the
    client can't control. Roots prevent accidents by well-behaved servers, not attacks by malicious ones. Sandboxing is
    the real control.
    [Deprecated Features](../domain-3/deprecated-features.md)

---

**51.** <span class="domain-tag">D1</span> A database MCP server wants to give the host the schema of every
table, so the application can attach it as context whenever the user asks about data. Which primitive fits best?

- A. A resource (or resource template for per-table schemas)
- B. A tool, so the model calls it on every turn
- C. A prompt, so the user must select it each time
- D. An elicitation, so the user types the schema

??? success "Answer"
    **A.** Resources are **application-controlled** context identified by URIs, which is exactly schema data the host
    attaches. A tool fits actions the model decides to take, such as running a query, and a prompt fits a workflow the
    user starts.
    [Core Concepts](../domain-1/core-concepts.md)

---

**52.** <span class="domain-tag">D5</span> An internal team runs 30 identical stateless MCP server instances
behind a plain round-robin load balancer. A tool call needs user input via MRTR, and the retry lands on a
different instance. What happens?

- A. The retry fails; MRTR requires sticky sessions
- B. The retry works, because the retry carries everything needed (arguments, `inputResponses`, `requestState`), so any instance can complete it, provided all instances can verify the sealed state (for example with a shared key)
- C. The load balancer must forward the retry to the original instance using `Mcp-Session-Id`
- D. The client must call `server/discover` again first

??? success "Answer"
    **B.** Statelessness is why remote servers can scale like ordinary web services. Each retry is independent, and
    `requestState` carries the server's context. The operational catch is that every instance must be able to verify
    state another instance sealed. Sessions no longer exist.
    [Operational Use Cases](../domain-5/operational-use-cases.md)

---

**53.** <span class="domain-tag">D4</span> Where must an MCP client put the OAuth access token when calling
a Streamable HTTP server?

- A. In the `Authorization: Bearer` header of every HTTP request to the MCP server, never in the query string
- B. In the query string of the MCP endpoint URL, for compatibility with proxies
- C. In `params._meta` of the first request only
- D. In the `Mcp-Name` header

??? success "Answer"
    **A.** Tokens go in the `Authorization` header on **every** request. The protocol is stateless, so there is no
    "authenticated connection" to rely on. Tokens MUST NOT go in URI query strings, where they leak into logs, history
    and referrers.
    [Authorization with OAuth 2.1](../domain-4/authorization.md)

---

**54.** <span class="domain-tag">D3</span> A server uses deprecated sampling and sends
`modelPreferences.hints: [{"name": "claude-3-sonnet"}]` with `intelligencePriority: 0.8`. The client only has
access to Gemini models. What may it do?

- A. It must return an error, because the hinted model isn't available
- B. It must use the cheapest model available
- C. It may map the hint to a comparable model from another provider; hints are advisory and the client makes the final choice
- D. It must forward the request to the server's Anthropic account

??? success "Answer"
    **C.** Hints are substrings matched flexibly, and clients MAY map them to equivalent models from other providers.
    Priorities for cost, speed and intelligence guide the choice, but the **client** decides. The client does have to
    respect `maxTokens`.
    [Deprecated Features](../domain-3/deprecated-features.md)

---

**55.** <span class="domain-tag">D2</span> A prompt definition declares its arguments. Which format does
the specification use?

- A. A full JSON Schema 2020-12 object, like a tool's `inputSchema`
- B. An RFC 6570 URI template
- C. A restricted flat JSON Schema, like elicitation's `requestedSchema`
- D. A simple list of `{ name, description?, required? }` entries, not JSON Schema

??? success "Answer"
    **D.** Prompt arguments are a plain list of names with optional descriptions and a `required` flag. Tools use full
    JSON Schema, elicitation uses a restricted flat schema, and resource templates use RFC 6570.
    [Schemas & Structured Data](../domain-2/schemas.md)

---

**56.** <span class="domain-tag">D5</span> How is the official MCP Registry primarily meant to be consumed?

- A. Directly by every host application on each user request, as a live source of truth
- B. By downstream aggregators and subregistries that scrape its unauthenticated read-only API infrequently (around hourly), store the data themselves and add curation; the registry has no uptime or durability guarantees
- C. Only through the MCP Inspector
- D. By package registries such as npm, which mirror it

??? success "Answer"
    **B.** The registry keeps metadata deliberately unopinionated. Aggregators (marketplaces, subregistries) add
    curation, ratings and security scanning, and hosts consume those, not the official registry directly.
    [Ecosystem & Portability](../domain-5/ecosystem-portability.md)

---

**57.** <span class="domain-tag">D4</span> A security reviewer is told: "A client-side XSS in our host
stole the token for our local stdio proxy service, and then the attacker spawned arbitrary commands as MCP
servers." Which mitigations address the damage stage of this chain?

- A. Validate `iss` and use PKCE
- B. Set `cacheScope: "private"` on all results
- C. Disable `server/discover`
- D. Sandbox processes the proxy spawns, restrict filesystem access, log all stdio spawning, and require extra authorisation for dangerous commands

??? success "Answer"
    **D.** The stdio transport itself isn't vulnerable. The risk is a proxy that spawns processes on request. Prevent
    the enabling bug (URL validation, CSP, sanitisation), and limit the damage by sandboxing and logging spawns and
    gating dangerous commands. OAuth checks and cache scope don't affect process spawning.
    [Attacks & Mitigations](../domain-4/attacks-mitigations.md)

---

**58.** <span class="domain-tag">D3</span> A resource the client subscribed to via `resourceSubscriptions`
changes. What does `notifications/resources/updated` contain?

- A. The new contents of the resource
- B. A diff against the previous version
- C. The resource's `uri` (plus the subscription id in `_meta`); the client decides whether to call `resources/read`
- D. Nothing; the client must poll to find out which resource changed

??? success "Answer"
    **C.** The notification carries only the **URI**, tagged with `io.modelcontextprotocol/subscriptionId`. The client
    chooses whether to re-read. Carrying contents in notifications would push potentially large or unwanted data
    through the listen stream.
    [Resources & Prompts](../domain-3/resources-prompts.md)

---

**59.** <span class="domain-tag">D1</span> A host is connected to five servers. Which component merges their
tools, resources and prompts into what the model sees?

- A. Each server, which advertises the others' tools
- B. The host, which manages context aggregation across its clients
- C. A shared client that talks to all five servers
- D. The model provider's API

??? success "Answer"
    **B.** Context aggregation is a **host** responsibility. Each client talks to exactly one server, and servers
    don't know about each other. The model provider's API only receives what the host sends.
    [Core Concepts](../domain-1/core-concepts.md)

---

**60.** <span class="domain-tag">D5</span> A nightly batch agent calls an MCP server with no human present.
How should it declare capabilities, and how does human oversight work?

- A. It should not declare `elicitation`, so servers won't depend on a user; oversight moves earlier (policy, scoped credentials, approving the automation) and later (audit review)
- B. It should declare `elicitation` and auto-accept every request
- C. It should declare every capability to maximise compatibility
- D. Human oversight isn't needed for automation

??? success "Answer"
    **A.** Capabilities are per request, so a client with no user present simply doesn't declare `elicitation`, and
    servers MUST NOT rely on it. Auto-accepting elicitations defeats their purpose. Oversight still exists; it shifts to
    configuration time and audit time.
    [Operational Use Cases](../domain-5/operational-use-cases.md)

---

## Answer key

| Q | Answer | Domain | | Q | Answer | Domain | | Q | Answer | Domain |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | C | D2 | | 21 | D | D1 | | 41 | A | D2 |
| 2 | D | D3 | | 22 | B | D3 | | 42 | B | D3 |
| 3 | A | D1 | | 23 | A | D5 | | 43 | A | D1 |
| 4 | A | D4 | | 24 | C | D4 | | 44 | D | D5 |
| 5 | B | D3 | | 25 | D | D2 | | 45 | B | D4 |
| 6 | D | D5 | | 26 | A | D3 | | 46 | A | D3 |
| 7 | C | D4 | | 27 | B | D1 | | 47 | B | D2 |
| 8 | C | D3 | | 28 | C | D4 | | 48 | B | D5 |
| 9 | B | D1 | | 29 | D | D5 | | 49 | C | D4 |
| 10 | C | D2 | | 30 | A | D3 | | 50 | D | D3 |
| 11 | D | D4 | | 31 | A, C | D4 | | 51 | A | D1 |
| 12 | A | D3 | | 32 | D | D1 | | 52 | B | D5 |
| 13 | A | D5 | | 33 | A | D2 | | 53 | A | D4 |
| 14 | C | D1 | | 34 | C | D3 | | 54 | C | D3 |
| 15 | B | D3 | | 35 | B | D5 | | 55 | D | D2 |
| 16 | C | D4 | | 36 | B | D4 | | 56 | B | D5 |
| 17 | D | D2 | | 37 | D | D3 | | 57 | D | D4 |
| 18 | A, C | D5 | | 38 | C | D1 | | 58 | C | D3 |
| 19 | D | D3 | | 39 | C | D5 | | 59 | B | D1 |
| 20 | A | D4 | | 40 | B | D4 | | 60 | A | D5 |

## Score by domain

| Domain | Questions | 80% threshold | If you're below it, revisit |
| --- | --- | --- | --- |
| **D1** MCP Fundamentals | 3, 9, 14, 21, 27, 32, 38, 43, 51, 59 (10) | 8 | [Core Concepts](../domain-1/core-concepts.md), [Interoperability & Value](../domain-1/interoperability.md) |
| **D2** Architecture & Components | 1, 10, 17, 25, 33, 41, 47, 55 (8) | 7 | [Schemas](../domain-2/schemas.md), [Messages](../domain-2/messages.md), [Transports](../domain-2/transports.md), [Versioning & Discovery](../domain-2/versioning-discovery.md) |
| **D3** Interactions & Execution | 2, 5, 8, 12, 15, 19, 22, 26, 30, 34, 37, 42, 46, 50, 54, 58 (16) | 13 | [MRTR & Elicitation](../domain-3/mrtr-elicitation.md), [Resources & Prompts](../domain-3/resources-prompts.md), [Error Handling](../domain-3/error-handling.md), [Deprecated Features](../domain-3/deprecated-features.md) |
| **D4** Security & Governance | 4, 7, 11, 16, 20, 24, 28, 31, 36, 40, 45, 49, 53, 57 (14) | 12 | [Attacks & Mitigations](../domain-4/attacks-mitigations.md), [Authorization](../domain-4/authorization.md), [Auditability](../domain-4/auditability.md) |
| **D5** Use Cases & Ecosystem | 6, 13, 18, 23, 29, 35, 39, 44, 48, 52, 56, 60 (12) | 10 | [Ecosystem & Portability](../domain-5/ecosystem-portability.md), [Extensions](../domain-5/extensions.md), [Operational Use Cases](../domain-5/operational-use-cases.md) |

!!! tip "After this exam"
    If you scored 85% or more, move on to the [flashcards](flashcards.md) and book the exam. If a single domain
    is below its threshold, rework that domain's chapters and quiz. Don't retake this mock straight away.
