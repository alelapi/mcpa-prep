---
title: "Lab 6: Security Hardening"
---

# Lab 6: Security Hardening

**Time:** 60 minutes · **Goal:** turn an open HTTP server into a protected OAuth **resource server**.
Then attack it the ways the security chapters describe: no token, a forged token, a token meant for a
different API, missing scopes, path traversal and DNS rebinding. Watch each attack fail, and find out
which single setting stops each one.

Background reading: [Authorization with OAuth 2.1](../domain-4/authorization.md) and
[Attacks & Mitigations](../domain-4/attacks-mitigations.md).

## 1. What the server is responsible for

In MCP authorization there are three parties:

| Role | In this lab |
| --- | --- |
| **Client** (the host's MCP client) | `curl`, playing a host that already has a token |
| **Authorization server (AS)**, which issues tokens | Pretend: `https://auth.example.com`, simulated by a dictionary |
| **Resource server**, which accepts tokens | **Our MCP server** |

The MCP server never sees a password or a login form. Its job is narrow, and every part of it matters:

1. **Say where to get a token**: a `401` challenge that points to Protected Resource Metadata.
2. **Validate every token before processing the request**: is it authentic, unexpired, **issued for
   this server**, and does it carry the needed **scope**?
3. **Refuse to be a proxy for other tokens.** No token passthrough.

The rest of the lab adds defence in depth for inputs and for local exposure.

## 2. The server

Create `secure_server.py`:

```python title="secure_server.py"
"""Lab 6: a protected MCP server (resource-server role) with hardened inputs."""
import logging
import sys
from pathlib import Path

from mcp.server import MCPServer
from mcp.server.auth.middleware.auth_context import get_access_token
from mcp.server.auth.provider import AccessToken
from mcp.server.auth.settings import AuthSettings
from mcp.server.mcpserver.exceptions import ResourceNotFoundError, ToolError
from mcp.server.transport_security import TransportSecuritySettings

logging.basicConfig(stream=sys.stderr, level=logging.WARNING)

SERVER_URL = "http://127.0.0.1:8798/mcp"
AUTH_SERVER = "https://auth.example.com"
DOCS = Path(__file__).parent / "docs"

# Stand-in for an authorization server. In production you would validate a JWT
# (signature, iss, exp, aud) or call the AS's introspection endpoint (RFC 7662).
FAKE_TOKENS = {
    "tok-reader":  {"client_id": "host-app", "sub": "ada", "scopes": ["docs:read"], "aud": SERVER_URL},
    "tok-writer":  {"client_id": "host-app", "sub": "ada", "scopes": ["docs:read", "docs:write"], "aud": SERVER_URL},
    "tok-profile": {"client_id": "host-app", "sub": "ada", "scopes": ["profile"], "aud": SERVER_URL},
    "tok-other-api": {"client_id": "host-app", "sub": "ada", "scopes": ["docs:read"], "aud": "https://other-api.example.com"},
}


class DemoTokenVerifier:
    async def verify_token(self, token: str) -> AccessToken | None:
        info = FAKE_TOKENS.get(token)
        if info is None:
            return None                                  # unknown -> 401
        return AccessToken(
            token=token,
            client_id=info["client_id"],
            subject=info["sub"],
            scopes=info["scopes"],
            resource=info["aud"],                        # who the token was issued FOR
            claims={"iss": AUTH_SERVER},
        )


mcp = MCPServer(
    "secure-docs",
    version="1.0.0",
    token_verifier=DemoTokenVerifier(),
    auth=AuthSettings(
        issuer_url=AUTH_SERVER,
        resource_server_url=SERVER_URL,
        required_scopes=["docs:read"],                   # every request needs at least this
        validate_token_resource=True,                    # reject tokens minted for another resource
    ),
)


@mcp.resource("docs://{+path}")
def read_doc(path: str) -> str:
    """Read a document under the docs directory."""
    target = (DOCS / path).resolve()
    if not target.is_relative_to(DOCS.resolve()):        # defence in depth behind ResourceSecurity
        raise ResourceNotFoundError(f"{path} not found")
    if not target.is_file():
        raise ResourceNotFoundError(f"{path} not found")
    return target.read_text()


@mcp.tool()
def delete_doc(path: str) -> str:
    """Delete a document. Requires the docs:write scope."""
    token = get_access_token()
    if token is None or "docs:write" not in token.scopes:
        raise ToolError("This action needs the docs:write scope.")
    return f"(pretend) deleted {path} on behalf of {token.subject}"


if __name__ == "__main__":
    DOCS.mkdir(exist_ok=True)
    (DOCS / "welcome.md").write_text("# Welcome\nOnly for authorised readers.\n")
    mcp.run(
        transport="streamable-http",
        host="127.0.0.1",
        port=8798,
        transport_security=TransportSecuritySettings(
            allowed_hosts=["127.0.0.1:8798"],
            allowed_origins=["http://127.0.0.1:8798"],
        ),
    )
```

```bash
python secure_server.py
```

!!! danger "The token table is a teaching prop"
    Real token verification checks a signature against the AS's published keys, and checks `iss`, `exp`
    and `aud`. Or it calls the AS's introspection endpoint. Never accept tokens by looking them up in a
    hard-coded table.

In a second terminal, set up the shared variables:

```bash
B=http://127.0.0.1:8798
U=$B/mcp
META='"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{}}'
H=(-H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -H "MCP-Protocol-Version: 2026-07-28")
LIST="{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"tools/list\",\"params\":{$META}}"
```

## 3. Attack 1: no token

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/list" -d "$LIST"
```

```http
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer error="invalid_token", error_description="Authentication required",
                  resource_metadata="http://127.0.0.1:8798/.well-known/oauth-protected-resource/mcp"

{"error": "invalid_token", "error_description": "Authentication required"}
```

This `401` is where the client's authorization flow starts, not a dead end. The `resource_metadata`
parameter tells a client that knows nothing about this server where to go next. Follow it:

```bash
curl -s $B/.well-known/oauth-protected-resource/mcp
```

```json
{"resource":"http://127.0.0.1:8798/mcp",
 "authorization_servers":["https://auth.example.com"],
 "scopes_supported":["docs:read"],
 "bearer_methods_supported":["header"]}
```

This is **Protected Resource Metadata (RFC 9728)**. From here a real client would:

1. fetch the AS's metadata from `https://auth.example.com`,
2. identify itself, preferably with a Client ID Metadata Document,
3. run the authorization code flow with **PKCE (S256)** and **`resource=http://127.0.0.1:8798/mcp`**
   (RFC 8707),
4. check `iss` in the response (RFC 9207),
5. retry with `Authorization: Bearer <token>`.

!!! question "Check yourself"
    Why does the client send `resource=` when it asks for a token?

    ??? success "Answer"
        So the AS issues a token whose **audience** is this specific MCP server. A token restricted to one
        audience is useless anywhere else, and that is what makes Attack 3 fail.

## 4. Attack 2: a forged token

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/list" -H "Authorization: Bearer tok-forged" -d "$LIST"
```

```http
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer error="invalid_token", error_description="Authentication required", resource_metadata="..."
```

The verifier returned `None`. An invalid or expired token gets a `401`, and the request never reaches
MCP processing: validation happens **before** the request is processed.

## 5. Attack 3: a real token for a different API

`tok-other-api` is completely genuine. The user really logged in, and the AS really issued it. But it
was issued **for `https://other-api.example.com`**.

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/list" -H "Authorization: Bearer tok-other-api" -d "$LIST"
```

```http
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer error="invalid_token", error_description="Authentication required", resource_metadata="..."
```

`validate_token_resource=True` compared the token's resource (its audience) with `resource_server_url`
and refused.

!!! tip "Try it: turn the check off"
    Set `validate_token_resource=False`, restart, and repeat the request:

    ```http
    HTTP/1.1 200 OK
    {"jsonrpc":"2.0","id":1,"result":{"tools":[{"name":"delete_doc",...}]}}
    ```

    The server now accepts a token that was never meant for it. That is the path to **token
    passthrough** and **confused deputy** attacks. Any service that holds one of the user's tokens (a
    compromised or malicious MCP server, say) can replay it here, and this server will act on it.
    The spec's rule is absolute: servers **MUST** validate that tokens were issued for them and
    **MUST NOT** accept any other token.

    Remember to set it back to `True`.

## 6. Attack 4: a valid token without the required scope

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/list" -H "Authorization: Bearer tok-profile" -d "$LIST"
```

```http
HTTP/1.1 403 Forbidden
www-authenticate: Bearer error="insufficient_scope", error_description="Required scope: docs:read",
                  resource_metadata="http://127.0.0.1:8798/.well-known/oauth-protected-resource/mcp"

{"error": "insufficient_scope", "error_description": "Required scope: docs:read"}
```

Compare the statuses:

- **`401`** means "I don't know who you are". The token is missing, invalid, expired or for someone else.
- **`403` `insufficient_scope`** means "I know who you are, but this token doesn't allow that". The
  client should run the **step-up** flow: re-authorize with the union of the scopes it already had and
  the challenged scopes, then retry, a limited number of times.

## 7. Scopes per operation

```bash
# reader token
curl -s $U "${H[@]}" -H "Mcp-Method: tools/call" -H "Mcp-Name: delete_doc" -H "Authorization: Bearer tok-reader" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/call\",\"params\":{\"name\":\"delete_doc\",\"arguments\":{\"path\":\"welcome.md\"},$META}}"
# writer token
curl -s $U "${H[@]}" -H "Mcp-Method: tools/call" -H "Mcp-Name: delete_doc" -H "Authorization: Bearer tok-writer" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"tools/call\",\"params\":{\"name\":\"delete_doc\",\"arguments\":{\"path\":\"welcome.md\"},$META}}"
```

```json
{"jsonrpc":"2.0","id":2,"result":{"content":[{"text":"Error executing tool delete_doc: This action needs the docs:write scope.","type":"text"}],"isError":true,...}}
{"jsonrpc":"2.0","id":3,"result":{"content":[{"text":"(pretend) deleted welcome.md on behalf of ada","type":"text"}],"isError":false,...}}
```

`get_access_token()` gives the tool the verified token, including **who** (`subject`) and **what they
may do** (`scopes`). The tool can therefore make per-operation decisions and act on behalf of a specific
user. That is also how you bind state handles to their owner (Lab 2).

!!! warning "Simplified on purpose"
    Returning `isError` tells the *model* that it isn't allowed, but the *client* gets no signal to
    upgrade the token. The spec's step-up mechanism is the HTTP-level `403` with
    `WWW-Authenticate: Bearer error="insufficient_scope", scope="docs:write"`, which a client can act on
    automatically. For the exam, remember that version.

## 8. Attack 5: path traversal through a resource template

```bash
for uri in "docs://welcome.md" "docs://../secure_server.py" "docs://%2E%2E/secure_server.py" "docs:///etc/passwd"; do
  curl -s $U "${H[@]}" -H "Mcp-Method: resources/read" -H "Mcp-Name: $uri" -H "Authorization: Bearer tok-reader" \
    -d "{\"jsonrpc\":\"2.0\",\"id\":4,\"method\":\"resources/read\",\"params\":{\"uri\":\"$uri\",$META}}"; echo
done
```

```json
{"jsonrpc":"2.0","id":4,"result":{"contents":[{"mimeType":"text/plain","text":"# Welcome\nOnly for authorised readers.\n","uri":"docs://welcome.md"}],...}}
{"jsonrpc":"2.0","id":4,"error":{"code":-32602,"message":"Unknown resource: docs://../secure_server.py","data":{"uri":"docs://../secure_server.py"}}}
{"jsonrpc":"2.0","id":4,"error":{"code":-32602,"message":"Unknown resource: docs://%2E%2E/secure_server.py","data":{"uri":"docs://%2E%2E/secure_server.py"}}}
{"jsonrpc":"2.0","id":4,"error":{"code":-32602,"message":"Unknown resource: docs:///etc/passwd","data":{"uri":"docs:///etc/passwd"}}}
```

The `{+path}` template accepts slashes, which makes it an obvious traversal target. Two layers stopped
the attack:

1. **The SDK's default `ResourceSecurity`** checks values pulled out of templates. After decoding, it
   rejects `..` path segments (including the encoded `%2E%2E`), absolute paths and NUL bytes. None of the
   three attacks reached `read_doc`.
2. **The handler's own `is_relative_to` check** is a second line of defence. It still protects the server
   if someone later opts the parameter out of the SDK policy.

The attacker gets the same `-32602` as for a file that doesn't exist, and no hint about why.

Resource URIs, tool arguments and elicitation answers all come from outside the server: the model or
the client. Validate them as untrusted input.

## 9. Attack 6: DNS rebinding

```bash
curl -si $U "${H[@]}" -H "Host: attacker.example:8798" -H "Mcp-Method: tools/list" \
  -H "Authorization: Bearer tok-reader" -d "$LIST"
```

```http
HTTP/1.1 421 Misdirected Request

Invalid Host header
```

In a DNS rebinding attack, a page on `attacker.example` gets its hostname to resolve to `127.0.0.1`.
The browser then sends requests to your local server, but the `Host` header still says
`attacker.example`. `TransportSecuritySettings` allows only the listed hosts and origins:

- a wrong **`Host`** gets `421` (this SDK's response),
- a wrong **`Origin`** gets `403`, as the spec requires (seen in Lab 4).

Binding to `127.0.0.1` rather than `0.0.0.0` also keeps the server off the local network.

## 10. Summary: attack → control → result

| Attack | Control | What you saw |
| --- | --- | --- |
| No token | Bearer auth, 401 challenge with `resource_metadata` | `401` + PRM discovery |
| Forged or expired token | Token verification before processing | `401` |
| Token issued for another API (passthrough, confused deputy) | **Audience validation** (`validate_token_resource`) | `401`, but `200` when switched off |
| Token lacking scope | `required_scopes` | `403 insufficient_scope` |
| Operation beyond the user's rights | Per-tool check of `scopes` and `subject` | Denied / allowed per token |
| Path traversal in resource URIs | `ResourceSecurity` + handler checks | `-32602` |
| Tampered or replayed `requestState` (Lab 5) | Sealed, bound state | `-32602` |
| DNS rebinding from a browser | Host/Origin allow-lists, localhost binding | `421` / `403` |
| Guessable state handles (Lab 2) | Random, user-bound handles | `isError` for unknown handles |

## 11. What the lab did not cover

These still matter for the exam. See the linked chapters:

- **SSRF** during OAuth metadata discovery: a client fetching attacker-supplied URLs.
  See [Attacks & Mitigations](../domain-4/attacks-mitigations.md#ssrf).
- **Tool poisoning and rug pulls**: malicious descriptions, and definitions that change after approval.
- **Local server compromise**: a host running an untrusted `command` from a configuration file.
- **Audit logging and tracing**: see [Auditability](../domain-4/auditability.md).

## What this showed you

| You saw | Study chapter |
| --- | --- |
| 401 challenge, PRM, audience, scopes, step-up | [Authorization with OAuth 2.1](../domain-4/authorization.md) |
| Token passthrough, confused deputy, DNS rebinding | [Attacks & Mitigations](../domain-4/attacks-mitigations.md) |
| Where the trust boundaries are | [Trust Boundaries](../domain-4/trust-boundaries.md) |
| Per-user, per-operation permission | [Permissions & Consent](../domain-4/permissions-consent.md) |
