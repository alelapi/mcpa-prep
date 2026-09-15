---
title: Architecture & Components
---

# Domain 2 — Architecture & Components <span class="weight">14%</span>

How MCP is put together: the participants and their boundaries, the message format, the data
structures, how messages travel, how versions are agreed, and how a request moves from the user
through the model to a server and back.

## Competencies

| Competency | Chapter |
| --- | --- |
| Schemas & Structured Data | [Schemas & Structured Data](schemas.md) |
| MCP Hosts, Clients and Servers | [Hosts, Clients & Servers](hosts-clients-servers.md) |
| Model Interaction Flow | [Model Interaction Flow](interaction-flow.md) |

The exam focus statement also names **message flow**, **capabilities** and **transports**, which
are architectural. They are covered in [JSON-RPC Messages & `_meta`](messages.md),
[Transports](transports.md) and [Versioning & Discovery](versioning-discovery.md).

<div class="grid cards" markdown>

- **[Hosts, Clients & Servers](hosts-clients-servers.md)**: responsibilities, the 1:1 rule, local vs. remote, isolation.
- **[JSON-RPC Messages & `_meta`](messages.md)**: requests, results, errors, notifications, `resultType`, reserved metadata.
- **[Schemas & Structured Data](schemas.md)**: JSON Schema 2020-12, tool schemas, content blocks, annotations.
- **[Transports](transports.md)**: stdio and Streamable HTTP in full, headers, Origin validation.
- **[Versioning & Discovery](versioning-discovery.md)**: `server/discover`, version negotiation, extensions, dual-era compatibility.
- **[Model Interaction Flow](interaction-flow.md)**: the full path from user prompt to tool result, and where the model sits.
- **[Quiz](quiz.md)**: 16 questions.

</div>
