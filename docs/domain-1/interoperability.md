# Interoperability & Value

*Competency: **Interoperability & Value**.*

---

## 1. What "interoperability" means for MCP

**Interoperability** means that independently built software works together without prior
coordination. An MCP server written in Go by one company works with an MCP client written in
TypeScript by another, even though the two teams never spoke, because both followed the same
specification.

That is the value proposition in one sentence, and the official documentation expresses it as
**"build once and integrate everywhere."** A server author implements MCP one time and becomes
usable from Claude, ChatGPT, VS Code, Cursor, MCPJam and every other MCP host. A host developer
implements MCP one time and gains access to every MCP server.

Interoperability depends on three properties working together:

1. **A precise specification.** Normative language (MUST, SHOULD, MAY), a TypeScript schema as
   the source of truth, and a generated JSON Schema. Two implementations built from the same spec
   produce compatible messages.
2. **Capability negotiation.** Implementations support different subsets, and each side declares
   what it has, so a rich client and a minimal server still work together on their common ground.
3. **Versioning and compatibility rules.** Every request names its protocol version, and there are
   defined rules for mismatches and for talking across protocol eras.

---

## 2. Value for each stakeholder

The documentation frames MCP's benefits by where you sit in the ecosystem.

### Developers

> MCP reduces development time and complexity when building, or integrating with, an AI
> application or agent.

Consider a team that wants its internal ticketing system usable from AI tools. **Without MCP**,
it writes a plugin for each AI application it cares about, each in that application's plugin
format, and maintains all of them. **With MCP**, it writes **one** MCP server. Every current and
future MCP host can use it.

SDKs lower the cost further. With the Python or TypeScript SDK, a basic server exposing a couple
of tools is a few dozen lines. You write the function; the SDK produces the JSON Schema, the
protocol messages and the transport handling.

### AI applications and agents

> MCP gives them access to an ecosystem of data sources, tools and apps, which enhances their
> capabilities and improves the end-user experience.

An AI application that supports MCP does not need its own integration team for each service. Its
capability grows as the **ecosystem** grows. When a new service ships an MCP server, every MCP host
can use it the same day.

### End users

> MCP results in more capable AI applications or agents that can access user data and take
> actions on the user's behalf when necessary.

Users get assistants that can actually *do* things: read their calendar, file a ticket, query a
dashboard. They also get **portability**: a server they configured in one host can be configured in
another.

### Organizations

Not listed separately in the docs, but a direct consequence, and relevant to the governance roles
the exam targets:

- **Reuse:** an internal MCP server in front of a system of record serves every AI application in
  the company.
- **Central control:** one place to apply authorization, logging and rate limiting for AI access to
  that system, instead of scattered custom integrations.
- **Vendor neutrality:** the protocol is open and governed by a neutral foundation, so switching AI
  vendors does not mean rewriting integrations.

---

## 3. Composability

The specification's third stated goal is to **"build composable integrations and workflows."**

**Composability** means small, focused pieces that combine into something larger without
modification. MCP achieves it structurally: a host connects to **many** servers simultaneously, each
through its own client, and the model can use capabilities from all of them in a single task.

The documentation's travel-planning example shows it. A host connects three independent servers:

- a **travel server** (flights, hotels),
- a **weather server** (forecasts),
- a **calendar/email server** (schedules, messages).

The user invokes a `plan-vacation` **prompt** (from the travel server), attaches their calendar as a
**resource** (from the calendar server), and the model calls `searchFlights` and `checkWeather`
**tools** from two different servers, then `createCalendarEvent` and `sendEmail`. No server knows
the others exist. The **host** composes them.

!!! important "Composition happens in the host"
    Servers do not talk to each other in MCP. Cross-server interactions are **controlled by the
    host**. This keeps servers simple, and it keeps the security boundary in one place.

---

## 4. The design principles behind the value

The specification states four design principles. Each one explains *why* MCP delivers
interoperability and composability, and they are good exam material.

### Principle 1: Servers should be extremely easy to build

- Host applications handle complex orchestration responsibilities
- Servers focus on specific, well-defined capabilities
- Simple interfaces minimize implementation overhead
- Clear separation enables maintainable code

**Why it matters:** the ecosystem's value scales with the **number of servers**. Complexity was
deliberately pushed into hosts, of which there are few and which are built by well-resourced teams,
and away from servers, of which there are many and which are often built by one person in an
afternoon. The `2026-07-28` statelessness redesign continues this: a stateless server is simpler to
write, deploy and scale.

### Principle 2: Servers should be highly composable

- Each server provides focused functionality in isolation
- Multiple servers can be combined seamlessly
- Shared protocol enables interoperability
- Modular design supports extensibility

**Why it matters:** a server that does one thing well can be combined with any other.

### Principle 3: Servers should not be able to read the whole conversation, nor "see into" other servers

- Servers receive only necessary contextual information
- **Full conversation history stays with the host**
- Each server maintains isolation
- Cross-server interactions are controlled by the host
- Host process enforces security boundaries

**Why it matters:** this is the **security** principle hidden inside the architecture. You can
connect a third-party server without handing it your entire chat history or letting it read what
another server returned. It is the reason for one-client-per-server, and it is foundational to
[Trust Boundaries](../domain-4/trust-boundaries.md).

### Principle 4: Features can be added to servers and clients progressively

- Core protocol provides minimal required functionality
- Additional capabilities can be negotiated as needed
- Servers and clients evolve independently
- Protocol designed for future extensibility
- Backwards compatibility is maintained

**Why it matters:** it lets the ecosystem grow without lockstep upgrades. A server adding
resource subscriptions does not break clients that never subscribe. Optional **extensions** (Tasks,
MCP Apps, Skills) extend this principle beyond the core.

---

## 5. How MCP keeps interoperating as it evolves

A protocol that changes risks fragmenting its ecosystem. MCP has explicit mechanisms against that.

**Date-based versions.** A version is a string like `2026-07-28`, meaning the last date
backward-incompatible changes were made. Backward-compatible improvements do **not** change the
version.

**Per-request version declaration.** Every request states its version. A server that doesn't support
it responds with the versions it does support, and the client retries.

**Revision states.** A revision is **Draft** (in progress), **Current** (ready, may still receive
compatible changes) or **Final** (complete, frozen).

**Feature lifecycle.** A feature is **Active**, **Deprecated** (still works; new implementations
shouldn't adopt it; at least twelve months before removal eligibility) or **Removed**. A public
[deprecated features registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)
lists every deprecation with its migration path.

**Cross-era compatibility.** Implementations may be **dual-era**, supporting both modern
(`2026-07-28`) and legacy (`initialize`-based) peers, with defined detection rules on each
transport.

**Extensions.** New capabilities can be incubated as optional extensions, negotiated via capabilities,
without touching the core, and graduate through the SEP process.

---

## 6. Where interoperability stops

A standard protocol guarantees that messages are understood. It does **not** guarantee identical
behavior everywhere. Being precise about this helps with judgement questions.

**Hosts implement different subsets.** One host may support elicitation and MCP Apps; another may
support tools only. A server that depends on elicitation won't get it from a client that doesn't
declare the capability, and **must not** assume it. Servers **SHOULD** degrade gracefully, for
example returning meaningful text for a host that can't render an MCP App.

**Hosts use primitives differently.** One host exposes prompts as slash commands, another as a menu,
another not at all. The protocol does not mandate UI.

**Tool names can collide.** Tool names are only unique **within a server**. Two servers can both
expose `search`. Hosts that aggregate tools **SHOULD** disambiguate, for example by prefixing with a
server identifier. They **SHOULD NOT** rely on `serverInfo.name` for this, because server names are
not guaranteed unique.

**Model behavior varies.** The same tools produce different results with different models. MCP
standardizes the interface to the tool, not the reasoning that decides to call it.

**Configuration formats are host-specific.** How you register a server with a host
(`claude_desktop_config.json`, VS Code settings, and so on) is **not** part of the specification. The
MCP Registry's `server.json` format helps here by standardizing *installation metadata*, but each host
still has its own configuration.

---

## 7. Recap

- **Interoperability** = independently built clients and servers work together because they follow one
  spec. **"Build once, integrate everywhere."**
- It rests on a precise spec, **capability negotiation**, and **versioning** rules.
- **Developers** build one server instead of N plugins; **AI applications** gain an ecosystem;
  **end users** get capable, portable assistants; **organizations** get reuse, central control and
  vendor neutrality.
- **Composability:** a host connects many focused servers and **composes them itself**. Servers never
  talk to each other.
- Four design principles: servers **easy to build**; servers **composable**; servers **isolated** from the
  conversation and each other; features **progressive**.
- Evolution is managed with date versions, per-request negotiation, revision states, a deprecation
  lifecycle, dual-era compatibility and extensions.
- Interoperability does **not** guarantee identical features, UI or model behavior across hosts, and tool
  names can collide across servers.
