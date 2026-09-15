---
title: "Lab 2: Build a Server"
---

# Lab 2: Build a Server

**Time:** 60 minutes · **Goal:** build a server that exposes all three server primitives (tools,
resources and prompts), reports errors the right way, keeps state without sessions, and sends cache
hints.

## 1. Choosing the right primitive

Before writing any code, decide who should be in control of each feature. This is the question the
exam asks most often about primitives:

| Primitive | Controlled by | Use it when | In this lab |
| --- | --- | --- | --- |
| **Tool** | The **model** decides when to call it | An action, or a lookup the model should decide to do | `search_notes`, `create_note`, drafts |
| **Resource** | The **application** decides what to attach | Context data the host or user picks | `notes://index`, `notes://{note_id}` |
| **Prompt** | The **user** picks it, e.g. from a slash menu | A reusable, parameterised message template | `summarize_note` |

## 2. The server

Create `notes_server.py`. It is a single file, so each section is explained below the code.

```python title="notes_server.py"
"""Lab 2: a small notes server exposing tools, resources and prompts."""
import logging
import secrets
from datetime import datetime, timezone

from pydantic import BaseModel, Field
from mcp.server import MCPServer
from mcp.server.caching import CacheHint
from mcp.server.mcpserver.exceptions import ResourceNotFoundError, ToolError
from mcp.types import ToolAnnotations

logging.basicConfig(level=logging.INFO)
log = logging.getLogger("notes")

mcp = MCPServer(
    "notes",
    version="1.0.0",
    instructions="Search notes before creating a new one to avoid duplicates.",
    # Caching hints: the tool and prompt lists are the same for everyone and change rarely.
    cache_hints={
        "tools/list": CacheHint(ttl_ms=300_000, scope="public"),
        "prompts/list": CacheHint(ttl_ms=300_000, scope="public"),
    },
)

NOTES: dict[str, dict] = {
    "n1": {"title": "MCP transports", "body": "stdio for local, Streamable HTTP for remote."},
    "n2": {"title": "MRTR", "body": "Servers return input_required; clients retry with a new id."},
}


class Note(BaseModel):
    id: str
    title: str
    body: str


# ---------- Tools ----------

@mcp.tool(annotations=ToolAnnotations(readOnlyHint=True, openWorldHint=False))
def search_notes(query: str) -> list[Note]:
    """Search notes by a word in the title or body.

    Args:
        query: A word to look for, case-insensitive.
    """
    q = query.lower()
    return [Note(id=k, **v) for k, v in NOTES.items() if q in (v["title"] + v["body"]).lower()]


@mcp.tool(annotations=ToolAnnotations(readOnlyHint=False, destructiveHint=False, idempotentHint=False))
def create_note(title: str, body: str = Field(description="Markdown body, at most 2000 characters")) -> Note:
    """Create a new note and return it."""
    if len(body) > 2000:
        # ToolError becomes a tool execution error (isError: true) whose message the model can read.
        raise ToolError(f"Body is {len(body)} characters; the maximum is 2000. Shorten it and retry.")
    note_id = f"n{len(NOTES) + 1}"
    NOTES[note_id] = {"title": title, "body": body}
    log.info("created %s", note_id)
    return Note(id=note_id, title=title, body=body)


# ---------- Stateful handles (explicit state, no sessions) ----------

DRAFTS: dict[str, dict] = {}

class Draft(BaseModel):
    draft_id: str

@mcp.tool()
def start_draft(title: str) -> Draft:
    """Start a draft note. Returns a draft_id to pass to append_to_draft. Drafts expire after 1 hour."""
    draft_id = "dft_" + secrets.token_urlsafe(12)      # unguessable handle, not dft_1, dft_2...
    DRAFTS[draft_id] = {"title": title, "lines": [], "created": datetime.now(timezone.utc)}
    return Draft(draft_id=draft_id)

@mcp.tool()
def append_to_draft(draft_id: str, line: str) -> str:
    """Append a line to a draft started with start_draft."""
    draft = DRAFTS.get(draft_id)
    if draft is None:
        raise ToolError("Unknown or expired draft_id. Call start_draft to create a new draft.")
    draft["lines"].append(line)
    return f"Draft now has {len(draft['lines'])} line(s)."


# ---------- Resources ----------

@mcp.resource("notes://index", mime_type="text/markdown", description="Index of all notes")
def notes_index() -> str:
    return "\n".join(f"- {k}: {v['title']}" for k, v in NOTES.items())

@mcp.resource("notes://{note_id}", mime_type="text/markdown", description="A single note by id")
def note_by_id(note_id: str) -> str:
    note = NOTES.get(note_id)
    if note is None:
        raise ResourceNotFoundError(f"Note {note_id} not found")   # -> JSON-RPC -32602
    return f"# {note['title']}\n\n{note['body']}"


# ---------- Prompts ----------

@mcp.prompt(title="Summarize a note")
def summarize_note(note_id: str, audience: str = "engineer") -> str:
    """Ask the model to summarize a note for a given audience."""
    note = NOTES.get(note_id, {"title": "?", "body": "(missing)"})
    return f"Summarize this note for a {audience} in three bullet points:\n\n# {note['title']}\n{note['body']}"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

## 3. Walking through the design

### Tool annotations are hints, not guarantees

`readOnlyHint`, `destructiveHint`, `idempotentHint` and `openWorldHint` help a host decide how carefully
to treat a tool. For example, it might skip the confirmation dialog for a read-only lookup.

The spec is explicit that clients **MUST** treat annotations as untrusted unless the server itself is
trusted. A malicious server can label `delete_everything` as `readOnlyHint: true`. Annotations help UX;
they are not a security boundary.

### Return a model, get structured output

`search_notes` returns `list[Note]`, and `start_draft` returns `Draft`. With a Pydantic model or typed
return value, the SDK does two things:

1. It publishes an `outputSchema` in `tools/list`.
2. It fills `structuredContent` in every result, and the client can validate it against that schema.

!!! tip "Try it"
    Change `start_draft` to return a plain `dict`. `tools/list` no longer shows a useful `outputSchema`,
    and `result.structured_content` becomes `None`. A client that relies on structured output then has
    nothing to validate against.

### Two kinds of error

| Situation | What to raise | What the client receives | Who handles it |
| --- | --- | --- | --- |
| The model sent a body that is too long | `ToolError("... Shorten it and retry.")` | `result` with `isError: true` and your message | **The model**, which can correct itself |
| A resource id doesn't exist | `ResourceNotFoundError` | JSON-RPC `error`, code `-32602` | **The client application** |
| A bug in your code (e.g. `KeyError`) | Nothing: it escapes | `isError: true` with a generic message, so internals are not leaked | You, from the logs |

The spec draws the same line. **Tool execution errors** go back to the model in the result so it can
recover. **Protocol errors** are problems with the request itself, and the application handles them.

!!! tip "Try it"
    Replace `ToolError` with `ValueError` in `create_note`. The model now sees only a generic
    `Error executing tool create_note`, with no hint about the 2000-character limit, so it cannot
    fix its call. **A good tool error tells the model what to do next.**

### Stateful work without sessions

The `2026-07-28` protocol has no sessions: no `Mcp-Session-Id`, no connection-scoped memory. So how
does a multi-step workflow like a draft keep its state? It does what `start_draft` does: the server
**returns an explicit handle** (`draft_id`), and the model passes it back on later calls.

Rules for handles, all visible in the code:

- **Unguessable.** `secrets.token_urlsafe`, never `dft_1`, `dft_2`. If handles are guessable, one user
  can reach another user's state. The spec calls this *state handle hijacking*, and it is covered in
  [Attacks & Mitigations](../domain-4/attacks-mitigations.md#state-handle-hijacking).
- **Documented.** The description tells the model where the handle comes from and how long it lives.
- **Expiring**, with an actionable error when an expired or unknown handle comes back.
- In a multi-user deployment, also **bound to the authenticated user**. Check the owner on every use,
  not just that the handle exists.

### Resources and templates

`notes://index` is a **direct resource**, listed by `resources/list`. `notes://{note_id}` is a
**resource template** (RFC 6570), listed by `resources/templates/list`. The host fills in the template
to build concrete URIs.

For a URI that doesn't exist, `2026-07-28` uses **`-32602` Invalid params**. The old `-32002`
"resource not found" code is retired.

### Cache hints

Every list and discovery result carries `ttlMs` and `cacheScope`. By default the SDK sends
`ttlMs: 0, cacheScope: "private"`, meaning "don't reuse this". This server overrides the default for
`tools/list` and `prompts/list`, which are identical for every user and change rarely:
`scope="public"` lets a shared cache reuse them for five minutes.

!!! warning "Choosing a scope"
    `public` means the result is the same for every caller. If `tools/list` depended on the user's
    permissions, it would have to be `private`. Otherwise one user's tool list could be served to
    another user.

## 4. Test it in memory

The SDK `Client` can connect directly to an `MCPServer` object, with no subprocess. That makes it great
for tests. Create `test_notes.py`:

```python title="test_notes.py"
import anyio, json
from mcp import Client
from notes_server import mcp

def wire(model):
    """Show a result the way it looks on the wire (camelCase JSON)."""
    return json.dumps(model.model_dump(by_alias=True, exclude_none=True, mode="json"))

async def main():
    async with Client(mcp) as client:
        print("protocol:", client.protocol_version)
        r = await client.call_tool("search_notes", {"query": "mrtr"})
        print("search ->", wire(r))
        r = await client.call_tool("create_note", {"title": "big", "body": "x" * 2500})
        print("oversize ->", wire(r))
        d = await client.call_tool("start_draft", {"title": "t"})
        print("draft ->", d.structured_content)
        bad = await client.call_tool("append_to_draft", {"draft_id": "dft_guess", "line": "x"})
        print("bad handle ->", wire(bad))
        print("templates ->", wire(await client.list_resource_templates()))
        print("read n2 ->", wire(await client.read_resource("notes://n2")))
        try:
            await client.read_resource("notes://n99")
        except Exception as e:
            print("read n99 ->", type(e).__name__, getattr(e, "error", e))
        print("prompts ->", wire(await client.list_prompts()))
        print("get_prompt ->", wire(await client.get_prompt("summarize_note", {"note_id": "n1"})))

anyio.run(main)
```

```bash
python test_notes.py 2>/dev/null
```

Output, with `_meta.serverInfo` removed for readability:

```text
protocol: 2026-07-28
search -> {"content": [{"type": "text", "text": "{\n  \"id\": \"n2\",\n  \"title\": \"MRTR\", ...}"}],
           "structuredContent": {"result": [{"id": "n2", "title": "MRTR",
             "body": "Servers return input_required; clients retry with a new id."}]},
           "isError": false, "resultType": "complete"}
oversize -> {"content": [{"type": "text", "text": "Error executing tool create_note: Body is 2500
             characters; the maximum is 2000. Shorten it and retry."}], "isError": true, "resultType": "complete"}
draft -> {'draft_id': 'dft_FvDeigN2YQh7qcgT'}
bad handle -> {"content": [{"type": "text", "text": "Error executing tool append_to_draft: Unknown or
             expired draft_id. Call start_draft to create a new draft."}], "isError": true, ...}
templates -> {"ttlMs": 0, "cacheScope": "private", "resourceTemplates": [{"name": "note_by_id",
             "uriTemplate": "notes://{note_id}", "description": "A single note by id",
             "mimeType": "text/markdown"}], "resultType": "complete"}
read n2 -> {"ttlMs": 0, "cacheScope": "private", "contents": [{"uri": "notes://n2",
             "mimeType": "text/markdown", "text": "# MRTR\n\nServers return input_required; ..."}], ...}
read n99 -> MCPError code=-32602 message='Note n99 not found' data={'uri': 'notes://n99'}
prompts -> {"ttlMs": 300000, "cacheScope": "public", "prompts": [{"name": "summarize_note",
             "title": "Summarize a note", "arguments": [{"name": "note_id", "required": true},
             {"name": "audience", "required": false}]}], "resultType": "complete"}
get_prompt -> {"messages": [{"role": "user", "content": {"type": "text", "text": "Summarize this
             note for a engineer in three bullet points: ..."}}], "resultType": "complete"}
```

!!! question "Check yourself"
    1. Why does `prompts/list` show `ttlMs: 300000, cacheScope: "public"` while `resources/templates/list`
       shows `0` and `private`?
    2. The model called `append_to_draft` with a made-up id. Is that a protocol error or a tool execution
       error, and why is that the right choice here?
    3. The prompt argument `audience` has `required: false`. Where did that come from?

    ??? success "Answers"
        1. Only `tools/list` and `prompts/list` got a `CacheHint`. Everything else uses the SDK's
           conservative default.
        2. A tool execution error (`isError: true`). The request itself was valid; the failure is in the
           tool's domain, and the model can recover by calling `start_draft`.
        3. The Python default value `audience: str = "engineer"`.

## What this showed you

| You saw | Study chapter |
| --- | --- |
| Model-, application- and user-controlled primitives | [Core Concepts](../domain-1/core-concepts.md) |
| Annotations, structured output, explicit handles | [Tools & the Invocation Lifecycle](../domain-3/tools.md) |
| Direct resources, templates, prompt arguments | [Resources & Prompts](../domain-3/resources-prompts.md) |
| `isError` vs `-32602` | [Error Handling](../domain-3/error-handling.md) |
| `ttlMs` and `cacheScope` | [Caching, Pagination & Completion](../domain-3/utilities.md) |
| Unguessable, user-bound handles | [Attacks & Mitigations](../domain-4/attacks-mitigations.md) |
