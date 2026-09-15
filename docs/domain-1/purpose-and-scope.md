# Why MCP Exists

*Competency: **MCP Purpose & Scope**.*

---

## 1. The problem: models are isolated

A large language model on its own knows only what was in its training data. It cannot see
your files, query your database, read today's tickets, check a calendar or take any action
in the world. Every useful AI application therefore needs a way to **connect the model to
external systems**, both to **read context** and to **act**.

This is not a new observation, and before MCP every AI application solved it the same way:
by writing custom integration code.

Picture an organization in 2024:

- A chat assistant needs GitHub, Jira, Slack and the internal wiki.
- An IDE assistant needs GitHub, the filesystem, the build system and Jira.
- A customer-support agent needs Jira, the CRM, Slack and the knowledge base.

Each application wrote its **own** GitHub integration, its **own** Jira integration, its own
Slack integration. Each used different code, different auth handling, different error
handling and different ways of describing tools to the model. When Jira changed its API,
three teams fixed three integrations.

### The M × N problem

Count the work. With **M** AI applications and **N** external systems, point-to-point
integration needs up to **M × N** connectors:

```text
        Without a shared protocol                  With MCP

   App 1 ──┬── GitHub                      App 1 ─┐          ┌─ GitHub server
           ├── Jira                        App 2 ─┼── MCP ───┼─ Jira server
           └── Slack                       App 3 ─┘          ├─ Slack server
   App 2 ──┬── GitHub                                        └─ Wiki server
           ├── Jira
           └── Filesystem
   App 3 ──┬── Jira                        Each app implements MCP once (M).
           ├── CRM                         Each system gets one server (N).
           └── Slack                       Total work: M + N.

   Total work: up to M × N
```

Make it concrete. **10** AI applications and **50** data sources:

- Point to point: up to **10 × 50 = 500** integrations to write and maintain.
- Shared protocol: **10 + 50 = 60** implementations: each application speaks MCP once, and
  each system has one MCP server.

That arithmetic is the entire economic argument for MCP. It turns a multiplicative problem
into an additive one.

### The same problem, solved before

The pattern should look familiar, because it has been solved this way repeatedly:

**USB-C.** Before universal ports, every device had its own connector and every computer
needed the matching port. The official MCP documentation uses exactly this analogy: *"Think of
MCP like a USB-C port for AI applications."* A standard port means any device works with any
computer.

**The Language Server Protocol (LSP).** The analogy the **specification itself** cites. Before
LSP, adding support for a programming language meant writing a plugin for every editor: Go
for VS Code, Go for Vim, Go for Emacs, then Rust for each of them. LSP defined one protocol
between editors and "language servers". Now a language implements one server, an editor
implements one client, and they all interoperate. The spec says MCP *"takes some inspiration
from the Language Server Protocol"* and standardizes *"how to integrate additional context and
tools into the ecosystem of AI applications"* in a similar way.

!!! tip "Exam-ready framing"
    MCP solves the **M×N integration problem** by giving AI applications and external systems a
    **shared, open protocol**, the way USB-C did for devices and LSP did for editors and
    languages. Its inspiration, per the spec, is **LSP**.

---

## 2. What MCP is

The specification's own definition:

> MCP is an open protocol that enables seamless integration between LLM applications and
> external data sources and tools.

And the three things it standardizes. MCP provides a standardized way for applications to:

1. **Share contextual information** with language models
2. **Expose tools and capabilities** to AI systems
3. **Build composable integrations** and workflows

Unpack the key words, because each one is load-bearing.

**Open.** The specification, SDKs and reference implementations are public and permissively
licensed (Apache 2.0 for code and specifications). Anyone can implement MCP without
permission. In December 2025 Anthropic donated MCP to the **Agentic AI Foundation**, a
directed fund under the Linux Foundation, so it is also governed neutrally rather than by one
vendor. See [Roles & Adoption](../domain-5/roles-adoption.md).

**Protocol.** MCP is a **wire protocol**: a definition of the messages two programs exchange,
what they mean, and the rules for exchanging them. It is not a library, a product, a hosted
service or a framework. SDKs *implement* the protocol; the protocol is the specification.

**LLM applications** on one side, **data sources and tools** on the other. MCP sits in the
middle.

---

## 3. What MCP covers

The specification is organized into layers. All implementations **MUST** support the base
protocol, versioning and the message patterns. Everything else is optional and implemented as
needed.

| Component | What it defines | Required? |
| --- | --- | --- |
| **Base protocol** | JSON-RPC message types, errors, `_meta`, statelessness | **MUST** |
| **Versioning & compatibility** | Per-request version, extension negotiation, legacy interop | **MUST** |
| **Message patterns** | Request/response, Multi Round-Trip Requests, subscribe/notify | **MUST** |
| **Authorization** | OAuth 2.1-based framework for HTTP transports | Optional |
| **Server features** | Tools, resources, prompts | Optional (per need) |
| **Client features** | Elicitation (plus deprecated sampling and roots) | Optional |
| **Utilities** | Caching, pagination, completion, progress, cancellation, (deprecated) logging | Optional |

Beyond the core, **extensions** add optional, opt-in functionality such as Tasks, MCP Apps and
Skills.

The MCP **project** is broader than the specification. It also includes:

- **SDKs** in many languages (TypeScript, Python, C#, Go and Rust are Tier 1)
- **Development tools**, notably the **MCP Inspector**
- **Reference server implementations**
- The **MCP Registry** for discovering public servers

---

## 4. What MCP deliberately does not cover

Scope boundaries are as examinable as scope, and several are stated explicitly.

### MCP does not dictate how the LLM is used

From the architecture documentation:

> MCP focuses solely on the protocol for context exchange — it does not dictate how AI
> applications use LLMs or manage the provided context.

MCP says nothing about which model you call, how you build the prompt, how many tool
definitions you put in context, how you rank resources, or how the agent loop works. A host
could hand every tool to the model, search for the right three, or let a smaller model choose.
All of those are compliant.

This matters in exam scenarios. If a question asks how a host should choose which tool
definitions to include in a prompt, the answer is a **host design decision**, not a protocol
rule.

### MCP does not enforce security

This is stated plainly in the specification:

> While MCP itself cannot enforce these security principles at the protocol level,
> implementors **SHOULD** build robust consent and authorization flows into their
> applications…

The protocol **defines** requirements (for example, that hosts must obtain consent before
invoking tools) but **cannot enforce** them. A malicious or careless host can ignore them, and
nothing in the wire format stops it. Consent, confirmation UI, sandboxing and access control
are **implementation** responsibilities. Domain 4 builds on this.

### MCP does not mandate user interfaces

Every primitive page in the spec includes a variant of: *"implementations are free to expose
[this] through any interface pattern that suits their needs — the protocol itself does not
mandate any specific user interaction model."* Slash commands for prompts, a file picker for
resources, confirmation dialogs for tools are all **suggestions**.

### MCP does not provide the connection to the model

MCP connects the **application** to external systems. The application's own connection to its
LLM (an Anthropic, OpenAI or local-model API call) is outside MCP. This is exactly why
**sampling** was deprecated: it pulled the client's model connection into MCP, and the
recommended migration is for servers to *"integrate directly with LLM provider APIs"*.

### MCP does not replace existing APIs

An MCP server is typically a **thin adapter** in front of an existing API, database or file
system. GitHub keeps its REST API. The GitHub MCP server translates between MCP messages and
that API, and presents the result in a form an AI application can use.

---

## 5. What MCP enables

The official documentation gives concrete examples of what becomes possible when AI
applications can reach external systems through a standard interface:

- **Personal assistants** that access your Google Calendar and Notion.
- **Coding agents** that generate a whole web app from a Figma design.
- **Enterprise chatbots** that connect to multiple databases across an organization so people
  can analyze data by chatting.
- **Creative and physical workflows**, such as a model creating a 3D design in Blender and
  sending it to a 3D printer.

The common thread: the AI application gains **context** (reading calendars, designs, database
schemas) and **agency** (creating events, writing code, starting a print) without the
application developer writing a bespoke integration for each system.

---

## 6. A short history

| Date | Event |
| --- | --- |
| **Nov 2024** | Anthropic open-sources MCP. First spec revision `2024-11-05`; stdio and HTTP+SSE transports |
| **Mar 2025** | `2025-03-26`: Streamable HTTP replaces HTTP+SSE; OAuth-based authorization framework; tool annotations |
| **Jun 2025** | `2025-06-18`: structured tool output, elicitation, resource links, servers classified as OAuth resource servers, `MCP-Protocol-Version` header |
| **Sep 2025** | MCP Registry launches in preview |
| **Nov 2025** | `2025-11-25`: URL-mode elicitation, tasks (experimental), Client ID Metadata Documents |
| **Dec 2025** | MCP donated to the **Agentic AI Foundation** under the Linux Foundation |
| **Jul 2026** | `2026-07-28`: **stateless redesign**, MRTR, `server/discover`, `subscriptions/listen`; sampling, roots and logging deprecated |

You will not be asked to recite dates. The history matters because it explains why so much
existing material is out of date, and why the specification version is part of every request.

---

## 7. Recap

- LLMs are isolated from data and tools. Before MCP, every AI application wrote custom
  integrations: the **M×N problem**.
- A shared protocol makes it **M + N**: each application implements MCP once, each system has
  one server.
- MCP is an **open wire protocol** built on JSON-RPC, inspired by the **Language Server
  Protocol**, often explained as **USB-C for AI applications**.
- It standardizes three things: sharing **context**, exposing **tools**, building
  **composable** integrations.
- Required of every implementation: **base protocol, versioning, message patterns**. Everything
  else is optional.
- Out of scope: **how the LLM is used**, **enforcement** of security (implementors must build it),
  **UI patterns**, the application's own **model connection**, and replacing existing APIs.
