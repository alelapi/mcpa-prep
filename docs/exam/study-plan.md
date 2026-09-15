# Study Plan

Two schedules. Pick one, then **book the exam before you start**. You have twelve months
from purchase and a retake included, so a date on the calendar costs little and helps a lot.

## 4-week plan (~6–8 h/week)

Recommended if MCP is new to you, or if you learned it before July 2026.

=== "Week 1 — Foundations & architecture"

    - [ ] [Exam Guide](index.md), [Blueprint](blueprint.md), **[What Changed](whats-new.md)**
    - [ ] Domain 1: [Why MCP Exists](../domain-1/purpose-and-scope.md), [Core Concepts](../domain-1/core-concepts.md), [Interoperability](../domain-1/interoperability.md)
    - [ ] Domain 2: [Hosts, Clients & Servers](../domain-2/hosts-clients-servers.md), [JSON-RPC Messages](../domain-2/messages.md)
    - [ ] [Lab 0](../labs/00-setup.md) and [Lab 1: Speak MCP by Hand](../labs/01-raw-json-rpc.md)
    - [ ] [Domain 1 quiz](../domain-1/quiz.md)

=== "Week 2 — Architecture & primitives"

    - [ ] Domain 2: [Schemas](../domain-2/schemas.md), [Transports](../domain-2/transports.md), [Versioning & Discovery](../domain-2/versioning-discovery.md), [Interaction Flow](../domain-2/interaction-flow.md)
    - [ ] Domain 3: [Tools](../domain-3/tools.md), [Resources & Prompts](../domain-3/resources-prompts.md)
    - [ ] [Lab 2: Build a Server](../labs/02-build-server.md), [Lab 3: Build a Client](../labs/03-build-client.md)
    - [ ] [Domain 2 quiz](../domain-2/quiz.md)

=== "Week 3 — Interactions (26%)"

    - [ ] [MRTR & Elicitation](../domain-3/mrtr-elicitation.md). Read it twice.
    - [ ] [Subscriptions, Progress & Cancellation](../domain-3/subscriptions-progress-cancellation.md)
    - [ ] [Error Handling](../domain-3/error-handling.md). Memorize the code table.
    - [ ] [Utilities](../domain-3/utilities.md), [Deprecated Features](../domain-3/deprecated-features.md)
    - [ ] [Lab 4: Streamable HTTP](../labs/04-streamable-http.md), [Lab 5: Elicitation & MRTR](../labs/05-elicitation-mrtr.md)
    - [ ] [Domain 3 quiz](../domain-3/quiz.md)

=== "Week 4 — Security, ecosystem, mocks"

    - [ ] All of [Domain 4](../domain-4/index.md) (24%)
    - [ ] [Lab 6: Security Hardening](../labs/06-security-hardening.md)
    - [ ] All of [Domain 5](../domain-5/index.md)
    - [ ] [Domain 4](../domain-4/quiz.md) and [Domain 5](../domain-5/quiz.md) quizzes
    - [ ] [Mock Exam 1](../practice/mock-exam-1.md), review, [Mock Exam 2](../practice/mock-exam-2.md)
    - [ ] [Flashcards](../practice/flashcards.md) the day before

## 10-day sprint (~2 h/day)

For people who already build MCP servers or clients.

| Day | Focus |
| --- | --- |
| 1 | [What Changed](whats-new.md), then [Mock Exam 1](../practice/mock-exam-1.md) cold to find gaps |
| 2 | Domain 2 in full, with [Lab 1](../labs/01-raw-json-rpc.md) |
| 3 | [Tools](../domain-3/tools.md), [MRTR & Elicitation](../domain-3/mrtr-elicitation.md), [Lab 5](../labs/05-elicitation-mrtr.md) |
| 4 | Rest of Domain 3, especially [Error Handling](../domain-3/error-handling.md) |
| 5 | [Authorization with OAuth 2.1](../domain-4/authorization.md) |
| 6 | [Attacks & Mitigations](../domain-4/attacks-mitigations.md), [Trust Boundaries](../domain-4/trust-boundaries.md) |
| 7 | Rest of Domain 4, [Lab 6](../labs/06-security-hardening.md) |
| 8 | Domain 5 and Domain 1 |
| 9 | [Mock Exam 2](../practice/mock-exam-2.md), then rework every miss |
| 10 | [Flashcards](../practice/flashcards.md), [Glossary](glossary.md), exam |

## How to study a protocol

!!! tip "Read messages, not just prose"
    For every mechanism, be able to **write the JSON** from memory: a `tools/call` request
    with its `_meta`, an `InputRequiredResult`, the retry with `inputResponses`, a
    `subscriptions/listen` acknowledgment. If you can write the message, you understand the
    mechanism well enough to reject the distractors.

!!! tip "Always ask: who sends this, and in which direction?"
    A large share of MCP questions reduces to direction. Clients send requests and
    notifications. Servers send responses and notifications, **never requests**. A question
    that has the server initiating a request is either describing the legacy protocol or is
    wrong.

!!! tip "Learn the MUSTs"
    The specification uses RFC 2119 keywords deliberately. Exam questions distinguish
    **MUST** from **SHOULD** from **MAY**. Where this course says MUST, the spec says MUST.
