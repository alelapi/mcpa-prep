# Official Resources

The primary sources the exam is built on. The specification is authoritative. Everything
else, including this course, is secondary.

## Certification

- [MCPA certification page](https://training.linuxfoundation.org/certification/model-context-protocol-associate-mcpa/)
- [Candidate Handbook](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2)
- [Certification FAQ](https://docs.linuxfoundation.org/tc-docs/certification/faq-mc)
- [Important Instructions](https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-mc)
- [Global Certification & Confidentiality Agreement](https://docs.linuxfoundation.org/tc-docs/certification/lf-cert-agreement/)

## The specification (`2026-07-28`)

Read these in this order:

1. [Specification overview](https://modelcontextprotocol.io/specification/2026-07-28)
2. [Key Changes (changelog)](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
3. [Architecture](https://modelcontextprotocol.io/specification/2026-07-28/architecture)
4. [Base protocol](https://modelcontextprotocol.io/specification/2026-07-28/basic): messages, errors, statelessness, `_meta`, JSON Schema usage
5. [Versioning and Compatibility](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)
6. Message patterns: [overview](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns), [MRTR](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr), [Subscriptions](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions), [Cancellation](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/cancellation), [Progress](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/progress)
7. Transports: [overview](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports), [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio), [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
8. Server features: [Discovery](https://modelcontextprotocol.io/specification/2026-07-28/server/discover), [Tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools), [Resources](https://modelcontextprotocol.io/specification/2026-07-28/server/resources), [Prompts](https://modelcontextprotocol.io/specification/2026-07-28/server/prompts)
9. Utilities: [Caching](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/caching), [Pagination](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/pagination), [Completion](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/completion), [Logging](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/logging)
10. Client features: [Elicitation](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation), [Sampling](https://modelcontextprotocol.io/specification/2026-07-28/client/sampling), [Roots](https://modelcontextprotocol.io/specification/2026-07-28/client/roots)
11. Authorization: [overview](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization), [AS discovery](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery), [Client registration](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration), [Security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)
12. [Deprecated features registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)
13. [Schema reference](https://modelcontextprotocol.io/specification/2026-07-28/schema) and [`schema.ts`](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/schema/2026-07-28/schema.ts), the source of truth

!!! tip "Machine-readable docs"
    Every page of modelcontextprotocol.io is available as raw Markdown by appending `.md` to
    its URL, and [llms.txt](https://modelcontextprotocol.io/llms.txt) lists them all. That is
    the easiest way to read the whole spec offline.

## Learning guides

- [What is MCP?](https://modelcontextprotocol.io/docs/getting-started/intro)
- [Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture): the best single conceptual walkthrough
- [Understanding MCP servers](https://modelcontextprotocol.io/docs/learn/server-concepts)
- [Understanding MCP clients](https://modelcontextprotocol.io/docs/learn/client-concepts)
- [Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices): **required reading for Domain 4**
- [Understanding Authorization](https://modelcontextprotocol.io/docs/tutorials/security/authorization)
- [Client Best Practices](https://modelcontextprotocol.io/docs/develop/clients/client-best-practices): progressive discovery and code mode
- [Build a server](https://modelcontextprotocol.io/docs/develop/build-server) and [Build a client](https://modelcontextprotocol.io/docs/develop/build-client)
- [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector) and [Debugging](https://modelcontextprotocol.io/docs/tools/debugging)

## Extensions

- [Extensions overview](https://modelcontextprotocol.io/extensions/overview)
- [Tasks](https://modelcontextprotocol.io/extensions/tasks/overview), [MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview), [Skills](https://modelcontextprotocol.io/extensions/skills/overview)
- [Authorization extensions](https://modelcontextprotocol.io/extensions/auth/overview): [OAuth Client Credentials](https://modelcontextprotocol.io/extensions/auth/oauth-client-credentials), [Enterprise-Managed Authorization](https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization)
- [Extension support matrix](https://modelcontextprotocol.io/extensions/client-matrix)

## Ecosystem and governance

- [Agentic AI Foundation (AAIF)](https://aaif.io/) and [MCP joins the AAIF](https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/)
- [Governance and Stewardship](https://modelcontextprotocol.io/community/governance)
- [SEP Guidelines](https://modelcontextprotocol.io/community/sep-guidelines)
- [SDKs](https://modelcontextprotocol.io/docs/sdk)
- [MCP Registry](https://registry.modelcontextprotocol.io/) and its [documentation](https://github.com/modelcontextprotocol/registry/tree/main/docs)
- [MCP Blog](https://blog.modelcontextprotocol.io/)

## Standards MCP builds on

You need to recognize these, not read them end to end.

| Standard | Role in MCP |
| --- | --- |
| [JSON-RPC 2.0](https://www.jsonrpc.org/specification) | Message format |
| [JSON Schema 2020-12](https://json-schema.org/draft/2020-12) | Tool input/output schemas, elicitation forms |
| [Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html) | Streaming responses on Streamable HTTP |
| [OAuth 2.1 (draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-13) | Authorization framework |
| [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728) Protected Resource Metadata | How an MCP server points to its authorization server |
| [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) AS Metadata / OIDC Discovery | How a client learns authorization server endpoints |
| [RFC 8707](https://www.rfc-editor.org/rfc/rfc8707.html) Resource Indicators | Binding tokens to a specific MCP server |
| [RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207) Issuer Identification | Preventing mix-up attacks |
| [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750) Bearer Tokens | `Authorization: Bearer` and `WWW-Authenticate` |
| [Client ID Metadata Documents](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-client-id-metadata-document-00) | Preferred client registration |
| [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) Dynamic Client Registration | Deprecated client registration |
| [RFC 6570](https://datatracker.ietf.org/doc/html/rfc6570) URI Templates | Resource templates |
| [W3C Trace Context](https://www.w3.org/TR/trace-context/) | `traceparent` in `_meta` |
