# Domain 1 Quiz — MCP Fundamentals

15 exam-style questions. Answer before expanding. Target: **13/15**.

!!! note
    Original questions written against the `2026-07-28` specification. They are not real exam items.

---

**1.** With 8 AI applications and 40 external systems, how many integrations does a shared protocol like MCP require, compared to point-to-point integration?

- A. 48 instead of 320
- B. 320 instead of 48
- C. 40 instead of 8
- D. 8 instead of 320

??? success "Answer"
    **A.** Point to point is up to M×N = 320. With MCP each application implements the protocol once (8)
    and each system has one server (40), giving M+N = 48.

---

**2.** Which existing protocol does the MCP specification explicitly name as its inspiration?

- A. gRPC
- B. The Language Server Protocol
- C. GraphQL
- D. OpenAPI

??? success "Answer"
    **B.** The spec says MCP "takes some inspiration from the Language Server Protocol", which
    standardized how editors integrate language support, much as MCP standardizes how AI applications
    integrate context and tools.

---

**3.** A user types `/summarize-meetings` in their AI assistant to start a workflow defined by an MCP server. Which primitive is this?

- A. Tool
- B. Resource
- C. Prompt
- D. Elicitation

??? success "Answer"
    **C.** Prompts are **user-controlled**, and slash commands are the canonical example. Tools are
    model-controlled; resources are application-controlled.

---

**4.** Which statement about hosts and clients is correct?

- A. A host and a client are the same thing
- B. A client can connect to many servers to reduce overhead
- C. A host creates one client per server connection, and each client talks to exactly one server
- D. Servers create clients to call back into the host

??? success "Answer"
    **C.** The 1:1 client–server relationship isolates servers from each other. The host is the
    application; clients are connector components inside it.

---

**5.** Which of the following is **outside** the scope of the MCP specification?

- A. The JSON-RPC message format
- B. How a host decides which tool definitions to put in the LLM's context
- C. How a client declares its capabilities
- D. How a server reports a tool execution error

??? success "Answer"
    **B.** MCP "does not dictate how AI applications use LLMs or manage the provided context." Tool
    selection strategy is a host design decision. The other three are defined by the spec.

---

**6.** The specification says MCP "cannot enforce" its security principles at the protocol level. What does this imply?

- A. MCP has no security requirements
- B. Consent, authorization and access controls must be built by implementors, especially hosts
- C. Security is handled automatically by JSON-RPC
- D. Only remote servers need security

??? success "Answer"
    **B.** The spec defines requirements (such as obtaining consent before invoking tools) but relies on
    implementors to build consent and authorization flows. Nothing on the wire forces compliance.

---

**7.** Which component set **MUST** every MCP implementation support?

- A. Tools, resources and prompts
- B. Authorization, elicitation and logging
- C. The base protocol, versioning, and the message patterns
- D. Streamable HTTP and OAuth 2.1

??? success "Answer"
    **C.** "All implementations MUST support the base protocol, versioning, and the message patterns."
    Server features, client features, utilities and authorization are optional.

---

**8.** Which design principle is the direct reason a server cannot see results returned by another server?

- A. Servers should be extremely easy to build
- B. Servers should not be able to read the whole conversation, nor "see into" other servers
- C. Features can be added progressively
- D. Servers should be highly composable

??? success "Answer"
    **B.** Full conversation history stays with the host; each server is isolated and cross-server
    interactions are controlled by the host.

---

**9.** In the `2026-07-28` protocol, how does a server learn which protocol version and client capabilities apply to a request?

- A. From the `initialize` handshake at the start of the connection
- B. From the `Mcp-Session-Id` header
- C. From `_meta` fields carried on that request
- D. From the previous request on the same connection

??? success "Answer"
    **C.** MCP is stateless. `io.modelcontextprotocol/protocolVersion` and
    `io.modelcontextprotocol/clientCapabilities` are required on every request. A and B describe the legacy
    protocol; D is explicitly forbidden.

---

**10.** Where does composition of multiple MCP servers happen?

- A. Servers call each other directly over MCP
- B. In the host, which connects to each server through its own client
- C. In a central MCP broker required by the spec
- D. In the MCP Registry

??? success "Answer"
    **B.** The host composes servers. Servers never talk to each other in MCP, and no broker is required.

---

**11.** An MCP server is best described as:

- A. A physical machine running MCP software
- B. A hosted service operated by the MCP project
- C. A program that exposes capabilities through MCP, whether it runs locally or remotely
- D. A library that must be embedded in the host

??? success "Answer"
    **C.** "MCP server refers to the program that serves context data, regardless of where it runs."

---

**12.** Two connected servers both expose a tool named `search`. What should an aggregating host do?

- A. Refuse to connect the second server
- B. Rely on `serverInfo.name` to tell them apart
- C. Implement a disambiguation strategy such as prefixing tool names with a server identifier
- D. Nothing; the protocol guarantees global tool-name uniqueness

??? success "Answer"
    **C.** Tool names are unique only within a server. Hosts SHOULD disambiguate, and SHOULD NOT rely on
    `serverInfo.name`, which is not guaranteed unique.

---

**13.** What does a protocol version string such as `2026-07-28` represent?

- A. The release date of the latest SDK
- B. The last date backward-incompatible changes were made
- C. The date the exam was published
- D. The expiry date of the revision

??? success "Answer"
    **B.** Versions follow `YYYY-MM-DD` and indicate the last backward-incompatible change. Compatible
    improvements do not bump the version.

---

**14.** Which best describes what a **deprecated** feature means in MCP?

- A. It has been removed and no longer works
- B. It still works and remains in the spec, but new implementations should not adopt it
- C. It is experimental
- D. It is available only as an extension

??? success "Answer"
    **B.** Deprecated features remain part of the spec for at least twelve months before becoming eligible
    for removal. Sampling, Roots and Logging are deprecated, not removed.

---

**15.** Which statement about interoperability is TRUE?

- A. Every MCP host supports every MCP feature
- B. The protocol mandates how hosts display prompts
- C. Capability negotiation lets implementations with different feature sets work together on their common ground
- D. Server configuration files are standardized across all hosts

??? success "Answer"
    **C.** Implementations support different subsets and declare them via capabilities. UI and host
    configuration formats are not mandated by the spec.

---

| Score | Verdict |
| --- | --- |
| 14–15 | Solid. Move on. |
| 11–13 | Re-read [Core Concepts](core-concepts.md). |
| < 11 | Rework the domain before continuing; everything later builds on it. |
