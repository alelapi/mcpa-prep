# Schemas & Structured Data

*Competency: **Schemas & Structured Data**.*

MCP is a protocol between programs and a language model. Both sides need to know the **shape**
of the data: what arguments a tool accepts, what a result looks like, what a form should ask
for. MCP uses **JSON Schema** for this, and a small set of **content types** for everything that
flows back.

The recommended experience for the exam includes the *"ability to read MCP server manifests and
capability definitions"*. This chapter is where that ability comes from.

---

## 1. The source of truth

The specification states that the protocol is defined by a **TypeScript schema**:

- [`schema.ts`](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/schema/2026-07-28/schema.ts)
  is the **source of truth** for all protocol messages and structures.
- A **JSON Schema** (`schema.json`) is **generated automatically** from it for use by tooling.

When the prose specification and the schema appear to disagree, the schema is authoritative for
message structure.

---

## 2. JSON Schema in MCP

JSON Schema is a vocabulary for describing and validating JSON. MCP uses it in three places:

1. **Tool `inputSchema`** — the arguments a tool accepts
2. **Tool `outputSchema`** — the structure of a tool's `structuredContent`
3. **Elicitation `requestedSchema`** — the form a user fills in

### Dialect rules

| Rule | Requirement |
| --- | --- |
| **Default dialect** | A schema **without `$schema`** is **JSON Schema 2020-12** |
| **Explicit dialect** | A schema **MAY** include `$schema` to use a different dialect (e.g. draft-07) |
| **Mandatory support** | Implementations **MUST** support at least **2020-12** |
| **Recommendation** | Implementors are **RECOMMENDED** to use 2020-12 |
| **Validation** | Schemas **MUST** be valid for their declared or default dialect; unsupported dialects must be handled **gracefully**, returning an error |
| **Documentation** | Implementations **SHOULD** document which additional dialects they support |

A 2020-12 schema (no `$schema` needed):

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer", "minimum": 0 }
  },
  "required": ["name"]
}
```

The same schema declared explicitly as draft-07:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer", "minimum": 0 }
  },
  "required": ["name"]
}
```

### `$ref` resolution: a security rule

JSON Schema lets `$ref` point at another schema, including one at an **absolute network URI**. A
validator that follows such references automatically would fetch arbitrary URLs chosen by whoever
wrote the schema. When the schema comes from an MCP server, that is a **server-side request forgery**
vector aimed at the client.

So the specification says:

- Implementations **MUST NOT** automatically dereference `$ref` values that resolve to a **network URI**.
- An opt-in mode that fetches non-local `$ref`s **MAY** exist, but **MUST be disabled by default** and
  **SHOULD** enforce a host allowlist, or at minimum reject loopback, link-local and private addresses,
  apply timeouts and size limits, and log dereferenced URIs.
- A schema that fails because of an unresolved external `$ref` **SHOULD be rejected**, not silently
  treated as permissive.

That last point closes a quieter hole. If an unresolvable reference were treated as "accept anything",
an attacker could disable validation just by pointing at a URL that never resolves.

### Composition keywords: a denial-of-service rule

`anyOf`, `oneOf`, `allOf`, `if`/`then`/`else` and `$defs` are expressive, and pathological schemas can
make validation exponentially expensive. Implementations **SHOULD** apply bounds, such as maximum
schema depth, a cap on total subschemas, or a per-validation time budget, so a malicious schema cannot
be used as a **denial-of-service vector** against the validator.

---

## 3. Tool schemas

A tool definition is the "manifest" most exam questions ask you to read:

```json
{
  "name": "get_weather_data",
  "title": "Weather Data Retriever",
  "description": "Get current weather data for a location",
  "inputSchema": {
    "type": "object",
    "properties": {
      "location": { "type": "string", "description": "City name or zip code" }
    },
    "required": ["location"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "temperature": { "type": "number", "description": "Temperature in celsius" },
      "conditions": { "type": "string" },
      "humidity": { "type": "number" }
    },
    "required": ["temperature", "conditions", "humidity"]
  },
  "annotations": { "readOnlyHint": true, "openWorldHint": true }
}
```

### `inputSchema`

- Defines the expected arguments; follows the dialect rules above.
- **MUST be a valid JSON Schema object, not `null`.**
- In `2026-07-28` it may use **any JSON Schema 2020-12 keyword**. <span class="new-2026">2026</span>
- Properties **MAY** carry an **`x-mcp-header`** annotation to mirror the value into an HTTP header (see
  [Transports](transports.md#custom-headers-from-tool-parameters)).

**For a tool with no parameters**, the spec gives two valid options:

| Schema | Behavior |
| --- | --- |
| `{ "type": "object", "additionalProperties": false }` | **Recommended.** Accepts only an empty object |
| `{ "type": "object" }` | Accepts any object, including one with properties |

The recommended form is stricter, so a model that invents arguments gets a validation error instead of
having them silently ignored.

### Why descriptions matter

The model never sees your code. It chooses tools and fills arguments **entirely** from `name`,
`description`, and the property `description`s inside `inputSchema`. A vague description ("does
weather stuff") produces wrong calls. A precise one ("City name or zip code, e.g. 'Paris' or '94103'")
produces right ones. **The schema is the model's documentation.**

That same fact is a security consideration: text in descriptions goes straight into the model's context,
which is why hidden instructions there (*tool poisoning*) work. See
[Attacks & Mitigations](../domain-4/attacks-mitigations.md).

### Tool names

| Rule | Level |
| --- | --- |
| 1–128 characters inclusive | SHOULD |
| Case-sensitive | SHOULD |
| Only `A–Z a–z 0–9 _ - .` | SHOULD |
| No spaces, commas or other special characters | SHOULD NOT |
| Unique within a server | SHOULD |

Valid examples: `getUser`, `DATA_EXPORT_v2`, `admin.tools.list`.

Names are unique only **within one server**. A host aggregating several servers **SHOULD** disambiguate
collisions (for example, prefixing with a server identifier), and **SHOULD NOT** rely on
`serverInfo.name`, which is not guaranteed unique.

### `outputSchema` and `structuredContent`

A tool result can carry two kinds of content:

- **Unstructured**: a `content` array of text, images, audio and resources, for the model and user to read.
- **Structured**: a `structuredContent` field, a JSON value for programs to consume.

If a tool declares an `outputSchema`:

- The server **MUST** return `structuredContent` that **conforms** to it.
- Clients **SHOULD** validate `structuredContent` against it.

```json
{
  "resultType": "complete",
  "content": [
    { "type": "text", "text": "{\"temperature\": 22.5, \"conditions\": \"Partly cloudy\", \"humidity\": 65}" }
  ],
  "structuredContent": { "temperature": 22.5, "conditions": "Partly cloudy", "humidity": 65 }
}
```

Three details:

1. **`structuredContent` may be any JSON value**: object, array, string, number, boolean or null. The
   spec's `list_users` example returns an **array**. <span class="new-2026">2026</span>
2. **Backward compatibility:** a tool returning structured content **SHOULD also** return the serialized
   JSON in a `text` content block, so clients that ignore `structuredContent` still get the data.
3. **`structuredContent` is not "structured outputs."** It is **server-produced result data**, unrelated to
   LLM structured outputs (schema-constrained model generation). The spec calls this out explicitly because
   the names are easy to confuse.

Why output schemas matter: they enable strict validation, give clients type information, and let hosts
**generate typed APIs** from tools, which is exactly what programmatic tool calling ("code mode") relies on.
See [Operational Use Cases](../domain-5/operational-use-cases.md).

---

## 4. Tool annotations

Annotations describe **behavior** as hints:

| Annotation | Meaning | Default | Meaningful when |
| --- | --- | --- | --- |
| `title` | Human-readable title | — | always |
| `readOnlyHint` | Tool does **not modify** its environment | `false` | always |
| `destructiveHint` | Tool **may perform destructive** updates (vs. only additive) | `true` | `readOnlyHint == false` |
| `idempotentHint` | Repeating the call with the same args has **no additional effect** | `false` | `readOnlyHint == false` |
| `openWorldHint` | Tool interacts with an **open world** of external entities | `true` | always |

Read the **defaults** carefully. They are **conservative**: absent any hint, a tool is assumed to be
**not read-only, possibly destructive, not idempotent, open-world**. A host that uses hints to decide
whether to auto-approve therefore errs toward caution when a server says nothing.

Examples: a web search tool is **open world**; a memory tool is **closed world**. `delete_file` is
destructive; `append_log` is additive (`destructiveHint: false`); `set_status("done")` is idempotent.

!!! danger "Annotations are untrusted"
    The specification: *"clients **MUST** consider tool annotations to be untrusted unless they come from
    trusted servers."* A malicious server can label `delete_everything` as `readOnlyHint: true`. Hints can
    make UX better for trusted servers; they can never be the basis of a security decision for untrusted
    ones.

---

## 5. Content blocks

Tool results, prompt messages and (deprecated) sampling messages share a common set of **content types**.

| `type` | Carries | Key fields |
| --- | --- | --- |
| `text` | Plain text | `text` |
| `image` | Image | `data` (base64), `mimeType` |
| `audio` | Audio | `data` (base64), `mimeType` |
| `resource_link` | A **pointer** to a resource | `uri`, `name`, `description?`, `mimeType?` |
| `resource` | An **embedded** resource, inline | `resource: { uri, mimeType, text \| blob }` |

```json
{ "type": "text", "text": "Tool result text" }
{ "type": "image", "data": "iVBORw0KG...", "mimeType": "image/png" }
{ "type": "audio", "data": "UklGRiQA...", "mimeType": "audio/wav" }
{ "type": "resource_link", "uri": "file:///project/src/main.rs", "name": "main.rs", "mimeType": "text/x-rust" }
{ "type": "resource", "resource": { "uri": "file:///project/src/main.rs", "mimeType": "text/x-rust", "text": "fn main() {}" } }
```

### Resource link vs. embedded resource

The distinction is examinable:

| | **`resource_link`** | **Embedded `resource`** |
| --- | --- | --- |
| Contains | A **URI** and metadata | The **contents** themselves |
| Client must | Fetch it (or subscribe) if it wants the data | Nothing, the data is already there |
| Good for | Large or optional data; letting the host decide | Small data the model definitely needs |
| Listed in `resources/list`? | **Not guaranteed** | Not applicable |
| Server should declare `resources`? | — | **SHOULD** |

A link keeps the result small and leaves the decision to the application, consistent with resources being
**application-controlled**.

Binary data (images, audio, resource `blob`) **MUST** be **base64-encoded** with a valid MIME type.

### Content annotations

All content types, resources and resource templates accept **annotations** that help the client decide how
to use them:

| Annotation | Values | Use |
| --- | --- | --- |
| `audience` | Array of `"user"`, `"assistant"` | Who it is for. `["user"]` might be shown but not sent to the model |
| `priority` | `0.0`–`1.0` | Importance. `1` ≈ effectively required; `0` = entirely optional |
| `lastModified` | ISO 8601 timestamp | When it last changed; sort or show recency |

```json
{
  "type": "image",
  "data": "base64-encoded-data",
  "mimeType": "image/png",
  "annotations": { "audience": ["user"], "priority": 0.9 }
}
```

`audience` is useful for **context economy**. A server can return a large human-readable report tagged
`["user"]` and a compact summary tagged `["assistant"]`, and a host can keep the report out of the model's
context window.

---

## 6. Other structured definitions

### Resource

```json
{
  "uri": "file:///project/src/main.rs",
  "name": "main.rs",
  "title": "Rust Software Application Main File",
  "description": "Primary application entry point",
  "mimeType": "text/x-rust",
  "size": 1024
}
```

`uri` and `name` identify it. `title`, `description`, `icons`, `mimeType` and `size` (bytes) are optional.
Contents come back from `resources/read` as `text` or `blob`.

### Resource template

```json
{
  "uriTemplate": "weather://forecast/{city}/{date}",
  "name": "weather-forecast",
  "title": "Weather Forecast",
  "mimeType": "application/json"
}
```

`uriTemplate` follows **RFC 6570 URI templates**. Arguments can be autocompleted with `completion/complete`.

### Prompt

```json
{
  "name": "code_review",
  "title": "Request Code Review",
  "description": "Asks the LLM to analyze code quality and suggest improvements",
  "arguments": [ { "name": "code", "description": "The code to review", "required": true } ]
}
```

Prompt **arguments are not JSON Schema**. They are a simple list of `{ name, description?, required? }`.
`prompts/get` returns `messages`, each with a `role` (`user` or `assistant`) and a content block.

### Elicitation `requestedSchema`

Form-mode elicitation uses a **deliberately restricted** subset of JSON Schema: a **flat object with
primitive properties only**, so any client can render it as a simple form.

| Allowed property type | Options |
| --- | --- |
| **string** | `minLength`, `maxLength`, `format` (`email`, `uri`, `date`, `date-time`), `default` |
| **number** / **integer** | `minimum`, `maximum`, `default` |
| **boolean** | `default` |
| **enum** (single select) | `enum: [...]`, or `oneOf: [{const, title}]` for titled options |
| **enum** (multi select) | `type: array` with `items.enum` or `items.anyOf: [{const, title}]`, `minItems`, `maxItems` |

**Not supported:** nested objects, arrays of objects (beyond enums) and other advanced keywords. Every
primitive supports `default`, and clients that support defaults **SHOULD** pre-populate fields with them.

This is a useful contrast for the exam. **Tool `inputSchema` accepts any 2020-12 keyword** (programs and
models fill it). **Elicitation `requestedSchema` is restricted** (humans fill it, through a generic UI).

---

## 7. Reading a server's "manifest"

Put it together. Given this `server/discover` result and one tool definition, what can you say?

```json
{
  "resultType": "complete",
  "supportedVersions": ["2026-07-28"],
  "capabilities": {
    "tools": { "listChanged": true },
    "resources": { "subscribe": true },
    "extensions": { "io.modelcontextprotocol/tasks": {} }
  },
  "_meta": { "io.modelcontextprotocol/serverInfo": { "name": "deploy-server", "version": "3.1.0" } },
  "ttlMs": 3600000,
  "cacheScope": "public"
}
```

```json
{
  "name": "deploy_service",
  "description": "Deploy a service version to an environment",
  "inputSchema": {
    "type": "object",
    "properties": {
      "service": { "type": "string" },
      "environment": { "type": "string", "enum": ["staging", "production"], "x-mcp-header": "Environment" },
      "version": { "type": "string" }
    },
    "required": ["service", "environment", "version"],
    "additionalProperties": false
  },
  "annotations": { "destructiveHint": true, "idempotentHint": true }
}
```

- Speaks **only** `2026-07-28`; a legacy client cannot use it.
- Offers **tools** and will send `tools/list_changed` to clients that subscribe via `subscriptions/listen`.
- Offers **resources** with **per-resource update subscriptions**, but not resource `listChanged`.
- Offers **no prompts** and **no completions**.
- Supports the **Tasks extension**: `deploy_service` might return a task handle instead of blocking.
- The discovery result can be cached for **one hour** and shared across users (`public`).
- `deploy_service` needs three strings, rejects extra arguments, and restricts `environment` to two values.
- Over Streamable HTTP, the client **must** send `Mcp-Param-Environment: production` alongside the body.
- It is **not read-only**, **may be destructive**, but is **idempotent**: deploying the same version twice has
  no additional effect. Those hints are **untrusted** unless this server is trusted.
- A careful host should **confirm** before calling it, especially for `production`.

---

## 8. Recap

- **`schema.ts`** is the source of truth; `schema.json` is generated.
- JSON Schema default dialect is **2020-12**; others via `$schema`; **2020-12 support is mandatory**.
- **Never auto-dereference network `$ref`s**; reject schemas with unresolved external refs; **bound**
  composition keywords against DoS.
- `inputSchema` **must be an object schema, not null**; for no parameters prefer
  `{"type":"object","additionalProperties":false}`.
- `outputSchema` present → server **MUST** return conforming `structuredContent`; clients **SHOULD** validate.
  `structuredContent` can be **any JSON value**, and should be mirrored as text for compatibility.
- Tool annotations `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` have **conservative
  defaults** and are **untrusted** unless the server is trusted.
- Content types: `text`, `image`, `audio` (base64), `resource_link` (pointer), `resource` (embedded).
- Content annotations: `audience`, `priority` (0–1), `lastModified`.
- Prompt arguments are a simple list, not JSON Schema. Elicitation schemas are **flat primitives only**.
