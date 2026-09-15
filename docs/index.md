# MCPA Preparation Course

A complete, self-paced course for the **Linux Foundation Model Context Protocol
Associate (MCPA)** certification.

!!! info "Exam at a glance"
    | | |
    | --- | --- |
    | **Certification** | Model Context Protocol Associate (MCPA) |
    | **Format** | Online, proctored, multiple choice |
    | **Duration** | 90 minutes |
    | **Price** | $250 USD (one retake included) |
    | **Level** | Beginner / associate |
    | **Prerequisites** | None formal |
    | **Validity** | 2 years |
    | **Eligibility window** | 12 months from purchase |
    | **Spec version referenced** | **MCP `2026-07-28`** |

    Source: [Linux Foundation certification page](https://training.linuxfoundation.org/certification/model-context-protocol-associate-mcpa/)

## The five domains

| # | Domain | Weight |
| --- | --- | --- |
| 1 | [MCP Fundamentals](domain-1/index.md) | **16%** |
| 2 | [Architecture & Components](domain-2/index.md) | **14%** |
| 3 | [Interactions & Execution](domain-3/index.md) | **26%** |
| 4 | [Security & Governance](domain-4/index.md) | **24%** |
| 5 | [Use Cases & Ecosystem](domain-5/index.md) | **20%** |

Half the exam is Domains 3 and 4. That is where most of your study time should go.

## Read this before anything else

!!! danger "The exam targets a protocol that changed fundamentally in July 2026"
    The Linux Foundation page links the exam to **MCP specification revision
    `2026-07-28`**. That revision redesigned the protocol:

    - The `initialize` handshake is **gone**. MCP is now **stateless**. Every request
      carries its own protocol version and client capabilities in `_meta`.
    - Protocol-level **sessions are gone** (no more `Mcp-Session-Id`).
    - Servers **no longer send requests to clients**. When a server needs input, it
      returns an `InputRequiredResult` and the client **retries** the request
      (Multi Round-Trip Requests).
    - **Sampling, Roots and Logging are deprecated.**
    - **Tasks** moved out of the core protocol into an extension.
    - `ping` was removed. A mandatory `server/discover` RPC was added.

    Most blog posts, tutorials, YouTube videos and older AI answers describe the
    **previous** protocol (`2025-11-25` and earlier). If you learned MCP from those,
    some of what you know is now wrong in ways the exam can test.

    **Start with [What Changed in 2026-07-28](exam/whats-new.md).** This whole course
    is written against the `2026-07-28` specification. Legacy behavior is covered
    only where you need to recognize it.

## How to use this course

1. **Read the [blueprint](exam/blueprint.md)** so you know what each domain covers.
2. **Read [What Changed](exam/whats-new.md)** even if you already use MCP.
3. **Follow the [study plan](exam/study-plan.md).**
4. **Work the chapters in order.** Each builds on the previous one, and each domain
   ends with a quiz.
5. **Do the [labs](labs/index.md).** MCP is a wire protocol. Watching the actual
   JSON-RPC messages move is the fastest way to make it concrete. Every lab was run
   against SDK `mcp 2.2.0`, which implements `2026-07-28`.
6. **Take the [mock exams](practice/index.md)** under timed conditions.

## What the exam tests

The Linux Foundation describes it as focusing on *"MCP internals, including
architecture, message flow, client-server interactions, capabilities, tools,
resources, prompts, transports, and security considerations."*

In practice that means three kinds of question:

- **Recognition.** What is `server/discover` for? What does `resultType:
  "input_required"` mean? Which primitive is user-controlled?
- **Mechanics.** A server needs the user to confirm an action halfway through a
  tool call. What goes over the wire, and in what order?
- **Judgement.** A remote MCP server receives a token whose audience is a different
  service. What must it do? Which transport fits a local, single-user integration?

The recommended background the exam assumes is: JSON-RPC basics,
experience calling LLM APIs, familiarity with agents and tool use, and basic
security literacy (API keys, OAuth 2.1, tokens). Where this course relies on those,
it explains them from scratch.

---

!!! warning "Disclaimer"
    Unofficial study material. Not affiliated with or endorsed by the Linux
    Foundation, the Agentic AI Foundation or the MCP project. All practice questions
    are original and are **not** real exam questions.
