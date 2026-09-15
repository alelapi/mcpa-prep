---
title: Interactions & Execution
---

# Domain 3 — Interactions & Execution <span class="weight">26%</span>

The heaviest domain, and the most mechanical. It covers how clients and servers actually do
things together: calling tools, reading resources, getting prompts, pausing for user input,
streaming progress, cancelling, handling failure, caching and paginating.

Most of what changed in `2026-07-28` lives here. If you study one domain twice, study this one.

## Competencies

| Competency | Chapters |
| --- | --- |
| Interaction Patterns & Response Handling | [MRTR & Elicitation](mrtr-elicitation.md), [Subscriptions, Progress & Cancellation](subscriptions-progress-cancellation.md) |
| Error Handling | [Error Handling](error-handling.md) |
| Tool Invocation Lifecycle | [Tools & the Invocation Lifecycle](tools.md) |
| Protocol Primitives | [Resources & Prompts](resources-prompts.md), [Caching, Pagination & Completion](utilities.md), [Deprecated Features](deprecated-features.md) |

## The three message patterns

Every interaction in this domain is built from exactly three patterns. The specification says all core
features are built from them, and every transport carries all three.

| Pattern | Shape | Used for |
| --- | --- | --- |
| **Request and response** | Client request → (optional request-scoped notifications) → result or error | Almost everything |
| **Multi Round-Trip Requests** | Request → `InputRequiredResult` → client retries with input → result | Elicitation, and deprecated sampling and roots |
| **Subscribe and notify** | `subscriptions/listen` → acknowledgment → stream of notifications | List changes, resource updates |

<div class="grid cards" markdown>

- **[Tools & the Invocation Lifecycle](tools.md)**: discovery, selection, confirmation, execution, results, stateful handles.
- **[Resources & Prompts](resources-prompts.md)**: URIs, templates, reading, subscriptions, prompt templates.
- **[MRTR & Elicitation](mrtr-elicitation.md)**: `InputRequiredResult`, `requestState`, form and URL elicitation.
- **[Subscriptions, Progress & Cancellation](subscriptions-progress-cancellation.md)**: `subscriptions/listen`, progress tokens, cancellation and timeouts.
- **[Error Handling](error-handling.md)**: protocol vs. tool errors, every code, HTTP statuses.
- **[Caching, Pagination & Completion](utilities.md)**: `ttlMs`, `cacheScope`, cursors, argument completion.
- **[Deprecated: Sampling, Roots & Logging](deprecated-features.md)**: how they work, why they are deprecated, what to use instead.
- **[Quiz](quiz.md)**: 20 questions.

</div>
