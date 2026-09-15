---
title: "Lab 0: Environment Setup"
---

# Lab 0: Environment Setup

**Time:** 10 minutes · **Goal:** a folder with a virtual environment and the MCP Python SDK installed.

## 1. Requirements

| Tool | Version used in these labs | Check with |
| --- | --- | --- |
| Python | 3.12 (3.10+ works) | `python3 --version` |
| `curl` | any recent | `curl --version` |
| A terminal | macOS, Linux or WSL | |

There is no need for an LLM API key. The labs play the part of the model themselves, so that you can
see exactly what a host would send and receive.

## 2. Create the working folder

```bash
mkdir mcp-lab && cd mcp-lab
python3 -m venv .venv
source .venv/bin/activate
```

Every later command assumes you are inside `mcp-lab` with the virtual environment active.

## 3. Install the SDK

```bash
pip install "mcp==2.2.0"
```

The `mcp` package pulls in everything the labs need:

- `mcp-types`: the Pydantic models for every protocol message
- `starlette` and `uvicorn`: the HTTP server used for Streamable HTTP
- `httpx`: the HTTP client
- `pydantic`, `jsonschema` and `cryptography`: validation, and the sealing of `requestState`

Check the install:

```bash
python -c "import mcp, mcp_types.version as v; print(v.MODERN_PROTOCOL_VERSIONS)"
```

You should see a collection containing `2026-07-28`.

!!! info "Why pin the version"
    MCP changed a lot between `2025-11-25` and `2026-07-28`. Sessions disappeared, MRTR arrived and the
    envelope moved into `_meta`. SDK releases track those changes. Pinning `2.2.0` keeps your output
    identical to the output shown in the labs.

## 4. Two SDK habits to learn now

**The Python API uses snake_case; the wire uses camelCase.**
In Python the attributes are `tool.input_schema`, `result.is_error`, `result.structured_content`. On the
wire the same fields are `inputSchema`, `isError`, `structuredContent`. To see the wire form of any
model, dump it with aliases:

```python
import json

def wire(model) -> str:
    return json.dumps(model.model_dump(by_alias=True, exclude_none=True, mode="json"))
```

**stdio servers log to stderr.** On stdio, stdout *is* the protocol channel. Lab 1 shows what happens
when you forget that.

## 5. Optional: MCP Inspector

The MCP Inspector is the official visual debugging tool. It needs Node.js:

```bash
npx @modelcontextprotocol/inspector
```

It is handy for clicking through a server's tools. The labs use raw messages and code instead, because
the exam asks about what is inside the messages.
