# Official Blueprint

The five domains and their competencies, as published on the
[Linux Foundation MCPA page](https://training.linuxfoundation.org/certification/model-context-protocol-associate-mcpa/),
mapped to the chapters that cover them.

!!! note "How to read this page"
    The Linux Foundation publishes only **competency names** for each domain, not
    detailed task statements. The descriptions under each competency below are this
    course's interpretation, grounded in the `2026-07-28` specification and in the
    exam focus statement: *"architecture, message flow, client-server interactions,
    capabilities, tools, resources, prompts, transports, and security
    considerations."*

---

## Domain 1 — MCP Fundamentals · 16%

| Competency | What it means in practice | Covered in |
| --- | --- | --- |
| **MCP Purpose & Scope** | The integration problem MCP solves; what the protocol covers and what it deliberately leaves out | [Why MCP Exists](../domain-1/purpose-and-scope.md) |
| **Core MCP Concepts** | Hosts, clients, servers; the three server primitives and their control model; JSON-RPC; capabilities; statelessness | [Core Concepts](../domain-1/core-concepts.md) |
| **Interoperability & Value** | Why a shared protocol matters to developers, AI applications and end users; composability; build once, integrate everywhere | [Interoperability & Value](../domain-1/interoperability.md) |

---

## Domain 2 — Architecture & Components · 14%

| Competency | What it means in practice | Covered in |
| --- | --- | --- |
| **Schemas & Structured Data** | JSON Schema 2020-12, `inputSchema` / `outputSchema`, `structuredContent`, content types, annotations, `$ref` rules | [Schemas & Structured Data](../domain-2/schemas.md) |
| **MCP Hosts, Clients and Servers** | The role of each participant, the 1:1 client–server rule, isolation between servers, design principles | [Hosts, Clients & Servers](../domain-2/hosts-clients-servers.md) |
| **Model Interaction Flow** | End to end: user → host → LLM → client → server → result → LLM. Where the model actually sits | [Model Interaction Flow](../domain-2/interaction-flow.md) |

Also architectural, and part of the exam focus statement:
[JSON-RPC Messages & `_meta`](../domain-2/messages.md),
[Transports](../domain-2/transports.md) and
[Versioning & Discovery](../domain-2/versioning-discovery.md).

---

## Domain 3 — Interactions & Execution · 26%

The heaviest domain.

| Competency | What it means in practice | Covered in |
| --- | --- | --- |
| **Interaction Patterns & Response Handling** | Request/response, Multi Round-Trip Requests, subscribe/notify; `resultType`; streaming vs. JSON responses | [MRTR & Elicitation](../domain-3/mrtr-elicitation.md), [Subscriptions, Progress & Cancellation](../domain-3/subscriptions-progress-cancellation.md) |
| **Error Handling** | Protocol errors vs. tool execution errors; the error code table; HTTP status mapping | [Error Handling](../domain-3/error-handling.md) |
| **Tool Invocation Lifecycle** | Discover → select → confirm → call → (input required → retry) → result → back to the model | [Tools & the Invocation Lifecycle](../domain-3/tools.md) |
| **Protocol Primitives** | Tools, resources, prompts, elicitation; utilities (caching, pagination, completion); deprecated sampling, roots, logging | [Resources & Prompts](../domain-3/resources-prompts.md), [Utilities](../domain-3/utilities.md), [Deprecated Features](../domain-3/deprecated-features.md) |

---

## Domain 4 — Security & Governance · 24%

| Competency | What it means in practice | Covered in |
| --- | --- | --- |
| **Trust Boundaries** | Where trust changes: user, host, model, client, server, upstream API. What each may see and do | [Trust Boundaries](../domain-4/trust-boundaries.md) |
| **Permissions & Consent** | Human in the loop, tool confirmation, elicitation rules, URL mode for sensitive data, OAuth scopes and least privilege | [Permissions & Consent](../domain-4/permissions-consent.md), [Authorization](../domain-4/authorization.md) |
| **Risk & Safety Controls** | Confused deputy, token passthrough, SSRF, state handle hijacking, local server compromise, prompt injection, tool poisoning | [Attacks & Mitigations](../domain-4/attacks-mitigations.md) |
| **Auditability & Observability** | Logging tool use, OpenTelemetry trace context in `_meta`, stderr, audit trails, identity for accountability | [Auditability & Observability](../domain-4/auditability.md) |

---

## Domain 5 — Use Cases & Ecosystem · 20%

| Competency | What it means in practice | Covered in |
| --- | --- | --- |
| **Roles, Responsibilities & Adoption** | Server authors, host developers, platform and security teams; governance (AAIF, Linux Foundation, SEPs, maintainers) | [Roles, Responsibilities & Adoption](../domain-5/roles-adoption.md) |
| **Operational Use Cases** | Local vs. remote servers, developer tools, enterprise data access, agents at scale, progressive discovery, code mode | [Operational Use Cases](../domain-5/operational-use-cases.md) |
| **Ecosystem & Portability** | SDKs and tiers, the MCP Registry, Inspector, extensions, backward compatibility across protocol eras | [Ecosystem & Portability](../domain-5/ecosystem-portability.md), [Extensions](../domain-5/extensions.md) |

---

## The recommended experience, restated as study targets

The Linux Foundation lists five areas of recommended experience. Each maps to
something you should be able to do by the end of this course.

1. **JSON-RPC or message-based protocols.** Read a raw MCP message and say whether it
   is a request, result, error or notification, and what it is doing.
2. **LLM API interaction.** Explain how a host turns MCP tool definitions into an LLM
   API call, and how a tool result flows back into the model's context.
3. **Agentic concepts.** Explain why tools are model-controlled, and where a human must
   stay in the loop.
4. **Security literacy.** Walk through the MCP OAuth 2.1 flow and explain why audience
   validation and PKCE are required.
5. **Reading manifests and capabilities.** Read a `server/discover` response or a tool
   definition and say what the server supports and what the client may do.
