---
title: "Lab 4: Streamable HTTP"
---

# Lab 4: Streamable HTTP

**Time:** 50 minutes · **Goal:** serve the Lab 2 server over HTTP, drive it with plain `curl`, and set off
every check the transport defines: header mirroring, version errors, Origin validation, and the removed
session-era methods.

## 1. Why a second transport

stdio is ideal when the host launches the server as a local process: there is no network, no port and
no authentication. It cannot serve a remote team, a SaaS integration, or a server that runs once for
thousands of users.

**Streamable HTTP** is the standard transport for remote servers. In `2026-07-28` it is simple:

- **One endpoint** (here `/mcp`) and **one method, POST**. Every JSON-RPC message is its own POST.
- The response is **either** a single JSON object (`application/json`) **or** an SSE stream
  (`text/event-stream`) for that one request, e.g. to send progress before the result.
- **No sessions**: no `Mcp-Session-Id`, no GET stream, no resumability.
- Key body fields are **mirrored into HTTP headers**, so load balancers and gateways can route requests
  without parsing JSON.

## 2. Start the server

Create `notes_http.py` next to `notes_server.py`:

```python title="notes_http.py"
"""Lab 4: serve the notes server over Streamable HTTP."""
from notes_server import mcp

if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="127.0.0.1", port=8799)   # endpoint: http://127.0.0.1:8799/mcp
```

Run it in its own terminal and leave it running:

```bash
python notes_http.py
```

If port 8799 is already taken, change it here and in the commands below.

!!! info "Bind to 127.0.0.1"
    The spec says local servers **SHOULD** bind only to localhost, not `0.0.0.0`. The SDK defaults to
    `127.0.0.1` for this reason.

## 3. Set up shell variables

In a second terminal, define the parts that repeat:

```bash
U=http://127.0.0.1:8799/mcp
META='"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{},"io.modelcontextprotocol/clientInfo":{"name":"curl","version":"1"}}'
H=(-H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -H "MCP-Protocol-Version: 2026-07-28")
```

The client **MUST** send an `Accept` header listing both content types, because the server may choose
either.

## 4. The happy path

### Discovery

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: server/discover" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"server/discover\",\"params\":{$META}}"
```

```http
HTTP/1.1 200 OK
content-type: application/json

{"jsonrpc":"2.0","id":1,"result":{"cacheScope":"private","capabilities":{...},
 "instructions":"Search notes before creating a new one to avoid duplicates.",
 "resultType":"complete","supportedVersions":["2026-07-28"],"ttlMs":0,...}}
```

### A tool call

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/call" -H "Mcp-Name: search_notes" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/call\",\"params\":{\"name\":\"search_notes\",\"arguments\":{\"query\":\"mcp\"},$META}}"
```

```http
HTTP/1.1 200 OK
content-type: application/json

{"jsonrpc":"2.0","id":2,"result":{"content":[...],"isError":false,"resultType":"complete",
 "structuredContent":{"result":[{"id":"n1","title":"MCP transports",...}]},...}}
```

The standard headers you just sent:

| Header | Mirrors | Required on |
| --- | --- | --- |
| `MCP-Protocol-Version` | `params._meta["io.modelcontextprotocol/protocolVersion"]` | Every POST |
| `Mcp-Method` | `method` | All requests |
| `Mcp-Name` | `params.name` or `params.uri` | `tools/call`, `resources/read`, `prompts/get` |

Tools can also declare parameters to mirror as `Mcp-Param-{Name}` headers, using the `x-mcp-header`
schema extension. A gateway can then, for example, route by a `region` argument.

## 5. Breaking it on purpose

!!! question "Predict"
    For each request below, predict the HTTP status **and** the JSON-RPC error code before you run it.

### 5.1 A header that disagrees with the body

The header says `tools/list`, but the body says `tools/call`:

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/list" -H "Mcp-Name: search_notes" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"tools/call\",\"params\":{\"name\":\"search_notes\",\"arguments\":{\"query\":\"mcp\"},$META}}"
```

```http
HTTP/1.1 400 Bad Request

{"jsonrpc":"2.0","id":3,"error":{"code":-32020,"message":"mcp-method header does not match the request body's method"}}
```

**Why this rule exists.** Picture a gateway that allows `tools/list` for everyone but restricts
`tools/call`. It checks the header. The MCP server executes the body. If they could disagree, an
attacker would put `tools/list` in the header and `tools/call` in the body and walk straight past the
policy. So servers **MUST** reject any mismatch with `400` and `-32020 HeaderMismatch`.

### 5.2 A missing required header

The same thing without `Mcp-Name`:

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/call" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":4,\"method\":\"tools/call\",\"params\":{\"name\":\"search_notes\",\"arguments\":{\"query\":\"x\"},$META}}"
```

```http
HTTP/1.1 400 Bad Request

{"jsonrpc":"2.0","id":4,"error":{"code":-32020,"message":"mcp-name header does not match the request body's 'name' parameter"}}
```

The spec counts a **missing** required header as a validation failure too, with the same code, `-32020`.
The `Mcp-Name` mismatch (header `create_note`, body `search_notes`) gives the same result.

### 5.3 A version header that disagrees with `_meta`

```bash
curl -s $U -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2026-07-28" -H "Mcp-Method: tools/list" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/list","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2025-11-25","io.modelcontextprotocol/clientCapabilities":{}}}}'
```

```json
{"jsonrpc":"2.0","id":5,"error":{"code":-32020,"message":"mcp-protocol-version header does not match the request envelope's protocol version"}}
```

### 5.4 A version the server doesn't support

Both the header and `_meta` say `2027-01-01`:

```bash
curl -si $U -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2027-01-01" -H "Mcp-Method: tools/list" \
  -d '{"jsonrpc":"2.0","id":6,"method":"tools/list","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2027-01-01","io.modelcontextprotocol/clientCapabilities":{}}}}'
```

```http
HTTP/1.1 400 Bad Request

{"jsonrpc":"2.0","id":6,"error":{"code":-32022,"message":"Unsupported protocol version",
 "data":{"supported":["2026-07-28"],"requested":"2027-01-01"}}}
```

Compare this with 5.3. There the request **contradicted itself**, so it got `-32020`. Here it is
consistent but **unsupported**, so it gets `-32022`, and `data.supported` tells the client what to use
instead.

### 5.5 A method the server doesn't implement

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/frobnicate" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":7,\"method\":\"tools/frobnicate\",\"params\":{$META}}"
```

```http
HTTP/1.1 404 Not Found

{"jsonrpc":"2.0","id":7,"error":{"code":-32601,"message":"Method not found","data":"tools/frobnicate"}}
```

The spec requires `404` **with** a JSON-RPC `-32601` body. The body is what tells this apart from the
bare `404` of an old HTTP+SSE server that has no modern endpoint.

### 5.6 A hostile `Origin`

A web page on `evil.example` tries to reach your local server through the browser:

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: tools/list" -H "Origin: https://evil.example" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":8,\"method\":\"tools/list\",\"params\":{$META}}"
```

```http
HTTP/1.1 403 Forbidden

Invalid Origin header
```

This is the **DNS rebinding** defence. A malicious site points its own hostname at `127.0.0.1`. The
browser then treats requests to your local server as same-site, and the page can drive your tools. The
browser always sends the page's real `Origin`, so the server **MUST** validate it. If it is present and
not allowed, the answer **MUST** be `403`. Lab 6 adds the matching `Host` header check.

### 5.7 Session-era traffic

An older client may still try a GET stream or a DELETE to end its session:

```bash
curl -si $U -H "Accept: text/event-stream" -H "MCP-Protocol-Version: 2026-07-28"
curl -si -X DELETE $U -H "MCP-Protocol-Version: 2026-07-28"
```

```http
HTTP/1.1 405 Method Not Allowed
allow: POST
```

Both give the same answer. The spec says a server that only speaks this revision **SHOULD** answer GET
and DELETE with `405`, **ignore** any `Mcp-Session-Id` header, and ignore `Last-Event-ID`.

### 5.8 A notification

```bash
curl -si $U "${H[@]}" -H "Mcp-Method: notifications/cancelled" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"notifications/cancelled\",\"params\":{\"requestId\":99,$META}}"
```

```http
HTTP/1.1 202 Accepted
content-length: 0
```

A notification has no `id`, so there is no response to send back. A server that accepts one **MUST**
return `202 Accepted` with an empty body.

!!! warning "Trap: cancellation on HTTP"
    This request only demonstrates the `202` mechanics. On Streamable HTTP, clients **don't** send
    `notifications/cancelled`. **Closing the response stream is the cancellation**, and the server MUST
    treat it that way. `notifications/cancelled` is only used on stdio, where there is no per-request
    connection to close.

## 6. Legacy traffic on the same endpoint

Send an old-style `initialize` with no version header:

```bash
curl -si $U -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":9,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"old","version":"1"}}}'
```

```http
HTTP/1.1 200 OK
content-type: text/event-stream
mcp-session-id: 2e7befc699884d22b0b09017d6cf99c0

event: message
data: {"jsonrpc":"2.0","id":9,"result":{...,"protocolVersion":"2025-11-25","serverInfo":{"name":"notes","version":"1.0.0"}}}
```

The SDK server is dual-era here as well. It detects a legacy client and falls back to the
`2025-11-25` transport, including a session id. A server that supports **only** `2026-07-28` would
reject this request instead.

The reverse also happens. Earlier, a request with a *missing* or *old* `MCP-Protocol-Version` header
fell into this legacy path and got `-32600 "Bad Request: Missing session ID"` rather than a modern
error. That is how a dual-era client tells the two eras apart: a **recognised modern error** in a
`400` body means "modern server, fix your request". Anything else means "legacy server, fall back to
`initialize`".

## 7. The same thing, with the SDK

The high-level `Client` from Lab 3 accepts a URL too:

```python title="http_client.py"
"""Lab 4: the same Client, pointed at a URL instead of a subprocess."""
import anyio
from mcp import Client


async def main() -> None:
    async with Client("http://127.0.0.1:8799/mcp") as client:
        print("protocol:", client.protocol_version)
        tools = await client.list_tools()
        print("tools   :", [t.name for t in tools.tools])
        result = await client.call_tool("search_notes", {"query": "mrtr"})
        print("call    :", result.structured_content)


anyio.run(main)
```

```text
protocol: 2026-07-28
tools   : ['search_notes', 'create_note', 'start_draft', 'append_to_draft']
call    : {'result': [{'id': 'n2', 'title': 'MRTR', 'body': 'Servers return input_required; clients retry with a new id.'}]}
```

The application code is identical to the stdio version; only the connection target changed. Transport
independence is a design goal: the data layer (JSON-RPC messages and primitives) knows nothing about
the transport layer.

## 8. Summary table

| Request | HTTP | JSON-RPC |
| --- | --- | --- |
| Valid request | `200` (JSON or SSE) | result |
| Valid notification | `202`, empty body | none |
| Header ≠ body, or a required header missing | `400` | `-32020` HeaderMismatch |
| Unsupported version | `400` | `-32022` UnsupportedProtocolVersion, with `data.supported` |
| Unknown method | `404` | `-32601` Method not found |
| Invalid `Origin` | `403` | optional, with no `id` |
| GET / DELETE on the endpoint | `405` | none |
| Missing or invalid token (Lab 6) | `401` + `WWW-Authenticate` | none |
| Token without the needed scope (Lab 6) | `403` + `insufficient_scope` | none |

## What this showed you

| You saw | Study chapter |
| --- | --- |
| POST-only, JSON vs SSE, 202, 405 | [Transports](../domain-2/transports.md) |
| Header mirroring and `-32020` | [Transports](../domain-2/transports.md), [Error Handling](../domain-3/error-handling.md) |
| `-32022` and era detection | [Versioning & Discovery](../domain-2/versioning-discovery.md) |
| Cancellation by closing the stream | [Subscriptions, Progress & Cancellation](../domain-3/subscriptions-progress-cancellation.md) |
| Origin validation, localhost binding | [Trust Boundaries](../domain-4/trust-boundaries.md), [Attacks & Mitigations](../domain-4/attacks-mitigations.md) |
