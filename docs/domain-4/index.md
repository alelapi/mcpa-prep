---
title: Security & Governance
---

# Domain 4 — Security & Governance <span class="weight">24%</span>

Almost a quarter of the exam. MCP connects language models, which can be manipulated, to tools that
can act and data that can leak, across software written by strangers. Security is not an add-on to
MCP; it is the central design problem.

## Start from the specification's own principles

The specification's overview states three key principles. Every chapter in this domain elaborates one or more of them.

**1. User consent and control**

- Users must explicitly consent to and understand all data access and operations
- Users must retain control over what data is shared and what actions are taken
- Implementors should provide clear UIs for reviewing and authorizing activities

**2. Data privacy**

- Hosts must obtain explicit user consent before exposing user data to servers
- Hosts must not transmit resource data elsewhere without user consent
- User data should be protected with appropriate access controls

**3. Tool safety**

- Tools represent arbitrary code execution and must be treated with appropriate caution
- Descriptions of tool behavior, such as annotations, should be considered **untrusted** unless obtained from a trusted server
- Hosts must obtain explicit user consent before invoking any tool
- Users should understand what each tool does before authorizing its use

And the sentence that frames the entire domain:

> While MCP itself **cannot enforce** these security principles at the protocol level, implementors **SHOULD**:
> build robust consent and authorization flows; provide clear documentation of security implications; implement
> appropriate access controls and data protections; follow security best practices; consider privacy implications
> in their feature designs.

## Competencies

| Competency | Chapter |
| --- | --- |
| Trust Boundaries | [Trust Boundaries](trust-boundaries.md) |
| Permissions & Consent | [Permissions & Consent](permissions-consent.md), [Authorization with OAuth 2.1](authorization.md) |
| Risk & Safety Controls | [Attacks & Mitigations](attacks-mitigations.md) |
| Auditability & Observability | [Auditability & Observability](auditability.md) |

<div class="grid cards" markdown>

- **[Trust Boundaries](trust-boundaries.md)**: who trusts whom, and what crosses each boundary.
- **[Authorization with OAuth 2.1](authorization.md)**: discovery, registration, PKCE, resource indicators, scopes, token validation.
- **[Permissions & Consent](permissions-consent.md)**: human in the loop, confirmation, least privilege, consent screens.
- **[Attacks & Mitigations](attacks-mitigations.md)**: confused deputy, token passthrough, SSRF, hijacking, local compromise, injection, poisoning.
- **[Auditability & Observability](auditability.md)**: logging, OpenTelemetry, identity for accountability.
- **[Quiz](quiz.md)**: 20 questions.

</div>

!!! tip "Primary reading for this domain"
    [Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices) and the
    [Authorization specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) with its
    [security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations).
    This course follows them closely.
