---
title: "Lab 1: Speak MCP by Hand"
---

# Lab 1: Speak MCP by Hand

**Time:** 40 minutes · **Goal:** talk to an MCP server with nothing but JSON lines, and understand every
field that comes back.

SDKs hide the protocol, which is exactly why this lab avoids them on the client side. You will start a
real server as a subprocess, write JSON-RPC messages to its stdin, and read its replies from stdout.
This is what every host does underneath.

## 1. A minimal server

Create `weather_server.py`:

```python title="weather_server.py"
import logging
from mcp.server import MCPServer

logging.basicConfig(level=logging.INFO)  # logging writes to stderr
log = logging.getLogger("weather")

mcp = MCPServer("weather", version="1.0.0", instructions="Use get_forecast for any weather question.")

FAKE = {"oslo": "4°C, snow", "rome": "24°C, sunny", "seattle": "12°C, rain"}

@mcp.tool()
def get_forecast(city: str) -> str:
    """Get today's forecast for a city.

    Args:
        city: City name, e.g. "Oslo"
    """
    log.info("forecast requested for %s", city)
    return FAKE.get(city.lower(), f"No data for {city}")

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Three things to notice:

- **`instructions`**: free text the server gives the host, typically added to the model's system prompt.
- **The docstring and type hints**: the SDK turns them into the tool's `description` and `inputSchema`.
  A missing docstring means a model that can't tell what the tool is for.
- **`logging` goes to stderr**, which the stdio transport leaves free for logs.

## 2. The raw client

Create `send_raw.py`. It starts the server, then writes six requests and reads one reply line for each:

```python title="send_raw.py"
import json, subprocess, sys

META = {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientCapabilities": {},
    "io.modelcontextprotocol/clientInfo": {"name": "raw-lab", "version": "0.1"},
}

messages = [
    {"jsonrpc": "2.0", "id": 1, "method": "server/discover", "params": {"_meta": META}},
    {"jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {"_meta": META}},
    {"jsonrpc": "2.0", "id": 3, "method": "tools/call",
     "params": {"name": "get_forecast", "arguments": {"city": "Oslo"}, "_meta": META}},
    {"jsonrpc": "2.0", "id": 4, "method": "tools/call",
     "params": {"name": "no_such_tool", "arguments": {}, "_meta": META}},
    {"jsonrpc": "2.0", "id": 5, "method": "tools/list", "params": {}},
    {"jsonrpc": "2.0", "id": 6, "method": "tools/list",
     "params": {"_meta": {**META, "io.modelcontextprotocol/protocolVersion": "1900-01-01"}}},
]

proc = subprocess.Popen([sys.executable, "weather_server.py"], stdin=subprocess.PIPE,
                        stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
for m in messages:
    line = json.dumps(m)
    print(">>>", line)
    proc.stdin.write(line + "\n"); proc.stdin.flush()
    print("<<<", proc.stdout.readline().strip(), "\n")
proc.stdin.close()
proc.wait(timeout=5)
print("--- stderr ---")
print(proc.stderr.read())
```

!!! question "Predict"
    Before running it, write down your answers:

    1. Is there an `initialize` request before `server/discover`? Should there be?
    2. What will message 4 (unknown tool) return: a JSON-RPC `error` or a `result`?
    3. What will message 5 (no `_meta`) return?
    4. What will message 6 (version `1900-01-01`) return, and which error code?

Run it:

```bash
python send_raw.py
```

## 3. Reading the replies

### Message 1: `server/discover`

```json
{"jsonrpc":"2.0","id":1,"result":{
  "cacheScope":"private",
  "capabilities":{"prompts":{"listChanged":true},
                  "resources":{"listChanged":true,"subscribe":true},
                  "tools":{"listChanged":true}},
  "instructions":"Use get_forecast for any weather question.",
  "resultType":"complete",
  "supportedVersions":["2026-07-28"],
  "ttlMs":0,
  "_meta":{"io.modelcontextprotocol/serverInfo":{"name":"weather","version":"1.0.0"}}}}
```

There was **no handshake**. In `2026-07-28` the protocol is stateless: every request carries its own
version and capabilities in `params._meta`, so the client can call `tools/list` first if it likes.

`server/discover` is still worth calling because it answers several questions at once:

| Field | Meaning |
| --- | --- |
| `supportedVersions` | Versions the server speaks. The client picks one it also supports. |
| `capabilities` | Features the server offers, so the client knows what to ask for. |
| `instructions` | Usage guidance for the host or model. |
| `resultType: "complete"` | Required on **every** result in `2026-07-28`. The other value, `input_required`, appears in Lab 5. |
| `ttlMs` / `cacheScope` | Cache hints: how long the client may reuse this answer, and whether it may share it across users (`public`) or only within one user's context (`private`). |
| `_meta.serverInfo` | Who answered. It lives in `_meta` because there is no handshake to carry it. |

### Message 2: `tools/list`

```json
{"name":"get_forecast",
 "description":"Get today's forecast for a city.\n\n    Args:\n        city: City name, e.g. \"Oslo\"\n    ",
 "inputSchema":{"type":"object","properties":{"city":{"title":"City","type":"string"}},
                "required":["city"],"title":"get_forecastArguments"},
 "outputSchema":{"properties":{"result":{"title":"Result","type":"string"}},
                 "required":["result"],"title":"get_forecastOutput","type":"object"}}
```

The SDK built a JSON Schema from `city: str`. It also generated an `outputSchema`, because the function
declares a return type. A tool's result must be an object, so a plain `str` return gets wrapped as
`{"result": ...}`.

### Message 3: `tools/call`

```json
{"jsonrpc":"2.0","id":3,"result":{
  "content":[{"text":"4°C, snow","type":"text"}],
  "isError":false,
  "resultType":"complete",
  "structuredContent":{"result":"4°C, snow"}, ...}}
```

The result carries two views of the same data:

- **`content`** is unstructured and meant for the model.
- **`structuredContent`** matches the `outputSchema` and is meant for programs.

The spec says a tool that returns structured content SHOULD also return its serialised JSON as a text
block, for clients that ignore structured output. That is why the text appears in both.

### Message 4: unknown tool

```json
{"jsonrpc":"2.0","id":4,"result":{
  "content":[{"text":"Unknown tool: no_such_tool","type":"text"}],
  "isError":true,"resultType":"complete", ...}}
```

!!! warning "SDK behaviour is not the specification"
    The specification lists "unknown tool" among the **protocol errors**, which should be a JSON-RPC
    `error` with code `-32602` (Invalid params). The Python SDK returns a **tool execution error** instead:
    a normal result with `isError: true`.

    For the exam, go with the spec: an unknown tool name is a protocol error, `-32602`. Keep `isError`
    for failures *inside* a tool that the model could fix, such as a bad date format or an API rate
    limit.

### Message 5: missing `_meta`

```json
{"jsonrpc":"2.0","id":5,"error":{"code":-32602,
  "message":"params._meta must be an object carrying the required 'io.modelcontextprotocol/protocolVersion' and 'io.modelcontextprotocol/clientCapabilities' envelope keys"}}
```

Without a handshake, the server cannot learn the client's version from anywhere else, so these two
keys are **required on every request**. `clientInfo` is only a SHOULD, which is why the error message
doesn't mention it.

### Message 6: unsupported version

```json
{"jsonrpc":"2.0","id":6,"error":{"code":-32022,"message":"Unsupported protocol version",
  "data":{"supported":["2026-07-28"],"requested":"1900-01-01"}}}
```

`-32022 UnsupportedProtocolVersion` includes `data.supported`, so a client can pick a version from the
list and retry straight away, with no extra discovery round.

### stderr

```text
INFO:weather:forecast requested for Oslo
INFO:mcp.server.mcpserver.server:Tool 'no_such_tool' failed: 'Unknown tool: no_such_tool'
```

The logs never touched the protocol stream.

## 4. The stdout trap

What happens if a server writes something that isn't JSON-RPC to stdout? The SDK protects you: while
it serves stdio it redirects `print()` to stderr. To see the real failure, use a tiny hand-written
server without the SDK.

```python title="handrolled_server.py"
"""Lab 1: a deliberately tiny stdio 'server' written without the SDK, with a stdout bug."""
import json
import sys

for raw in sys.stdin:
    msg = json.loads(raw)
    print(f"DEBUG got {msg['method']}")                     # BUG: this goes to stdout
    # print(f"DEBUG got {msg['method']}", file=sys.stderr)  # FIX: log to stderr
    reply = {"jsonrpc": "2.0", "id": msg["id"], "result": {"resultType": "complete", "tools": []}}
    sys.stdout.write(json.dumps(reply) + "\n")
    sys.stdout.flush()
```

```python title="send_handrolled.py"
import json, subprocess, sys

p = subprocess.Popen([sys.executable, "handrolled_server.py"], stdin=subprocess.PIPE,
                     stdout=subprocess.PIPE, text=True)
req = {"jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {}}
out, _ = p.communicate(json.dumps(req) + "\n", timeout=5)
for line in out.splitlines():
    try:
        json.loads(line)
        print("valid JSON-RPC ->", line)
    except json.JSONDecodeError:
        print("NOT JSON       ->", repr(line))
```

```text
NOT JSON       -> 'DEBUG got tools/list'
valid JSON-RPC -> {"jsonrpc": "2.0", "id": 1, "result": {"resultType": "complete", "tools": []}}
```

A real client reading line by line would try to parse `DEBUG got tools/list` as a message and fail.
Depending on the client, that means a dropped connection or a "server disconnected" error in the host.

The stdio rules in the spec exist for exactly this reason:

- The server **MUST NOT** write anything to stdout that is not a valid MCP message.
- The client **MUST NOT** write anything to the server's stdin that is not a valid MCP message.
- Messages are newline-delimited and **MUST NOT** contain embedded newlines.
- The server **MAY** write any logging to stderr, and the client should not treat stderr output as an
  error.

!!! tip "Try it"
    Swap the two `print` lines in `handrolled_server.py` so the debug output goes to stderr, then run
    `send_handrolled.py` again. Only the valid line remains.

## 5. Bonus: an old client meets a new server

A `2025-11-25` client opens with `initialize`. What does a `2026-07-28` SDK server do with that?

```python title="legacy_probe.py"
import json, subprocess, sys
proc = subprocess.Popen([sys.executable, "weather_server.py"], stdin=subprocess.PIPE,
                        stdout=subprocess.PIPE, stderr=subprocess.DEVNULL, text=True)
legacy = {"jsonrpc": "2.0", "id": 1, "method": "initialize",
          "params": {"protocolVersion": "2025-11-25", "capabilities": {},
                     "clientInfo": {"name": "old-client", "version": "1.0"}}}
print(">>>", json.dumps(legacy)); proc.stdin.write(json.dumps(legacy) + "\n"); proc.stdin.flush()
print("<<<", proc.stdout.readline().strip())
proc.stdin.close(); proc.wait(timeout=5)
```

```text
<<< {"jsonrpc":"2.0","id":1,"result":{"capabilities":{...},
     "instructions":"Use get_forecast for any weather question.",
     "protocolVersion":"2025-11-25","serverInfo":{"name":"weather","version":"1.0.0"}}}
```

The server answers in the old style: an `InitializeResult`, no `resultType`, and `serverInfo` at the top
level rather than in `_meta`. The SDK server is **dual-era**, which the spec allows: a server **MAY**
implement both behaviours. This is how the ecosystem migrates without a flag day.

The spec's compatibility matrix gives the rest of the picture:

| Client | Server | Outcome |
| --- | --- | --- |
| Legacy | Dual-era | Works; the server answers `initialize` (what you just saw). |
| Legacy | Modern-only | Fails: `initialize` is rejected. The error SHOULD name the supported versions, because a legacy client has no way to move forward. |
| Dual-era | Legacy | Works. On stdio the client probes with `server/discover`, gets a non-modern error or no reply, and falls back to `initialize`. |
| Modern-only | Legacy | Fails. Probing first makes the failure deterministic. |

On stdio, a dual-era client **MUST NOT** base the fallback on one specific error code, because legacy
servers answer unknown methods in different ways. A *recognised modern* error such as `-32022` means
"modern server, wrong version": the client retries with a supported version and does not fall back.

## What this showed you

| You saw | Study chapter |
| --- | --- |
| No handshake; `_meta` on every request | [Versioning & Discovery](../domain-2/versioning-discovery.md) |
| JSON-RPC shape, ids, `resultType` | [Messages](../domain-2/messages.md) |
| `inputSchema`, `outputSchema`, `structuredContent` | [Tools & the Invocation Lifecycle](../domain-3/tools.md) |
| `-32602` vs `isError`, `-32022` | [Error Handling](../domain-3/error-handling.md) |
| stdout is the channel, stderr is for logs | [Transports](../domain-2/transports.md) |
| Cache hints on discovery results | [Caching, Pagination & Completion](../domain-3/utilities.md) |
