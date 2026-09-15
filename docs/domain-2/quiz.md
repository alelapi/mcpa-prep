# Domain 2 Quiz — Architecture & Components

16 exam-style questions. Target: **14/16**.

---

**1.** Which participant is responsible for enforcing consent requirements before a tool is invoked?

- A. The server
- B. The client
- C. The host
- D. The LLM

??? success "Answer"
    **C.** The host "enforces security policies and consent requirements" and "handles user authorization
    decisions". It is the only participant with the user, the model and all servers in view.

---

**2.** A raw message reads `{"jsonrpc":"2.0","method":"notifications/progress","params":{...}}`. What is it?

- A. A request from the client
- B. A notification; the receiver must not respond
- C. A result response
- D. A malformed message because it lacks an id

??? success "Answer"
    **B.** A message with `method` and no `id` is a notification. Notifications MUST NOT include an id, and
    the receiver MUST NOT respond.

---

**3.** Which statement about request IDs in MCP is correct?

- A. They may be `null`, as in base JSON-RPC
- B. They must be globally unique forever
- C. They must be a string or integer, not null, and not match any other in-flight request from the sender
- D. They must be UUIDs

??? success "Answer"
    **C.** MCP forbids null ids (unlike base JSON-RPC) and requires uniqueness only among requests still
    awaiting a response.

---

**4.** A client receives a result with `resultType: "task"` but has not negotiated any extension defining that value. What must it do?

- A. Treat it as `"complete"`
- B. Treat the result as invalid
- C. Retry the request with a new id
- D. Ignore the `resultType` field

??? success "Answer"
    **B.** An unrecognized `resultType` MUST be considered invalid. Only an **absent** `resultType` (from a
    legacy server) is treated as `"complete"`.

---

**5.** Which `_meta` fields are **required** on every client request?

- A. `clientInfo` and `logLevel`
- B. `protocolVersion` and `clientCapabilities`
- C. `progressToken` and `traceparent`
- D. `serverInfo` and `subscriptionId`

??? success "Answer"
    **B.** `io.modelcontextprotocol/protocolVersion` and `io.modelcontextprotocol/clientCapabilities` are
    required. `clientInfo` is SHOULD. A request missing a required field is rejected with `-32602` (HTTP 400).

---

**6.** Which `_meta` prefix is **NOT** reserved for MCP?

- A. `io.modelcontextprotocol/`
- B. `dev.mcp/`
- C. `com.example.mcp/`
- D. `org.modelcontextprotocol.api/`

??? success "Answer"
    **C.** A prefix is reserved when its **second label** is `modelcontextprotocol` or `mcp`. In
    `com.example.mcp/` the second label is `example`.

---

**7.** A security team wants to grant elevated access to requests whose `clientInfo.name` is `"CorpApprovedAgent"`. What is wrong with this?

- A. Nothing, clientInfo is verified by the protocol
- B. clientInfo is self-reported and unverified, and must not be relied on for security decisions
- C. clientInfo is only available over stdio
- D. clientInfo is deprecated

??? success "Answer"
    **B.** `clientInfo` and `serverInfo` are for display, logging and debugging. Implementations SHOULD NOT
    rely on them for security decisions. Identity for security comes from authorization.

---

**8.** A tool takes no arguments. Which `inputSchema` does the specification recommend?

- A. `null`
- B. `{}`
- C. `{ "type": "object", "additionalProperties": false }`
- D. `{ "type": "array" }`

??? success "Answer"
    **C.** `inputSchema` must be a valid JSON Schema object, not null. The recommended form accepts only an
    empty object; `{ "type": "object" }` is also valid but accepts any properties.

---

**9.** A tool defines an `outputSchema`. Which is required?

- A. The server MUST return `structuredContent` conforming to the schema
- B. The client MUST reject any `content` array
- C. The server MUST omit text content
- D. The output schema MUST use draft-07

??? success "Answer"
    **A.** With an output schema, servers MUST return conforming structured results; clients SHOULD validate
    them. For compatibility the server SHOULD also return the JSON serialized in a text block.

---

**10.** A tool has no annotations. How should a careful client interpret its behavior?

- A. Read-only and closed-world
- B. Not read-only, potentially destructive, not idempotent, open-world
- C. Idempotent and safe to retry
- D. Annotations are required, so the tool is invalid

??? success "Answer"
    **B.** The defaults are conservative: `readOnlyHint` false, `destructiveHint` true, `idempotentHint` false,
    `openWorldHint` true. And annotations are untrusted anyway unless the server is trusted.

---

**11.** A server launched over stdio prints `Server started!` using `print()`. What happens?

- A. Nothing; stdout is ignored
- B. The client receives a non-JSON line on stdout and fails to parse it; logs belong on stderr
- C. The client treats it as a notification
- D. The server is automatically restarted

??? success "Answer"
    **B.** The server MUST NOT write anything to stdout that is not a valid MCP message. Logging goes to stderr.

---

**12.** On Streamable HTTP, how does a client cancel an in-flight request?

- A. POST a `notifications/cancelled` message
- B. Send HTTP DELETE with the session id
- C. Close the SSE response stream for that request
- D. Send `ping` with a cancel flag

??? success "Answer"
    **C.** Closing the response stream is the cancellation signal on Streamable HTTP. `notifications/cancelled`
    is used on stdio. Sessions and ping no longer exist.

---

**13.** A Streamable HTTP request has header `Mcp-Name: read_report` but the body says `"name": "delete_reports"`. What must a server that processes the body do?

- A. Execute the body's tool, since the body is the source of truth
- B. Execute the header's tool
- C. Reject with `400 Bad Request` and `-32020` HeaderMismatch
- D. Ignore the header

??? success "Answer"
    **C.** Servers MUST reject header/body mismatches. Otherwise a gateway authorizing by header and a server
    executing by body would act on different things.

---

**14.** A local MCP server uses Streamable HTTP on the user's laptop. Which measures does the spec require or recommend against DNS rebinding?

- A. Enable CORS for all origins
- B. Validate the `Origin` header (403 if invalid) and bind to 127.0.0.1 rather than 0.0.0.0
- C. Use `Mcp-Session-Id`
- D. Require HTTP GET for all requests

??? success "Answer"
    **B.** Servers MUST validate Origin and SHOULD bind locally to loopback, and SHOULD authenticate connections.

---

**15.** Which is TRUE about `server/discover`?

- A. Clients must call it before any other request
- B. Servers must implement it, clients may call it, and the result carries caching hints
- C. It establishes a session
- D. It replaces `tools/list`

??? success "Answer"
    **B.** Servers MUST implement it. Clients MAY call it up front, or send any request directly and handle
    version errors. It establishes no state and its result is cacheable.

---

**16.** A dual-era client on stdio sends `server/discover` and receives error `-32601` (Method not found). What should it conclude?

- A. The server is modern but misconfigured; retry
- B. The server is legacy; fall back to the `initialize` handshake
- C. The server supports no versions
- D. The client must switch to Streamable HTTP

??? success "Answer"
    **B.** On stdio, a `DiscoverResult` or a recognized modern error means modern. Any other error, or a timeout,
    means legacy. The fallback must not be keyed to one specific code, but `-32601` is a typical legacy response.

---

| Score | Verdict |
| --- | --- |
| 15–16 | Strong. |
| 12–14 | Re-read [Messages](messages.md) and [Transports](transports.md). |
| < 12 | Rework the domain. Domain 3 depends on it. |
