# Authorization with OAuth 2.1

*Competencies: **Permissions & Consent**, **Trust Boundaries**. The recommended background explicitly lists
"OAuth 2.1, token handling, authentication".*

This is the most standards-heavy chapter in the course. It builds the MCP authorization flow step by step,
explaining **why** each piece exists: every MUST in this part of the spec closes a specific, known attack.

---

## 1. When authorization applies

Authorization is **OPTIONAL** for MCP implementations. When supported:

| Transport | Rule |
| --- | --- |
| **HTTP-based** (Streamable HTTP) | **SHOULD** conform to the MCP authorization specification |
| **stdio** | **SHOULD NOT** follow it; **retrieve credentials from the environment** instead |
| **Alternative transports** | **MUST** follow established security best practices for their protocol |

Why not OAuth on stdio? A stdio server is a **subprocess the client launched**. There is no network endpoint to protect,
no way for anyone else to reach it, and no browser redirect target. The launching host simply passes credentials such as
`GITHUB_TOKEN` in the environment.

Clients and servers **MAY** also negotiate custom strategies, and **authorization extensions** cover machine-to-machine and
enterprise scenarios (section 12).

---

## 2. The roles

MCP maps directly onto OAuth roles:

| OAuth role | MCP participant | Responsibility |
| --- | --- | --- |
| **Resource server** | The protected **MCP server** | Accepts and validates access tokens |
| **Client** | The **MCP client** | Obtains tokens and makes requests on behalf of the user |
| **Resource owner** | The **user** | Grants consent |
| **Authorization server (AS)** | May be co-located with the MCP server, or a **separate** identity provider | Authenticates the user, issues tokens |

Classifying the MCP server as a **resource server** (since `2025-06-18`) is what lets organizations put MCP behind their
**existing** identity provider (Okta, Entra ID, Auth0, Keycloak) instead of each MCP server running its own login system.

---

## 3. The standards involved

MCP implements a **selected subset** of these, for security and interoperability:

| Standard | Why MCP uses it |
| --- | --- |
| **OAuth 2.1** (draft) | The base framework: authorization code + PKCE, no implicit flow |
| **RFC 6750** Bearer Token Usage | `Authorization: Bearer`, `WWW-Authenticate` challenges |
| **RFC 9728** Protected Resource Metadata | How the MCP server tells clients **which AS to use** |
| **RFC 8414** AS Metadata / **OIDC Discovery** | How the client learns **AS endpoints** and capabilities |
| **RFC 8707** Resource Indicators | Binding a token to **one MCP server** |
| **RFC 9207** Issuer Identification | Stopping **mix-up attacks** |
| **Client ID Metadata Documents** (draft) | Registering clients with **no prior relationship** |
| **RFC 7591** Dynamic Client Registration | Legacy registration (**deprecated**) |

---

## 4. Requirements overview

1. Authorization servers **MUST** implement **OAuth 2.1** with appropriate security for confidential and public clients.
2. ASes and clients **SHOULD** support **Client ID Metadata Documents**.
3. ASes and clients **MAY** support **Dynamic Client Registration**, which is **deprecated** and retained for compatibility.
4. MCP servers **MUST** implement **Protected Resource Metadata** (RFC 9728), and clients **MUST** use it for AS discovery.
5. ASes **MUST** provide at least one of **RFC 8414 AS Metadata** or **OIDC Discovery**, and clients **MUST support both**.

---

## 5. The complete flow

Here is the whole sequence first, then each step in detail.

```mermaid
sequenceDiagram
    participant B as Browser
    participant C as MCP client
    participant M as MCP server (resource server)
    participant A as Authorization server

    C->>M: MCP request without token
    M-->>C: 401 + WWW-Authenticate: resource_metadata=..., scope=...
    C->>M: GET Protected Resource Metadata
    M-->>C: { resource, authorization_servers: [...], scopes_supported }
    Note over C: choose an AS
    C->>A: GET AS metadata (.well-known, in priority order)
    A-->>C: { issuer, authorization_endpoint, token_endpoint,<br/>code_challenge_methods_supported, ... }
    Note over C: validate issuer; verify PKCE support
    alt Client ID Metadata Document
        Note over C: client_id = https URL of its metadata
        A->>C: (AS fetches client metadata URL)
    else Dynamic Client Registration (deprecated)
        C->>A: POST /register
        A-->>C: client_id
    else Pre-registered
        Note over C: use configured client_id
    end
    Note over C: create PKCE verifier + S256 challenge,<br/>state, resource=canonical MCP URI,<br/>record expected issuer
    C->>B: open authorization URL
    B->>A: authorize (code_challenge, resource, scope, state)
    Note over A: user logs in and consents
    A-->>B: redirect: code, state, iss
    B->>C: callback
    Note over C: verify state; validate iss == recorded issuer
    C->>A: token request (code, code_verifier, resource)
    A-->>C: access token (+ refresh token)
    C->>M: MCP request + Authorization: Bearer token
    Note over M: validate token, including audience = this server
    M-->>C: MCP response
```

---

## 6. Step 1: Discovering the authorization server

### The 401 challenge

A client first calls the MCP server without a token. The server answers **`401 Unauthorized`**:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource",
                         scope="files:read"
```

- `resource_metadata` points to the server's **Protected Resource Metadata** (PRM) document.
- `scope` (optional, **SHOULD** be included) tells the client which scopes the request needs.

Clients **MUST** be able to parse `WWW-Authenticate` and respond to 401s.

### Protected Resource Metadata (RFC 9728)

MCP servers **MUST** implement PRM, and it **MUST** include **`authorization_servers`** with **at least one** AS:

```json
{
  "resource": "https://mcp.example.com/mcp",
  "authorization_servers": ["https://auth.example.com"],
  "scopes_supported": ["files:read", "files:write"],
  "bearer_methods_supported": ["header"]
}
```

Servers **MUST** make PRM discoverable in **one** of two ways, and clients **MUST support both**:

1. **`WWW-Authenticate` header** with `resource_metadata` on 401 responses.
2. **Well-known URI**:
    - **path-specific**: MCP endpoint `https://example.com/public/mcp` → `https://example.com/.well-known/oauth-protected-resource/public/mcp`
    - **root**: `https://example.com/.well-known/oauth-protected-resource`

Clients **MUST** use the header's URL when present, otherwise **fall back** to the well-known URIs **in that order**
(path-specific first, then root).

**Multiple authorization servers.** PRM may list several. The **client chooses**. Each is an **independent** AS: client
identifiers are unique to the AS that issued them, so clients **MUST** keep **separate registration state and tokens per AS** and
**MUST NOT** assume credentials from one work with another.

### Authorization server metadata

With the AS issuer URL in hand, the client fetches its metadata. MCP uses the default `oauth-authorization-server` suffix and
defines no MCP-specific well-known URI. Clients **MUST** try several locations **in priority order**:

**Issuer with a path**, e.g. `https://auth.example.com/tenant1`:

1. `https://auth.example.com/.well-known/oauth-authorization-server/tenant1` (OAuth, path **insertion**)
2. `https://auth.example.com/.well-known/openid-configuration/tenant1` (OIDC, path **insertion**)
3. `https://auth.example.com/tenant1/.well-known/openid-configuration` (OIDC, path **appending**)

**Issuer without a path**, e.g. `https://auth.example.com`:

1. `https://auth.example.com/.well-known/oauth-authorization-server`
2. `https://auth.example.com/.well-known/openid-configuration`

### Validate the issuer

After fetching metadata, clients **MUST** check that its **`issuer` is identical** to the issuer identifier used to build the URL.
If they differ, the client **MUST NOT** use the metadata.

> A document fetched from `https://attacker.example/.well-known/oauth-authorization-server` that contains
> `"issuer": "https://honest.example"` **MUST** be rejected.

Without this check, an attacker hosting metadata could **claim** to be an honest AS, and every later step would trust the wrong endpoints.

---

## 7. Step 2: Client registration

The client needs a `client_id` from the AS. There is often **no prior relationship**: an MCP client like an IDE may connect to a
server the IDE's developers have never heard of. MCP supports three mechanisms.

### Selection priority

Clients supporting all options **SHOULD** use this order:

1. **Pre-registered** client information, if available for this server
2. **Client ID Metadata Documents**, if the AS advertises `client_id_metadata_document_supported: true`
3. **Dynamic Client Registration**, if the AS has a `registration_endpoint`
4. **Prompt the user** to enter client information

### Client ID Metadata Documents (CIMD), the preferred option

The client uses **an HTTPS URL as its `client_id`**. That URL serves a JSON document describing the client:

```json
{
  "client_id": "https://app.example.com/oauth/client-metadata.json",
  "client_name": "Example MCP Client",
  "client_uri": "https://app.example.com",
  "logo_uri": "https://app.example.com/logo.png",
  "redirect_uris": ["http://127.0.0.1:3000/callback", "http://localhost:3000/callback"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

When the AS sees a URL-formatted `client_id`, it **fetches** the document and validates it.

**Client requirements**

- Host the document at an **HTTPS URL with a path component** (e.g. `https://example.com/client.json`).
- Include at least **`client_id`**, **`client_name`**, **`redirect_uris`**.
- The document's `client_id` **MUST exactly match** its URL.
- **MAY** use `private_key_jwt` for client authentication.

**AS requirements**

- **SHOULD** fetch documents for URL-formatted `client_id`s, and **SHOULD** cache them respecting HTTP cache headers.
- **MUST** validate the fetched `client_id` matches the URL exactly.
- **MUST** validate the **redirect URI** in the authorization request against the document's `redirect_uris`.
- **MUST** validate the document is valid JSON with required fields.
- **SHOULD** follow CIMD security considerations: **SSRF** protection (the AS is fetching a URL an unknown party supplied),
  **trust policies**, and **localhost warnings**.

**Why it's preferred:** no registration API, no database of registered clients on the AS, and the identity is **portable**: a CIMD
client ID works with **any** AS that supports CIMD, with no re-registration when the AS changes.

**Its limitation:** CIMD proves the client **controls a domain**. It cannot prove **which local process** is listening on a `localhost`
redirect URI. An attacker can present a legitimate client's metadata URL and a `localhost` redirect of their own. So ASes **SHOULD**
warn on `localhost`-only redirect URIs, **MUST** clearly display the redirect URI hostname, and **MAY** require extra attestation.

### Pre-registration

For clients and servers with an existing relationship. Clients **SHOULD** support static credentials: a hardcoded `client_id` for a
known AS, or a UI where the user enters credentials they registered themselves.

### Dynamic Client Registration (deprecated)

The client `POST`s to the AS's `registration_endpoint` (RFC 7591) and receives a `client_id` with no user interaction. **Deprecated in
`2026-07-28`**, retained for ASes without CIMD support.

A detail added in this revision: clients **MUST** specify an appropriate **`application_type`**. Under OIDC an omitted value defaults to
`"web"`, which can conflict with native-style redirect URIs.

| Client kind | `application_type` |
| --- | --- |
| Desktop apps, mobile apps, CLI tools, **local web apps on `localhost`** | `"native"` |
| Remote browser apps on a non-local host | `"web"` |

### Authorization server binding

Clients using pre-registered or persisted DCR credentials **MUST**:

- **key those credentials by the AS `issuer`**,
- **MUST NOT reuse** credentials with a **different** AS,
- **MUST re-register** when the AS changes (detected via updated PRM).

Pre-registered credentials whose AS no longer matches **SHOULD** produce an error rather than a silent attempt. CIMD client IDs don't need
this, because they are portable.

---

## 8. Step 3: The authorization request

### PKCE: mandatory

**PKCE** (Proof Key for Code Exchange) stops a stolen **authorization code** from being redeemed.

1. The client generates a random secret **`code_verifier`**.
2. It sends **`code_challenge = BASE64URL(SHA256(code_verifier))`** with `code_challenge_method=S256` in the authorization request.
3. At the token endpoint it sends the original **`code_verifier`**.
4. The AS hashes it and compares with the challenge from step 2.

An attacker who intercepts the code (a malicious app registered for the same redirect scheme, a leaked log) doesn't have the verifier, so
the code is useless to them.

MCP requirements:

- Clients **MUST implement PKCE**.
- Clients **MUST use `S256`** when technically capable.
- Clients **MUST verify PKCE support before proceeding**: if AS metadata lacks **`code_challenge_methods_supported`**, the client **MUST refuse
  to proceed**. This applies to OIDC providers too, even though OIDC doesn't define that field.
- ASes providing OIDC Discovery **MUST** include `code_challenge_methods_supported`.

### Resource indicators: mandatory

The **`resource`** parameter (RFC 8707) names **which MCP server** the token is for. Clients:

1. **MUST include it** in **both** the authorization request **and** the token request.
2. **MUST** set it to the MCP server they intend to use the token with.
3. **MUST** use the server's **canonical URI**.
4. **MUST send it regardless of whether the AS supports it.**

**Canonical server URI:**

| Valid | Invalid |
| --- | --- |
| `https://mcp.example.com/mcp` | `mcp.example.com` (no scheme) |
| `https://mcp.example.com` | `https://mcp.example.com#fragment` (fragment) |
| `https://mcp.example.com:8443` | |
| `https://mcp.example.com/server/mcp` (path needed to identify the server) | |

Provide the **most specific** URI. Prefer **no trailing slash** unless it's significant. Implementations **SHOULD** accept uppercase scheme or
host for robustness, though the canonical form is lowercase.

```text
&resource=https%3A%2F%2Fmcp.example.com
```

**Why:** it lets the AS issue a token whose **audience** is exactly this MCP server. A token for server A then can't be replayed at server B.

### Scope selection

Clients **SHOULD** follow least privilege, using this priority for the **initial** request:

1. The **`scope`** from the initial `WWW-Authenticate` challenge, if present.
2. Otherwise **all scopes in `scopes_supported`** from PRM (omitting `scope` if that's undefined).

`scopes_supported` is intended to be the **minimal set for basic functionality**; more are requested later via step-up (section 10).

### State

Clients **SHOULD** use and verify the **`state`** parameter and **discard** results with a missing or mismatched state (CSRF protection).

### Recording the expected issuer

**Before redirecting** the browser, the client **MUST record the `issuer`** from the **validated** AS metadata, stored with the PKCE verifier (and
`state`). The next step depends on it.

---

## 9. Step 4: The response and the token

### Validate `iss`: the mix-up attack

A client talks to **many** ASes over its lifetime. An attacker controlling **one** of them can try to get the client to send it a code issued by
an **honest** AS (a **mix-up attack**). Without knowing which AS actually produced a response, the client may redeem an honest code at the
attacker's token endpoint.

**RFC 9207** fixes this: the AS includes **`iss`** in the authorization response, and the client checks it.

- ASes **SHOULD** include `iss` in authorization responses, **including error responses**. If they do, they **MUST** advertise
  `authorization_response_iss_parameter_supported: true`.
- Clients **MUST** validate **before transmitting the code to any token endpoint**:

| Metadata advertises `iss` support | `iss` in response | Client action |
| --- | --- | --- |
| `true` | present | Compare to recorded issuer (simple string comparison) |
| `true` | **absent** | **Reject** |
| `false` / absent | present | Compare to recorded issuer |
| `false` / absent | absent | Proceed |

Comparison is **exact**: no case folding, default-port removal, trailing-slash or percent-encoding normalization. On mismatch, even for an **error
response**, the client **MUST NOT** act on or display `error`, `error_description` or `error_uri`.

!!! note "PKCE doesn't stop mix-up"
    PKCE doesn't help here, because the client would hand the `code_verifier` to the attacker's token endpoint too. Resource indicators don't help
    either, since the attacker intercepts before the honest AS is involved. **`iss` validation is the mitigation.** A future revision is expected to
    make AS inclusion of `iss` a MUST.

### The token request

The client exchanges the code, sending the **`code_verifier`** and the same **`resource`**. It receives an **access token** and possibly a **refresh token**.

---

## 10. Step 5: Using and validating tokens

### Client side

- Send the token in the **`Authorization: Bearer <token>`** header.
- Include it on **every HTTP request** to the MCP server.
- **MUST NOT** put tokens in the **URI query string** (they leak into logs, history and referrers).
- **MUST NOT** send tokens other than those issued by **the MCP server's AS**.

```http
POST /mcp HTTP/1.1
Host: mcp.example.com
Authorization: Bearer <access-token>
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: read_file
```

### Server side

- **MUST validate** access tokens per OAuth 2.1 §5.2, **before processing** the request.
- **MUST validate the audience**: the token was issued **specifically for this server**; reject tokens that don't name it.
- **MUST only accept tokens valid for its own resources**; **MUST NOT accept or transit any other tokens**.
- Invalid or expired token → **`401`**.
- If the server calls upstream APIs, it uses a **separate** token from the upstream AS and **MUST NOT pass through** the client's token.

### Error statuses

| Status | Meaning |
| --- | --- |
| **401 Unauthorized** | Authorization required, or token invalid/expired |
| **403 Forbidden** | Invalid scopes or insufficient permissions |
| **400 Bad Request** | Malformed authorization request |

### Step-up: insufficient scope at runtime

A client holding a `files:read` token tries to write. The server **SHOULD** respond:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope",
                         scope="files:write",
                         resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource",
                         error_description="File write permission required for this operation"
```

Server guidance:

- `scope` lists the scopes **needed for this operation**. The server isn't required to include previously granted ones.
- Include **all** required scopes **in one challenge**. Challenging one at a time forces repeated authorization round-trips.
- Be **consistent** in strategy, and mind UX.
- **MUST account for scope hierarchies** (a broader scope implying narrower ones) when deciding whether a token suffices.

The client's **step-up flow**:

1. **Parse** the error.
2. **Compute the union** of previously requested scopes and the challenged scopes, so it **doesn't lose** permissions it already had.
3. **Re-authorize** with that set.
4. **Retry** the original request, **only a few times**, then treat it as a permanent failure.

Clients **SHOULD** set retry limits and track upgrade attempts per resource and operation. Clients acting **for a user SHOULD** attempt step-up;
**`client_credentials`** clients **MAY** step up or abort. **Scope accumulation is the client's responsibility**, which keeps servers stateless
about scope sets.

### Refresh tokens

Clients that want refresh tokens:

- **MUST** keep them **confidential** in transit and storage;
- **SHOULD** include `refresh_token` in `grant_types` metadata;
- **MAY** add **`offline_access`** to `scope` when the AS lists it in `scopes_supported`;
- **MUST NOT assume** refresh tokens will be issued.

MCP servers **SHOULD NOT** include `offline_access` in challenges or PRM `scopes_supported`, because refresh tokens are not a **resource**
requirement. For **public clients**, ASes **MUST rotate** refresh tokens.

---

## 11. Security considerations summary

| Threat | Required control |
| --- | --- |
| Token reused at another service | Client **MUST** send `resource`; server **MUST** validate audience |
| Token theft | Secure storage; ASes **SHOULD** issue **short-lived** access tokens; **rotate** refresh tokens for public clients |
| Eavesdropping | All AS endpoints **HTTPS**; redirect URIs **localhost or HTTPS** |
| Code interception or injection | **PKCE with S256**; refuse if unsupported |
| Mix-up | Record issuer; validate **`iss`** |
| Open redirection | Redirect URIs **registered**; AS **exact-match** validation; `state` checked |
| Metadata spoofing | **Issuer equality** check on AS metadata |
| CIMD abuse | AS SSRF protection; localhost warnings; trust policies |
| Confused deputy (proxy servers) | Per-client **consent** before forwarding to third-party ASes |
| Accepting other services' tokens | Validate before processing; **no passthrough** |

The threats themselves are covered in [Attacks & Mitigations](attacks-mitigations.md).

---

## 12. Authorization extensions

The core flow assumes an **interactive user**. Two official extensions (repository `ext-auth`) cover other cases. Like all extensions they are
**opt-in** and negotiated via capabilities.

### OAuth Client Credentials: `io.modelcontextprotocol/oauth-client-credentials`

**Machine-to-machine**, with no user present: background services, **CI/CD pipelines**, server-to-server integrations, daemons.

| Credential format | How | Note |
| --- | --- | --- |
| **JWT Bearer Assertion** (RFC 7523) | Client signs a JWT with its private key; `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer` | **Recommended**: short-lived, key never transmitted |
| **Client secret** | `grant_type=client_credentials` with `client_id` + `client_secret` | Long-lived; use a secrets manager, rotate, scope minimally |

Tokens are typically shorter-lived, so clients implement refresh logic. Servers validate the JWT signature (usually via JWKS) and check scopes.

### Enterprise-Managed Authorization: `io.modelcontextprotocol/enterprise-managed-authorization`

**Centralized** control through the organization's **identity provider** (Okta, Entra ID, corporate SSO) instead of every employee authorizing every
server.

1. The employee signs in to the MCP client via **enterprise SSO**; the client stores the **ID token** (or SAML assertion).
2. The client exchanges it at the IdP for an **ID-JAG** (Identity Assertion JWT Authorization Grant). **The IdP evaluates policy** here.
3. The client exchanges the ID-JAG at the **MCP authorization server** for an **MCP access token**, **without** redirecting the user to the MCP AS.
4. The client calls the MCP server with that token.

Benefits: **central policy** (group membership, roles, conditional access), **SSO** with no per-server prompts, **central revocation** effective across
all clients at once, and an **auditable** trail for compliance. MCP authorization servers validate ID-JAGs (signature via the IdP's JWKS, audience,
issuer, expiry), map claims to permissions, and use the **subject claim** as the stable identifier for account linking (falling back to email only for
pre-existing accounts).

| Scenario | Use |
| --- | --- |
| Standard interactive user | **Core** MCP authorization |
| Background service, CI/CD, server-to-server | **OAuth Client Credentials** |
| Employees accessing servers under corporate policy | **Enterprise-Managed Authorization** |

---

## 13. Recap

- Authorization is **optional**; **HTTP SHOULD** use it; **stdio SHOULD NOT** (credentials from the **environment**).
- MCP server = **OAuth resource server**; MCP client = **OAuth client**; AS may be separate.
- **Discovery:** `401` + `WWW-Authenticate: resource_metadata` → **PRM** (MUST implement; `authorization_servers`) → **AS metadata** (RFC 8414 or OIDC,
  priority order) → **issuer must equal** the URL's issuer.
- **Registration priority:** pre-registered → **CIMD** (HTTPS URL `client_id`, preferred, portable) → **DCR** (deprecated; set `application_type`) →
  ask the user. Key stored credentials **by issuer**.
- **Authorization request:** **PKCE S256 mandatory** (refuse if `code_challenge_methods_supported` absent); **`resource` = canonical MCP URI** in auth
  **and** token requests, always; scope from challenge, else `scopes_supported`; `state`; record issuer.
- **Response:** validate **`iss`** against the recorded issuer (mix-up defense); exact comparison.
- **Tokens:** `Authorization: Bearer` on **every** request, **never in the query string**. Servers **validate audience**, **never pass tokens through**,
  return **401** for invalid.
- **Step-up:** `403` + `insufficient_scope` + `scope`; client requests the **union** of old and new scopes; limit retries; servers honor hierarchies.
- **Refresh:** confidential; `offline_access` optional; not a server scope; rotate for public clients.
- Extensions: **Client Credentials** (M2M; JWT assertion recommended) and **Enterprise-Managed** (IdP policy via **ID-JAG**).
