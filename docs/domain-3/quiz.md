# Domain 3 Quiz — Interactions & Execution

20 exam-style questions. This is the heaviest domain. Target: **17/20**.

---

**1.** A server processing `tools/call` needs the user to confirm a purchase. In `2026-07-28`, what does the server do?

- A. Sends an `elicitation/create` JSON-RPC request to the client and waits
- B. Returns a result with `resultType: "input_required"` containing an elicitation in `inputRequests`
- C. Returns JSON-RPC error `-32042`
- D. Sends `notifications/elicitation/complete`

??? success "Answer"
    **B.** Servers MUST NOT initiate requests. They return an `InputRequiredResult`, and the client retries. A is
    legacy; `-32042` and the completion notification were removed.

---

**2.** When retrying a request after an `InputRequiredResult`, which is **REQUIRED** of the client?

- A. Reuse the original JSON-RPC id so the server can correlate
- B. Use a new id, include `inputResponses`, and echo `requestState` exactly if one was provided
- C. Decode `requestState` and update the fields that changed
- D. Send the retry to a different endpoint

??? success "Answer"
    **B.** The retry MUST use a different id, and the client MUST echo `requestState` byte-for-byte without inspecting
    or modifying it.

---

**3.** Which client requests may a server answer with `InputRequiredResult`?

- A. Any request
- B. `tools/call`, `resources/read` and `prompts/get` only
- C. `tools/list` and `server/discover` only
- D. `subscriptions/listen` only

??? success "Answer"
    **B.** Servers MUST NOT send `InputRequiredResult` on any other request.

---

**4.** A server stores `{"amount": 3000, "approved": false}` as plain base64 JSON in `requestState`, and uses it to decide whether to charge. What is the flaw?

- A. `requestState` must be JSON
- B. `requestState` is attacker-controlled; because it influences business logic, its integrity MUST be protected (HMAC/AEAD) and failed verification rejected
- C. `requestState` may only contain the tool name
- D. There is no flaw because the client must echo it exactly

??? success "Answer"
    **B.** The spec requires servers to treat `requestState` as attacker-controlled. A malicious client can edit it. The
    client-side "echo exactly" rule does not protect the server.

---

**5.** Which set of measures does the spec say servers SHOULD bind into `requestState` to limit replay?

- A. The client IP, User-Agent and TLS version
- B. The authenticated principal, a short expiry, and an identifier of the originating request
- C. The model name and temperature
- D. The subscription id

??? success "Answer"
    **B.** These stop cross-user, stale and cross-request reuse. For operations that must be single-use, the server must
    additionally enforce that server-side.

---

**6.** A server needs a user's third-party API key to call an external service. Which elicitation mode must it use?

- A. Form mode with a password field
- B. URL mode, so the secret never passes through the MCP client
- C. Either mode, at the server's discretion
- D. Sampling

??? success "Answer"
    **B.** Servers MUST NOT use form mode for passwords, API keys, tokens or payment credentials, and MUST use URL mode for them.

---

**7.** A client declares `"elicitation": {}` in its capabilities. What does this mean?

- A. The client supports both form and URL modes
- B. The client supports form mode only
- C. The client supports no elicitation
- D. The declaration is invalid

??? success "Answer"
    **B.** For backward compatibility, an empty object is equivalent to `{ "form": {} }`.

---

**8.** A user opens a URL-mode elicitation and the client returns `{"action": "accept"}`. What does `accept` indicate?

- A. The out-of-band interaction has completed successfully
- B. The user consented to open the URL; the server determines completion when the request is retried
- C. The user entered their credentials into the client
- D. The server must now send a completion notification

??? success "Answer"
    **B.** The client isn't told the outcome of the out-of-band interaction. The server works it out on retry from
    `requestState` or its own state.

---

**9.** A user closes an elicitation dialog by pressing Escape without choosing. Which action should the client return?

- A. `decline`
- B. `cancel`
- C. `accept` with empty content
- D. A JSON-RPC error

??? success "Answer"
    **B.** `cancel` means dismissed without an explicit choice. `decline` is an explicit refusal.

---

**10.** Which is a **MUST** for clients handling URL-mode elicitation?

- A. Pre-fetch the URL to show a preview
- B. Show the full URL and obtain explicit consent before opening it, and never auto-open or pre-fetch it
- C. Open the URL in an embedded web view the LLM can read
- D. Render the URL as a clickable link in the chat

??? success "Answer"
    **B.** Clients MUST NOT pre-fetch, MUST NOT open without consent, MUST show the full URL, and MUST open it in a way the
    client and LLM can't inspect.

---

**11.** A tool is called with `departure_date: "2020-01-01"` and the date must be in the future. How should the server report this?

- A. JSON-RPC error `-32602`
- B. JSON-RPC error `-32603`
- C. A result with `isError: true` and an actionable message
- D. HTTP 422

??? success "Answer"
    **C.** Input validation errors are tool execution errors, so the model can self-correct. Protocol errors are for
    unknown tools or malformed requests.

---

**12.** In `2026-07-28`, what does a server return for `resources/read` on a URI that doesn't exist?

- A. A result with an empty `contents` array
- B. JSON-RPC error `-32002`
- C. JSON-RPC error `-32602`
- D. JSON-RPC error `-32601`

??? success "Answer"
    **C.** Not-found changed from `-32002` to `-32602`. Servers MUST NOT return empty `contents` for a missing resource.
    Clients SHOULD still accept `-32002` from older servers.

---

**13.** Which error code does a server return when a request needs a client capability the client did not declare?

- A. `-32020`
- B. `-32021`
- C. `-32022`
- D. `-32600`

??? success "Answer"
    **B.** `MissingRequiredClientCapability` (`-32021`), with `data.requiredCapabilities`. `-32020` is HeaderMismatch;
    `-32022` is UnsupportedProtocolVersion.

---

**14.** A client opens `subscriptions/listen` with JSON-RPC id `7`. What value appears in `io.modelcontextprotocol/subscriptionId` on notifications from that stream?

- A. A random UUID chosen by the server
- B. `7`
- C. The session id
- D. The tool name

??? success "Answer"
    **B.** The subscription id is the JSON-RPC id of the `subscriptions/listen` request.

---

**15.** Which notification is **NOT** delivered on a `subscriptions/listen` stream?

- A. `notifications/tools/list_changed`
- B. `notifications/resources/updated`
- C. `notifications/progress`
- D. `notifications/subscriptions/acknowledged`

??? success "Answer"
    **C.** Progress (and log) notifications are request-scoped and travel on the response stream of the request they relate to.

---

**16.** How does a client cancel an in-flight request over stdio?

- A. Close stdin
- B. Send `notifications/cancelled` with the request id
- C. Send HTTP DELETE
- D. Stop reading stdout

??? success "Answer"
    **B.** On stdio the client MUST send `notifications/cancelled`. Closing stdin is how a client shuts the server down.

---

**17.** A server returns `tools/list` with `ttlMs: 300000` and `cacheScope: "private"`. Which is correct?

- A. Any gateway may serve this list to any user for 5 minutes
- B. The client may reuse it for 5 minutes within the same authorization context only; a different token needs a different cache
- C. The client must poll every 5 minutes
- D. The list must not be cached

??? success "Answer"
    **B.** `private` restricts reuse to the same authorization context. TTL is a freshness hint, not a polling interval.

---

**18.** A paginated list response contains `"nextCursor": ""`. What should the client do?

- A. Treat it as the end of results
- B. Request the next page, passing the empty string as the cursor
- C. Treat it as an error
- D. Decode the cursor

??? success "Answer"
    **B.** Clients may make no determination from a cursor's value other than whether one was provided. An empty string is a
    valid cursor. Only an absent `nextCursor` means the end.

---

**19.** Which statement about Sampling in `2026-07-28` is correct?

- A. It has been removed from the specification
- B. It is deprecated: still functional, delivered via MRTR, but new implementations should not adopt it and should call LLM provider APIs directly
- C. It moved to the Tasks extension
- D. It replaced elicitation

??? success "Answer"
    **B.** Sampling, Roots and Logging are deprecated, not removed. Elicitation remains active.

---

**20.** A server wants to keep a shopping cart across multiple tool calls. Which approach fits `2026-07-28`?

- A. Store the cart against the connection or `Mcp-Session-Id`
- B. Return an explicit handle from a creation tool, accept it as an argument on later calls, and validate the caller's authorization against it every call
- C. Keep the cart in `requestState` on every tool call
- D. Use `ping` to keep the cart alive

??? success "Answer"
    **B.** MCP has no protocol-level session. State uses explicit handles, which for authenticated servers are names, not
    capabilities, so authorization is checked on each call.

---

| Score | Verdict |
| --- | --- |
| 18–20 | Ready for Domain 4. |
| 14–17 | Re-read [MRTR & Elicitation](mrtr-elicitation.md) and [Error Handling](error-handling.md). |
| < 14 | Rework the domain; it is a quarter of the exam. |
