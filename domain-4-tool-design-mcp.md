# CCA-F Domain 4: Tool Design & MCP Integration (18% of exam)

**Source note:** Built from Anthropic's current MCP documentation (Claude Code's `/en/mcp` reference) and the Agent SDK's custom-tools documentation. Verified live, June 2026. This domain rewards precision on naming conventions, exact mechanics (transport types, scopes, error-handling contracts), and the specific failure modes each design choice guards against — not general API-design theory.

---

## 1. Tool Descriptions as Routing Mechanisms

### 1.1 The Core Principle
**Claude decides which tool to call based primarily on the tool's name and description.** This is not a minor detail — it's the entire mechanism by which "the right tool gets called at the right time" actually works. A tool with a vague or misleading description will be selected at the wrong moments, or not selected when it should be, regardless of how well the tool itself is implemented.

### 1.2 Writing Descriptions That Route Correctly
A description should state clearly *what the tool does* and, implicitly or explicitly, *when it's the right choice* — the same principle MCP server *instructions* extend to the server level (see §4.5): "explain what category of tasks your tools handle, when Claude should reach for them, and key capabilities." A description that only restates the tool's name in different words gives Claude nothing more to route on than the name itself already provided.

### 1.3 Field-Level Descriptions Matter Too
In the Agent SDK, individual schema fields can carry their own descriptions (`.describe()` in Zod, a `"description"` key in JSON Schema) — these aren't decorative; they're additional routing and validation signal Claude reads when deciding *how* to call a tool it has already selected, not just *whether* to call it.

---

## 2. Tool Input Schema Design

### 2.1 The Four-Part Anatomy of a Tool (Agent SDK)
Every custom tool is defined by exactly four parts: **name** (unique identifier), **description** (what it does, read by Claude to decide when to call it), **input schema** (the arguments Claude must supply), and a **handler** (the async function that actually runs). Precision on this four-part structure, and on the exact mechanism connecting each part to a specific point in the agentic loop, is a foundational exam expectation for this domain.

### 2.2 Required vs. Optional Parameters
**TypeScript (Zod)**: optional fields use `.default()` — this both marks the field optional *and* supplies the fallback value used when omitted.
**Python (dict schema)**: every key in the simple dict schema is treated as required — there's no native optional marker. To make a parameter optional, **omit it from the schema entirely**, mention it in the tool's description string instead, and read it defensively in the handler with `args.get("param", default_value)`. This is a real, documented asymmetry between the two SDK languages worth knowing precisely, since it's exactly the kind of detail a scenario question could test by showing Python code that incorrectly tries to mark a dict-schema key as optional.

### 2.3 Enums and Constraints
**TypeScript**: `z.enum([...])` natively expresses a constrained set of allowed values.
**Python**: the simple dict schema (`{"param": float}`) has **no enum equivalent** — when you need an enum, a range constraint, optional fields, or nested objects, you must pass the **full JSON Schema dict** to the `@tool` decorator instead of the shorthand dict-of-types form. Knowing this escape hatch exists — and exactly when you're forced to reach for it — is a concrete, testable fact.

### 2.4 Schema Design as Error Prevention
A tightly-constrained schema (enums instead of free-text strings where a fixed vocabulary exists, explicit types, required fields that are genuinely always needed) prevents a whole category of malformed calls before they ever reach the handler — the same "push correctness into the type system rather than handling it at runtime" instinct that shows up across the engineering reference docs, applied here to tool-input design specifically.

---

## 3. Tool Error Handling, Idempotency, and Partial Success

### 3.1 The Single Most Important Mechanic in This Domain
**How a handler reports an error determines whether the agentic loop continues or stops entirely — and this is a stark, binary distinction, not a matter of degree:**

| What the handler does | Result |
|---|---|
| Throws an uncaught exception | **The agent loop stops.** Claude never sees the error. The entire `query()` call fails. |
| Catches the error and returns `isError: true` (with the error described in the `content` array) | **The agent loop continues.** Claude sees the failure as data and can retry, try a different approach, or explain the failure to the user. |

**This is almost certainly one of the highest-yield single facts in this entire domain.** A tool handler that lets exceptions propagate uncaught is not "failing gracefully" — it's terminating the entire interaction. Production-quality tool design **always** wraps fallible operations (network calls, parsing, anything that can throw) in a try/catch (or try/except) specifically so failures become recoverable data the agent can reason about, rather than hard crashes.

### 3.2 Distinguishing a Real Error From "Odd-Looking Data"
The documented framing is precise: `isError: true` exists specifically so a failure is marked as a *failed call*, not interpreted as a strange-but-valid result. Returning a plain-text "Error: not found" string in the `content` array **without** setting `isError: true` risks Claude treating that string as legitimate data rather than recognizing a failure occurred — the flag, not the wording, is what carries the actual signal.

### 3.3 Idempotency — Why It Matters for Retry-Safety
Domain 1 §6.2 already covered retry logic generally. The tool-design-specific instance: if a tool might be called more than once with the same input (a retry after a transient failure, a duplicate call from agent confusion), **a non-idempotent tool — one where repeating the same call causes additional effects each time — turns an ordinary retry into a correctness bug** (e.g., "charge $10" repeated twice charges $20). The `idempotentHint` annotation (§3.5 below) is the documented, formal way to signal this property, but the underlying design discipline (favor idempotent operations, or build in deduplication, for anything retry-prone) matters regardless of whether you annotate it.

### 3.4 Partial Success Patterns
For operations that can partially succeed (e.g., a batch update where some items succeed and others fail), the documented best practice is to **return that partial outcome explicitly in the content**, rather than collapsing the whole call into a single success/failure — Claude can only react intelligently to a partial-success outcome ("17 of 20 updated, here are the 3 failures") if the tool result actually communicates that nuance, rather than reporting either a blanket success or a blanket `isError: true` that obscures the 17 that worked.

### 3.5 Tool Annotations — Formal, Documented Behavior Hints
Four boolean hint fields, passed as optional metadata when defining a tool:

| Annotation | Default | Meaning |
|---|---|---|
| `readOnlyHint` | `false` | Tool doesn't modify its environment — **this is the one with a real behavioral effect**: it controls whether Claude can batch/parallelize this call with other read-only tools. |
| `destructiveHint` | `true` | Tool may perform destructive updates. Informational only. |
| `idempotentHint` | `false` | Repeated calls with the same args have no additional effect. Informational only. |
| `openWorldHint` | `true` | Tool reaches systems outside your process. Informational only. |

**The critical exam-relevant caveat, stated explicitly in the documentation: annotations are metadata, not enforcement.** A tool marked `readOnlyHint: true` can still write to disk if that's what its handler actually does — the annotation doesn't constrain the handler's behavior, it only informs Claude's scheduling/reasoning. Keeping the annotation **accurate to what the handler actually does** is the developer's responsibility, not something the system verifies. This is a direct, narrower instance of the same "configuration/hints are advisory, hooks/permissions are enforced" principle from Domain 1 §6.4 and Domain 2 §2.5.

---

## 4. MCP Architecture — Servers, Clients, and the Three Primitives

### 4.1 What MCP Is and Why It Exists
The Model Context Protocol is an **open, standard protocol for AI-tool integrations** — its value proposition is that a tool/data-source built once as an MCP server can be connected to any MCP-compatible client (Claude Code, Claude.ai, other AI products) without bespoke integration work per client. This is the precise reason it's described as a universal connector rather than a Claude-specific feature.

### 4.2 The Three MCP Primitives
- **Tools**: callable functions Claude can invoke — the most common primitive, directly analogous to the custom tools covered in §1–§3.
- **Resources**: content Claude can *reference* (via `@server:protocol://resource/path` syntax in Claude Code) rather than actively call — listed and fetched the same way files are, automatically included as attachments when referenced.
- **Prompts**: server-exposed prompt templates that become invokable as commands (`/mcp__servername__promptname` in Claude Code), optionally accepting arguments.

**Exam-relevant distinction:** know that these are three genuinely different primitives with different invocation patterns (called vs. referenced vs. executed-as-a-command), not three names for the same underlying mechanism.

### 4.3 MCP Transport Mechanisms — Precise, Current State
| Transport | Status | Use case |
|---|---|---|
| **stdio** | Current, standard | Local servers running as a subprocess — direct system access, custom scripts |
| **HTTP** (`streamable-http`) | **Current, recommended** for remote servers | Cloud-based services; supports OAuth |
| **SSE** | **Deprecated** | Legacy remote servers; use HTTP instead where available |
| **WebSocket** | Current, narrow use case | Servers that need to **push** events to Claude unprompted (persistent bidirectional connection) — HTTP is preferred when the server only responds to requests, since HTTP supports OAuth and WebSocket does not |

**The single most exam-relevant precision point:** SSE is explicitly deprecated in current documentation — a scenario question presenting SSE as the modern, recommended choice for a new remote server integration should be recognized as describing outdated practice.

### 4.4 MCP Installation Scopes and Precedence
| Scope | Loads in | Shared with team | Stored in |
|---|---|---|---|
| **Local** (default) | Current project only | No | `~/.claude.json` |
| **Project** | Current project only | Yes, via `.mcp.json` in version control | `.mcp.json` |
| **User** | All projects | No | `~/.claude.json` |

**Precedence when the same server is defined in multiple places (highest to lowest): Local > Project > User > Plugin-provided > claude.ai connectors.** The entire server entry from the highest-precedence source is used — fields are not merged across scopes. Project-scoped servers from `.mcp.json` require explicit user approval before first use, for security reasons (an untrusted repo shouldn't be able to silently auto-connect an MCP server on your behalf).

### 4.5 Server Instructions and Tool Search — The Current Context-Management Architecture
**Tool search is enabled by default in current Claude Code**, and it fundamentally changes how MCP context cost works: **only tool names and server instructions load at session start; full tool schemas stay deferred until Claude actually needs them**, discovered via a search mechanism rather than loaded upfront. This means adding more MCP servers has minimal impact on context window usage by default — a significant, current architectural fact, and a direct contradiction of an older mental model where every connected server's full tool list loaded unconditionally.

**For MCP server authors, this raises the importance of server *instructions*** (a field distinct from any individual tool's description) — since instructions are what's visible *before* tool search even runs, they need to explain what category of tasks the server's tools handle, functioning similarly to how a skill's description routes invocation (Domain 2 §3).

**The exception**: `alwaysLoad: true` on a server (or `anthropic/alwaysLoad: true` on an individual tool's metadata) exempts it from deferral — every tool from that server loads into context at session start regardless of tool search settings. Documented guidance: reserve this for a small number of tools Claude genuinely needs on every turn, since each upfront-loaded tool is a permanent context cost for the rest of the session.

### 4.6 MCP Output Limits
A documented, concrete number: **the default warning threshold for any single MCP tool output is 10,000 tokens, with a default hard maximum of 25,000 tokens** (configurable via `MAX_MCP_OUTPUT_TOKENS`). Tools that legitimately need to return very large results (full database schemas, complete file trees) can declare a per-tool `anthropic/maxResultSizeChars` metadata value to raise their own threshold without requiring the environment variable to be globally raised — a real, designed escape hatch for the "this one tool's output is just inherently large" case, distinct from raising the limit for every tool indiscriminately.

---

## 5. MCP Security — Access Scoping, Authentication, Production Hardening

### 5.1 The Core Security Principle
**Verify you trust a server before connecting it — a server that fetches external content can expose you to prompt injection risk.** This is the documented, explicit warning, and it's a real, structural risk distinct from "the server might be buggy": a malicious or compromised MCP server's tool *results* are exactly the kind of untrusted content that can attempt to inject instructions into the agent's context (directly the same "instruction injected via untrusted content" risk pattern that shows up generally in agentic security thinking).

### 5.2 OAuth 2.0 Authentication
Claude Code auto-detects when a remote server needs authentication (a `401`/`403` response, or a `WWW-Authenticate` header pointing to an authorization server) and walks through a browser-based OAuth flow. Precise, current details worth knowing:
- Discovery checks **RFC 9728** (Protected Resource Metadata) first, falling back to **RFC 8414** (authorization server metadata).
- **`oauth.scopes`** lets an administrator pin the exact scopes requested, restricting a server to a security-team-approved subset even when the upstream server advertises broader scopes — the documented, supported mechanism for least-privilege scope restriction.
- For non-OAuth authentication schemes (Kerberos, short-lived tokens, internal SSO), **`headersHelper`** runs a command at connection time and merges its JSON output into request headers — the documented escape hatch for custom auth that doesn't fit the OAuth model.

### 5.3 Managed MCP Configuration — Organizational Control
For organizations, two complementary control mechanisms:
- **`managed-mcp.json`** — exclusive control: defines the *complete* fixed set of servers users can connect to.
- **Allowlists/denylists** (`allowedMcpServers`/`deniedMcpServers`) — policy-based control, matching servers by URL, command, or name, layered on top of whatever else is configured rather than replacing it entirely.

**The exam-relevant distinction**: exclusive control (a fixed, complete list) versus policy-based filtering (allow/deny rules applied to a broader, otherwise-open configuration) are different governance models suited to different organizational risk postures — a scenario describing "we want users to be able to add their own servers but block a specific known-risky one" maps to allow/denylist, not `managed-mcp.json`.

### 5.4 Tool Permission Control — `tools` vs. Allowed/Disallowed Lists
This is a precise, two-layer model worth getting exactly right (Agent SDK framing, but the underlying concept applies in Claude Code too):

| Mechanism | Layer | Effect |
|---|---|---|
| `tools: [...]` allowlist | **Availability** | Unlisted tools are removed from Claude's context entirely — Claude never even sees them as an option |
| `disallowedTools` | **Permission** | The tool **stays visible** in context, but every call to it is denied — Claude may waste a turn attempting it before the call is rejected |

**The documented, precise recommendation: prefer the `tools` availability list over `disallowedTools` when you want to restrict built-ins**, specifically because it prevents Claude from wasting effort attempting something that was always going to be denied — a small efficiency point that's also a real, testable distinction between two superficially-similar-looking mechanisms.

### 5.5 MCP Activity Auditing
Production hardening includes auditing MCP tool calls (which tools were called, by which server, with what arguments, attributable to which user/session) — the MCP-specific instance of the general observability/audit-logging discipline that applies to any tool-use system at scale, directly relevant to compliance-sensitive deployments.

---

## 6. The Built-In Toolset — Choosing Built-In vs. Custom

### 6.1 What Ships With Claude Code
Bash, Read, Edit, Write, Glob, Grep, WebFetch, WebSearch, Agent, AskUserQuestion, ExitPlanMode, and a handful of others — these are available without any configuration and form the substrate every other feature in this domain (skills, subagents, hooks) operates on top of.

### 6.2 Choosing Built-In vs. Custom — The Decision
- **Reach for a built-in** when the task is generic file/shell/web interaction that the built-in tools already cover well — building a custom "read this file" tool when `Read` already exists is needless duplication.
- **Reach for a custom tool (via the Agent SDK) or an MCP server** when the task requires domain-specific logic, a specific external API/database, or a capability genuinely outside what Bash/Read/Edit/etc. can express cleanly — a weather API call, a structured database query with business-logic validation, or anything where you want a tightly-scoped, narrowly-typed interface rather than "shell out via Bash and hope the command is right."
- **The boundary case worth naming explicitly**: Bash *can* technically call any API via `curl`, but a purpose-built tool with a typed schema, a documented description, and structured error handling (per §3) is more reliable and more auditable than an equivalent ad hoc Bash invocation — this is the real, substantive justification for building custom tools at all rather than just always reaching for Bash.

---

## 7. Quick-Reference Summary for This Domain

- **Error handling is binary and high-stakes**: uncaught throw = loop stops entirely; caught + `isError: true` = loop continues with the failure as data. This is the single highest-yield fact in the domain.
- **`tools` (availability) vs. `disallowedTools`/`allowedTools` (permission)**: removing from context vs. denying after Claude already tried — different layers, different costs.
- **SSE is deprecated**; HTTP is the current recommended remote transport; WebSocket is for servers that push unprompted.
- **MCP scope precedence**: Local > Project > User > Plugin > claude.ai connectors — whole entries win, no field merging.
- **Tool search is on by default**: server instructions matter more than ever, since they're what's visible before any tool schema loads; `alwaysLoad` is the narrow, deliberate exception.
- **Annotations are metadata, not enforcement** — `readOnlyHint` is the one with a real behavioral effect (parallelization); the rest are informational only, and none constrain what the handler actually does.

This domain carries 18% of the exam. §3.1 (the throw-vs-`isError` distinction) and §4.3 (current transport status, especially SSE's deprecation) are the two facts most likely to separate current, accurate knowledge from a stale or assumed mental model.
