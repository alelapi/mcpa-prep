---
title: Hands-on Labs
---

# Hands-on Labs

The MCPA exam is multiple choice and does not ask you to write code. So why do labs?

Because most exam questions describe a situation, and it is much easier to reason about a situation you
have already seen. Once you have watched a server reject a request with `-32020`, you do not need to
memorise what `HeaderMismatch` means. The same goes for:

- a `requestState` you tampered with getting refused,
- a `401` that carries a `resource_metadata` URL,
- a debug `print` that corrupts a stdio stream.

Every lab below was run against the official **Python SDK `mcp 2.2.0`**, which speaks protocol
version `2026-07-28`. Every output shown is real output, trimmed only for width.

## The labs

| Lab | You will | Main exam domains |
| --- | --- | --- |
| [0. Environment Setup](00-setup.md) | Install Python, the SDK and a working folder | none |
| [1. Speak MCP by Hand](01-raw-json-rpc.md) | Send raw JSON-RPC to a server over stdio and read every field | 2, 3 |
| [2. Build a Server](02-build-server.md) | Expose tools, resources and prompts; return proper errors; set cache hints | 1, 3 |
| [3. Build a Client](03-build-client.md) | Discover a server's features and map tools to an LLM's function-calling format | 2, 5 |
| [4. Streamable HTTP](04-streamable-http.md) | Drive a remote server with `curl` and trigger every header check | 2, 3, 4 |
| [5. Elicitation & MRTR](05-elicitation-mrtr.md) | Watch a tool pause for user input across several rounds | 3, 4 |
| [6. Security Hardening](06-security-hardening.md) | Protect a server with OAuth bearer tokens, scopes, audience checks and input policies | 4 |

Labs 1 to 5 each build on the previous one's files, so do them in order. Lab 6 stands on its own.

## How to use a lab

1. **Predict before you run.** Each lab has *Predict* boxes. Write your guess down before you run the
   code. A wrong prediction is where the learning happens.
2. **Type the code.** Don't copy and paste it. Typing makes you read every line.
3. **Break things on purpose.** The *Try it* boxes suggest changes that make the code fail in
   instructive ways.
4. **Link it back.** Each lab ends with *What this showed you*, which points to the study chapters
   the lab reinforces.

!!! warning "SDK behaviour is not the specification"
    An SDK is one implementation of the spec, and it sometimes differs from it. It may keep older
    behaviour for compatibility, or pick a default where the spec allows several. When the labs hit such
    a difference, they flag it in a box like this one.

    **The exam tests the specification, not the SDK.** When they disagree, answer with the spec.
