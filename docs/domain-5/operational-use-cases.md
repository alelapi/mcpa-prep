# Operational Use Cases

*Competency: **Operational Use Cases**.*

Where MCP is used, how each use case maps onto primitives, transports and authorization, and the operational
patterns that make large deployments work. The patterns here come from the official
[Client Best Practices](https://modelcontextprotocol.io/docs/develop/clients/client-best-practices) and the
tools and extensions documentation.

---

## Part 1: Use case families

### 1. Developer tools and coding agents

**Hosts:** Claude Code, VS Code (GitHub Copilot), Cursor, JetBrains IDEs, Goose.

**Servers:** filesystem, git, GitHub, issue trackers, error monitoring (Sentry), CI systems, databases, cloud consoles, design tools (Figma).

The documentation's headline example: *"Claude Code can generate an entire web app using a Figma design."*

| Need | Primitive | Typical deployment |
| --- | --- | --- |
| Read and edit project files | Tools (and resources) on a **filesystem** server | **Local, stdio** |
| Inspect repo history | Tools / resources on a **git** server | Local, stdio |
| Open PRs, read issues | Tools on a **GitHub** server | Remote (Streamable HTTP + OAuth) or local with a token in env |
| Investigate production errors | Tools on **Sentry**'s hosted server | **Remote**, OAuth |
| Standard workflows ("/review", "/fix-issue") | **Prompts** | Any |
| Project context (open file, README) | **Resources** | Any |

**Risks to manage:** code-execution tools; **prompt injection through issue and PR text** written by outsiders; secrets in the workspace; one-click installation
of local servers.

### 2. Knowledge and data access

**Example:** *"Enterprise chatbots can connect to multiple databases across an organization, empowering users to analyze data using chat."*

A database server typically exposes:

- **resources** for schemas (`postgres://db/schema/{table}` via templates),
- **tools** for queries (`run_query`, ideally read-only),
- **prompts** with few-shot examples of good queries.

**Operational concerns:** **read-only credentials** behind the tool; per-user **row-level authorization** from the validated token; `cacheScope: "private"` for
user-specific reads; **pagination** for large lists; **completion** on template parameters (with access control, to avoid disclosure); audit logging of queries.

### 3. Enterprise systems of record

CRM, ticketing, HR, finance, document management. Characteristics:

- **Remote** servers, often operated by the platform team or the SaaS vendor.
- **Corporate identity**: Enterprise-Managed Authorization, or core OAuth backed by the corporate IdP.
- Strong need for **consent on writes**, **least-privilege scopes**, **auditability**.
- **Elicitation** for confirmations ("Approve refund of $1,200?") and **URL-mode** elicitation for connecting third-party accounts.
- Sometimes an **MCP gateway** in front of many internal servers, centralizing authentication, policy, rate limits and logging. The Streamable HTTP
  header mirroring (`Mcp-Method`, `Mcp-Name`, `Mcp-Param-*`) exists precisely so such **intermediaries** can route and enforce policy without parsing bodies.

### 4. Personal productivity

**Example:** *"Agents can access your Google Calendar and Notion, acting as a more personalized AI assistant."*

Calendar, email, notes, task managers. Mostly **remote servers with per-user OAuth**. The travel-planning scenario from the docs composes several such servers:
a prompt starts the workflow, calendar resources give context, tools search flights, book hotels, create events and send email, with **elicitation** for booking
confirmation.

**Risks:** these hosts combine **private data** (email, calendar) with **untrusted content** (incoming email bodies) and **outbound actions** (send email). That
is the high-risk combination from [Trust Boundaries](../domain-4/trust-boundaries.md), so confirmation with visible arguments matters most here.

### 5. Creative and physical-world automation

**Example:** *"AI models can create 3D designs on Blender and print them out using a 3D printer."*

Tools that drive desktop applications or devices, usually **local**. **MCP Apps** fit well when the user needs to see and manipulate rich output (a 3D preview,
a chart, a PDF) in the conversation.

### 6. Automation and machine-to-machine

CI/CD pipelines, scheduled reports, back-office agents: **no human in the loop at call time**.

- Authorization via the **OAuth Client Credentials** extension (JWT bearer assertions recommended).
- Clients **don't declare `elicitation`** when no user is present, so servers **must not** depend on it. A server that truly needs a human returns
  `MissingRequiredClientCapabilityError`, or is simply not used in that context.
- **Tasks extension** for long jobs, with durable handles that survive restarts.
- Human oversight moves **earlier** (policy, scoped credentials, approval of the automation itself) and **later** (audit review).

---

## Part 2: Deployment patterns

| Pattern | Shape | Good for | Watch out for |
| --- | --- | --- | --- |
| **Local stdio server** | Host launches server subprocess | Files, local tools, single user | Runs with user privileges; consent to command; sandbox |
| **Local HTTP server** | Server on `127.0.0.1` | Sharing one local server among several local apps | **Origin validation**, loopback binding, token auth |
| **Remote server** | Public or internal HTTPS endpoint | Shared services, many users | OAuth, audience validation, rate limits, SSRF-safe clients |
| **Horizontally scaled remote** | Many stateless instances behind a load balancer | High traffic | Explicit handles and `requestState` carry state; no sticky sessions needed |
| **Gateway / proxy** | One endpoint fronting many servers | Central policy, auth, logging | **Confused deputy** if using static upstream client IDs; header–body validation |
| **Embedded / in-process** | Custom transport inside one application | Tight integration | Must preserve JSON-RPC, patterns and per-request metadata |

!!! tip "Why statelessness matters operationally"
    Because `2026-07-28` requests are self-contained, a remote MCP server scales like any stateless web service: round-robin load balancing, autoscaling, rolling
    deploys, instance failure without lost sessions. MRTR retries and explicit handles carry the little state that must survive between requests. SEP-2575's
    motivation was exactly this operational pain.

---

## Part 3: Patterns for hosts at scale

### 7. The problem

As hosts connect more servers they accumulate **hundreds or thousands of tools**. Two costs appear:

1. **Definition cost.** Every tool definition in the model's context costs tokens **every turn**. With dozens of servers, definitions can consume **most of the
   context window** before the model reads the user's message. Too many options also **degrade tool selection**. The documentation's illustration: roughly
   **150,000 tokens** of definitions loaded up front, versus about **2,000** with progressive discovery.
2. **Intermediate-result cost.** In a multi-step task (read a document, transform it, write it elsewhere), every intermediate result passes **through the
   model's context**, costing tokens and latency even when the model doesn't need to read it.

Two patterns address them: **progressive discovery** controls *when* definitions enter context; **programmatic tool calling** controls *how* tools are invoked.
Neither changes the protocol. Both are host design.

### 8. Progressive tool discovery

**Idea:** fetch tool definitions with `tools/list` as normal, but **don't inject them all** into the model's context. Give the model a lightweight
**`search_tools`** meta-tool, and load full definitions only when needed.

**When to switch:** implement a threshold, for example when definitions exceed **1–5% of the context window**. Below it, loading everything is fine.

**Search strategies:**

| Strategy | How | Trade-off |
| --- | --- | --- |
| **Keyword** | BM25, regex over names and descriptions | Simple and effective with descriptive names |
| **Embedding** | Vector similarity over descriptions | Handles synonyms and semantics better |
| **Subagent** | A small fast model picks tools for the task | Usually very good; costs more |
| **Hybrid** | Combine rankings, or pick a strategy per query | Best of several |

Some model providers offer **built-in tool search**, which may be preferable unless you need custom ranking or **access-control filtering**.

**The three-layer approach:**

1. **Catalog.** The model calls `search_tools({ query: "update salesforce record" })` and gets **names and one-line descriptions** only.
2. **Inspect.** The model calls `get_tool_details({ name: "salesforce_updateRecord" })` and gets the **full schema** for that tool alone.
3. **Execute.** The model calls the tool with full knowledge of its interface.

**Dynamic server management** extends this to whole servers: keep a registry of available servers with descriptions, **connect** a server only when the model
needs it (then `server/discover` and `tools/list`), and **disconnect** servers no longer relevant, freeing context. Skills can declare which servers they need,
so the host connects them only when the skill is invoked.

**Implementation guidelines:**

| Guideline | Why |
| --- | --- |
| Offer **multiple detail levels** (name only, name+description, full schema) | Let the model pay only for what it needs |
| **Cache** definitions host-side | Re-injecting later needs no new `tools/list` |
| **Refresh on `list_changed`** | Keep the search catalog correct |
| **Group tools by server** | Helps the model reason about related capabilities and provenance |
| Follow **`ttlMs` / `cacheScope`**; treat a list as stale on `list_changed` even before TTL expiry | Correct freshness |

**Prompt caching interaction.** Providers cache the prompt prefix, **including the tools array**. Adding or removing tools mid-conversation **invalidates** it,
and the cache miss can cost more than the definitions saved. To preserve caching: **append** new definitions after the cache breakpoint instead of re-sorting,
or route every call through a single stable **`call_tool({name, args})`** meta-tool so the array never changes; treat server disconnection as a
**conversation-boundary** operation. This is also why servers **SHOULD return tools in deterministic order**.

### 9. Programmatic tool calling ("code mode")

**Idea:** instead of calling tools one at a time with every result flowing through the model, the model **writes code that calls tools**. The code runs in a
**sandbox**, and **only the final output** returns to the model.

**How it works:**

1. **Generate a typed API from MCP schemas.** The host turns each tool's `inputSchema` and **`outputSchema`** into typed functions.

    ```typescript
    function logging_getLogs(input: { level: "error" | "warn" | "info"; since: number }):
      Promise<{ entries: LogEntry[] }> { return mcp.callTool("logging_getLogs", input); }

    function ticketing_createIssue(input: { title: string; body?: string; priority: "low" | "medium" | "high" }):
      Promise<{ issueId: string }> { return mcp.callTool("ticketing_createIssue", input); }
    ```

2. **The model writes a script.** "File a ticket for each unique error in the last hour":

    ```typescript
    const logs = await logging_getLogs({ level: "error", since: Date.now() - 3600000 });
    const unique = new Map<string, LogEntry>();
    for (const log of logs.entries) if (!unique.has(log.message)) unique.set(log.message, log);
    for (const [message, log] of unique) {
      await ticketing_createIssue({ title: `Error: ${message}`, body: `First seen: ${log.timestamp}`, priority: "high" });
    }
    console.log(`Filed ${unique.size} tickets from ${logs.entries.length} error logs`);
    ```

3. **The sandbox executes it.** Function calls are intercepted and routed through the **host broker** as `tools/call` requests. Thousands of log entries flow
   between servers **without entering the model's context**. The model sees one summary line.

**Why `outputSchema` matters here:** with it, the host generates precise return types. Without it, use a generic type, or extract a typed result with a small
model through a host-brokered helper (validating the result, since extraction can hallucinate or drop fields). **The real fix is for server authors to provide
`outputSchema`.**

**Architecture:**

- **Sandbox**: runs model code with **no direct network access**; its only interface is the generated stubs.
- **Host broker**: maps calls to servers, executes them, returns results; **holds credentials**, which are never exposed to generated code.
- **Model**: sees only `console.log` output or a return value.

Example runtimes: Deno or `isolated-vm` (JavaScript), Monty (Python, experimental), pctx (TypeScript, early), **Wasmtime** (any language via Wasm).

**Security considerations** (from the docs):

- **Per-call authorization.** The broker is still the MCP host for spec purposes. Apply the **same human-in-the-loop policy** to sandbox-originated calls.
  Approving a script **isn't blanket approval** for every call it makes; categorical grants are allowed but each call is still checked against them.
- **Cross-server data flow.** Results from one server are **untrusted input** to another; review brokered calls like direct ones. Output truncation alone
  doesn't prevent exfiltration.
- **Network isolation** of the sandbox; **no credential exposure**; **resource limits** (timeouts, memory); **filter and truncate** output before it reaches the model.

**Error handling:** a tool failure arrives as `isError: true`, not a transport error. Generated wrappers should **throw** on it so model code can `try`/`catch`; an
uncaught error is surfaced as the script's result so the model can self-correct, and the model must report partial side effects already committed.

**Combining both patterns:** discover the few tools needed, load their schemas, then write **one** script calling them. This minimizes both definition tokens and
result tokens.

---

## Part 4: Workflow patterns

### 10. Stateful workflows

MCP has no session. For multi-step stateful work (carts, browser sessions, transactions), servers return **explicit handles** from creation tools; the model passes
them back; servers **validate ownership on every call**, state the **retention** in the tool description, and return a **tool execution error** for expired handles so
the model can start over.

### 11. Human approval workflows

- **Inline confirmation**: host dialog before a tool call.
- **Server-driven confirmation**: the server returns `InputRequiredResult` with a **form elicitation** ("Confirm deploy to production?") and completes on retry with
  `accept`. The server enforces the approval itself, so it can't be bypassed by a misconfigured host policy.
- **Long approval gates**: the **Tasks extension**, where a task moves to `input_required` and the client supplies input via `tasks/update`.

### 12. Long-running operations

| Duration | Mechanism |
| --- | --- |
| Seconds | Normal request; **progress** notifications; timeouts |
| Tens of seconds to minutes | Progress with timeout reset, plus a maximum timeout; cancellation |
| Minutes to hours, or unreliable connections | **Tasks extension**: durable `taskId`, `tasks/get` polling, `tasks/cancel` |

### 13. Rich interactive output

When text isn't enough (exploring data, many-option configuration, rich media, live monitoring, multi-step review), **MCP Apps** render a sandboxed HTML UI inside the
conversation that can call the server's tools. Servers should still return meaningful **text** for hosts without the extension.

### 14. Reusable expertise

**Skills over MCP** lets a server publish workflow instructions (`SKILL.md` plus supporting files) that hosts discover and load on demand, keeping instructions next
to the service they describe. **Prompts** are the core-protocol analogue for user-triggered templates.

---

## Part 5: Anti-patterns

| Anti-pattern | Why it's a problem | Better |
| --- | --- | --- |
| Exposing every tool of every server to the model every turn | Token cost; poor tool selection | Progressive discovery above a threshold |
| One mega-tool `execute(action, params)` | Model can't see capabilities; weak validation; coarse permissions | Focused tools with precise schemas |
| Vague tool descriptions | Wrong calls | Say what it does, when to use it, argument formats |
| Returning huge results to the model | Context bloat | Pagination, `resource_link`, `audience` annotations, code mode |
| Relying on connection state | Breaks under load balancing; forbidden | Explicit handles, `requestState` |
| Auto-approving based on untrusted annotations | Trivially bypassed | Trust by server provenance; confirm risky calls |
| Asking for secrets in form elicitation | Secrets pass through the client | URL mode |
| Passing user tokens to upstream APIs | Forbidden token passthrough | Separate upstream credentials |
| Shuffling `tools/list` order | Breaks client and prompt caches | Deterministic order |
| Protocol errors for bad argument values | Model can't self-correct | `isError: true` with actionable text |
| Building new features on sampling, roots or logging | Deprecated | Provider APIs; arguments/config; stderr/OpenTelemetry |

---

## Recap

- Use cases: **developer tools** (local filesystem/git, remote GitHub/Sentry), **data access** (schema resources, read-only query tools), **enterprise systems** (remote,
  corporate identity, gateways), **personal productivity** (per-user OAuth; high injection risk), **creative/physical** (local tools, MCP Apps), **M2M** (client
  credentials, no elicitation, Tasks).
- Deployment: local stdio, local HTTP (Origin + loopback + auth), remote, **stateless horizontally scaled**, gateways (watch confused deputy), in-process custom transports.
- **Progressive discovery**: threshold ~**1–5%** of context; keyword / embedding / subagent / hybrid search; **catalog → inspect → execute**; dynamic server
  connection; cache and refresh on `list_changed`; **protect prompt caching** (append, stable `call_tool`, deterministic order).
- **Code mode**: typed API from schemas (**`outputSchema` matters**), model-written script in a **network-isolated sandbox**, host broker holds credentials, **same
  per-call approval policy**, cross-server results untrusted, `isError` → exception.
- Workflows: explicit handles; elicitation-based approvals; Tasks for long jobs; MCP Apps for rich UI; Skills for reusable instructions.
