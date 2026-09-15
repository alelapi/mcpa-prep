# Ecosystem & Portability

*Competency: **Ecosystem & Portability**.*

A protocol is only as useful as the ecosystem around it. This chapter covers the pieces you'll meet in practice
(SDKs, the Inspector, reference servers, the MCP Registry, host support) and what **portability** does and does
not mean: moving servers across hosts, across transports, and across protocol eras.

---

## Part 1: SDKs

### 1. Official SDKs and tiers

SDKs implement the protocol so developers don't hand-write JSON-RPC. Official SDKs are **tiered**:

| SDK | Tier |
| --- | --- |
| **TypeScript** | Tier 1 |
| **Python** | Tier 1 |
| **C#** | Tier 1 |
| **Go** | Tier 1 |
| **Rust** | Tier 1 |
| **Java** | Tier 2 |
| **Ruby** | Tier 2 |
| **Swift** | Tier 3 |
| **PHP** | Tier 3 |
| **Kotlin** | Tier 3 |

All SDKs support creating servers that expose tools, resources and prompts, building clients that connect to any server, local and remote transports,
and type-safe protocol compliance.

### 2. What the tiers mean

| Requirement | **Tier 1: Fully Supported** | **Tier 2: Commitment to Full Support** | **Tier 3: Experimental** |
| --- | --- | --- | --- |
| **Conformance tests** | **100%** pass | **80%** pass | No minimum |
| **New protocol features** | **Before** the new spec version releases | Within **6 months** | No commitment |
| **Issue triage** | Within **2 business days** | Within a month | None |
| **Critical bug fix** | Within **7 days** | Within two weeks | None |
| **Stable release** | Required, clear versioning | At least one | Not required |
| **Documentation** | Comprehensive, examples for all features | Basic core coverage | No minimum |
| **Dependency policy** | Published | Published | Not required |
| **Roadmap** | Published | Plan toward Tier 1, or explanation | Not required |

Tier 1 SDKs implement the complete protocol, **including optional capabilities** like elicitation (and deprecated sampling). **Experimental features and
extensions (Tasks, MCP Apps) are not required for any tier.**

### 3. Conformance testing

Tiers are measured with **automated conformance tests** in the
[`modelcontextprotocol/conformance`](https://github.com/modelcontextprotocol/conformance) repository. Scores count only **applicable required tests**:
those for the targeted spec version, excluding pending, skipped, experimental, disputed, and (unless the SDK claims it) legacy backward-compatibility tests.

The same repository ties into the SEP process: Standards Track SEPs with observable behavior need a **conformance scenario** merged, with a traceability file
mapping every MUST and SHOULD to a check, before reaching **Final**.

**Why it matters for portability:** conformance testing is what makes "implements MCP" a **verifiable** claim, so a server built with one SDK really works
with a client built with another.

---

## Part 2: Developer tools

### 4. MCP Inspector

The **reference tool for testing and debugging MCP servers**: one package, `@modelcontextprotocol/inspector`, providing **three clients behind one binary**:

| Client | Invocation | For |
| --- | --- | --- |
| **Web** | `npx @modelcontextprotocol/inspector` | Full graphical inspector; the default |
| **CLI** | `npx @modelcontextprotocol/inspector --cli` | Scriptable, machine-readable output for CI, pipelines, coding agents |
| **TUI** | `npx @modelcontextprotocol/inspector --tui` | Interactive terminal UI |

All three share one core: the **same transports**, configuration files, **OAuth state**, and **protocol-era negotiation** (legacy vs. modern `2026-07-28`).

```bash
# Inspect a local stdio server
npx @modelcontextprotocol/inspector node path/to/server/index.js

# Inspect a published npm server
npx -y @modelcontextprotocol/inspector npx @modelcontextprotocol/server-filesystem ~/Desktop

# Call a tool on a remote server from the CLI, JSON output
npx @modelcontextprotocol/inspector --cli https://api.example.com/mcp --transport http \
  --method tools/call --tool-name get_weather --tool-arg city=Boston --format json
```

Requires **Node 22.19.0+**. The web client prints a URL with a **one-time session token**, a small but telling security detail: even a local debugging tool
authenticates its browser session.

Typical uses: verify `server/discover` and capabilities, exercise tools and resources, watch raw protocol traffic, test OAuth, test behavior across protocol eras,
and run **smoke tests in CI** with the CLI.

### 5. Reference servers

The [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers) repository holds **reference implementations**, such as the **filesystem**
server (which demonstrates respecting roots) and **everything**-style test servers. They are **examples**, not production-hardened products.

---

## Part 3: The MCP Registry

### 6. What it is

> The MCP Registry is the official centralized **metadata** repository for publicly accessible MCP servers.

Live at **`registry.modelcontextprotocol.io`**, currently in **preview** (breaking changes or data resets may occur before general availability). It provides:

- a single place for server creators to **publish metadata**,
- **namespace management** through verification,
- a **REST API** for clients and aggregators to discover servers,
- **standardized installation and configuration** information.

### 7. What it stores: `server.json`

Metadata, **not code**:

```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.username/email-integration-mcp",
  "title": "Email Integration",
  "description": "Send emails and manage email accounts",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@username/email-integration-mcp",
      "version": "1.0.0",
      "transport": { "type": "stdio" }
    }
  ]
}
```

It contains the server's **unique name**, **where to find it** (a package or a remote URL), **how to run it** (arguments, environment variables), and discovery data
(description, capabilities).

### 8. Relationship to package registries

**npm, PyPI, NuGet, Docker Hub** and others host **packages with code**. The MCP Registry hosts **metadata pointing to them**. Supported `registryType` values are **`npm`** (`registry.npmjs.org` only), **`pypi`** (`pypi.org` only), **`nuget`**, **`oci`** (Docker/OCI
images), **`cargo`** (crates.io) and **`mcpb`** (prebuilt MCP bundles from GitHub or GitLab releases). Remote servers can be listed by URL.

### 9. Namespaces and verification

Server names use **reverse-DNS namespaces** that tie a server to a verified owner:

| Namespace | Proof required |
| --- | --- |
| `io.github.domdomegg/my-cool-mcp` | Logged in to GitHub as `domdomegg`, or a GitHub Action on that user's repos |
| `me.adamjones/my-cool-mcp` | Ownership of `adamjones.me` via **DNS** or **HTTP** challenge |

**Package ownership** is verified too:

- **npm:** `package.json` **`mcpName`** must equal the server name.
- **PyPI:** the README (package description) must contain **`mcp-name: <server name>`**, which may be hidden in an HTML comment.

!!! important "Verified publisher ≠ safe server"
    Namespace verification proves **who published** the metadata. It says nothing about whether the code is safe. The registry **delegates security scanning**
    to the underlying **package registries** and to **downstream aggregators**, and focuses on namespace authentication and metadata hosting.

### 10. Consumers: aggregators, subregistries, hosts

- The registry is designed to be consumed **primarily by downstream aggregators**, such as marketplaces, which scrape its **unauthenticated read-only REST API**
  (for example `GET /v0.1/servers`, cursor-paginated, filterable by `updated_since`) on a regular but **infrequent** basis (about hourly) and store data themselves.
  There are **no uptime or durability guarantees**.
- Its metadata is **deliberately unopinionated**; aggregators add **curation, ratings and security scanning**.
- A **subregistry** is an aggregator that also implements the registry's **OpenAPI spec**, so hosts can consume it through a standard interface. Subregistries may
  add custom data under a namespaced `_meta` key (ratings, download counts, scan results).
- **Host applications are not meant to consume the official registry directly.** They consume subregistries or marketplaces implementing the OpenAPI spec.
- **Private servers** (internal networks, private package registries) are **not supported**. Organizations should run **their own private registry** implementing
  the OpenAPI spec. The official registry codebase isn't designed for self-hosting.

### 11. Lifecycle, moderation, spam

- **Version metadata is immutable** once published; update by publishing a new version.
- The **`status`** field can change to `deprecated` or `deleted`. Deleted servers are hidden from default listings but **never permanently removed**, and aggregators
  should keep statuses current. `deleted` typically indicates a moderation-policy violation (spam, malware).
- Spam prevention: namespace verification, character limits and validation, manual takedown.
- Publishers may include custom metadata under `_meta["io.modelcontextprotocol.registry/publisher-provided"]`, up to **4 KB**.
- Abuse: report to the **package registry** and open a registry issue titled `Abuse report: …`.

---

## Part 4: Host support

MCP is supported across AI assistants (Claude, ChatGPT), development tools (VS Code, Cursor, JetBrains, MCPJam), agent frameworks (Goose) and more. Support for
**optional** features varies by host:

- Some hosts support **tools only**; others add resources, prompts and elicitation.
- **Extension** support is tracked in the extension support matrix; MCP Apps is broadly supported, authorization extensions and Skills less so.
- Protocol version adoption also varies during the `2026-07-28` transition.

**Design implication for server authors:** depend only on what the client **declares**. Check capabilities per request, provide **text fallbacks**, and don't assume
elicitation, subscriptions or any extension.

---

## Part 5: Portability

### 12. What is portable, and what isn't

| Portable (defined by the spec) | Not portable (host-specific) |
| --- | --- |
| JSON-RPC messages and their semantics | **Configuration files** (`claude_desktop_config.json`, VS Code settings…) |
| Primitives: tools, resources, prompts | **UI** for prompts, resources, approvals |
| Capability negotiation | Which **optional features and extensions** a host supports |
| Transports: stdio, Streamable HTTP | **Tool selection** and context management strategy |
| Authorization flow | **Approval policies** and auto-approval rules |
| Error codes and result types | The **model** and how it behaves with your tools |
| Caching and pagination semantics | Tool **name disambiguation** scheme |

`server.json` narrows the configuration gap by standardizing **installation metadata** that hosts and marketplaces can translate into their own config.

### 13. Transport portability

The **same data-layer messages** run over stdio and Streamable HTTP, so one server implementation can offer both, typically by choosing the transport at startup.
What changes between them:

- **metadata**: HTTP adds mirrored headers;
- **cancellation**: stream close vs. `notifications/cancelled`;
- **credentials**: environment vs. OAuth;
- **security controls**: Origin validation and loopback binding for HTTP.

Custom transports over byte streams **SHOULD reuse stdio framing**, which keeps them portable too.

### 14. Portability across protocol eras

The biggest portability challenge right now is the **legacy → modern** transition.

| Situation | Outcome | What to do |
| --- | --- | --- |
| Modern client, modern server | Works | — |
| **Dual-era** client, legacy server | Works via fallback | stdio: probe `server/discover`, fall back to `initialize`. HTTP: inspect the `400` body |
| Modern-only client, legacy server | Fails | Probe first (on stdio) to fail cleanly; upgrade the server or add dual-era support |
| **Legacy client, modern-only server** | **Fails** (no fall-forward) | Server **should name its supported versions** in the `initialize` error; support dual-era during migration |
| Legacy client, dual-era server | Works | Server answers `initialize` with legacy semantics |

Cache the detected era per server process (stdio) or origin (HTTP). Details: [Versioning & Discovery](../domain-2/versioning-discovery.md).

### 15. Portability over time: deprecations

Deprecated features (sampling, roots, logging, DCR, HTTP+SSE) still work, but a server that depends on them will become **less portable** as hosts drop support and as
the features become eligible for removal (on or after 2027-07-28 for most). Portable designs **avoid building new functionality on deprecated features**.

### 16. A portability checklist for server authors

- [ ] Target **`2026-07-28`**; add dual-era support only if legacy clients matter
- [ ] Implement **`server/discover`** with accurate capabilities and useful `instructions`
- [ ] Offer **stdio and Streamable HTTP** if the server makes sense both locally and remotely
- [ ] **Check client capabilities per request**; never assume elicitation or extensions
- [ ] Provide **text fallbacks** for structured content and MCP Apps
- [ ] Use **clear, collision-resistant tool names** within the spec's character rules
- [ ] Provide **`outputSchema`** so hosts can type results (and use code mode)
- [ ] Avoid **deprecated features** for new work
- [ ] Publish **`server.json`** to the MCP Registry under a verified namespace
- [ ] Test with the **Inspector** (web for exploration, CLI in CI), and against more than one host

---

## Recap

- **SDKs:** Tier 1 = TypeScript, Python, C#, Go, Rust; Tier 2 = Java, Ruby; Tier 3 = Swift, PHP, Kotlin. Tier 1 = **100% conformance**, new features **before** spec release,
  triage in **2 business days**, critical fixes in **7 days**. Extensions aren't required for any tier.
- **Conformance tests** make "implements MCP" verifiable, and gate **Final** SEPs.
- **Inspector:** one package, **web / CLI / TUI**, shared transports, OAuth state and era negotiation; CLI for CI.
- **MCP Registry** (preview): **metadata only** (`server.json`); **reverse-DNS namespaces** verified via GitHub, DNS or HTTP; package ownership via `mcpName` (npm) and
  `mcp-name:` (PyPI); **verification ≠ safety** (scanning delegated to package registries and aggregators); consumed by **aggregators and subregistries** (OpenAPI), not
  directly by hosts; **no private servers**; immutable versions; `deleted` status.
- **Portable:** messages, primitives, capabilities, transports, auth flow, errors. **Not portable:** host config, UI, optional feature support, tool selection, approval
  policy, model behavior.
- Era portability: **dual-era clients** reach everyone; **legacy clients can't reach modern-only servers**. Avoid deprecated features for durability.
