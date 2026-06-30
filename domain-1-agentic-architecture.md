# CCA-F Domain 1: Agentic Architecture & Orchestration (27% of exam)

**Source note:** This doc is built directly from Anthropic's current Claude Code and Agent SDK documentation (verified live, June 2026), not from general AI-agent theory. Where the exam's framing is broader than Claude-specific tooling (e.g., "agents vs. workflows" as a general concept), that's flagged explicitly. This is the highest-weighted domain on the exam — worth the most study time per point available.

---

## 1. The Agentic Loop — The Foundational Mental Model

### 1.1 What the Loop Actually Is
Every Claude-based agent — whether it's Claude Code, an Agent SDK application, or a hand-rolled API integration — runs the same fundamental cycle: Claude receives a request, decides whether to use a tool or respond directly, and if it uses a tool, the result is appended to the conversation and sent back for the next iteration. The loop continues until Claude produces a final answer with no further tool calls needed.

**The two `stop_reason` values that matter most for exam purposes:**
- **`tool_use`** — Claude wants to call one or more tools. The harness must execute them and return results.
- **`end_turn`** — Claude has finished; no more tool calls are pending. This is the correct signal to terminate the loop.

### 1.2 The Critical Anti-Pattern the Exam Tests Directly
A recurring exam theme (confirmed by the official syllabus language: *"avoiding anti-patterns such as parsing natural language signals to determine loop termination, setting arbitrary iteration caps as the primary stopping mechanism, or checking for assistant text content as a completion indicator"*) is this exact failure mode:

**Wrong:** Scanning Claude's text output for phrases like "I'm done" or "Task complete" to decide whether to stop the loop. This is fragile — Claude's phrasing varies, and a model that says "I'm not done yet, let me also check X" inside an explanation would falsely trigger termination.

**Wrong:** Using a fixed iteration count (e.g., "stop after 10 tool calls") as the *primary* mechanism for ending the loop. This conflates a safety backstop with actual completion logic — a task might legitimately need 15 tool calls, or might finish correctly in 2.

**Right:** Check the structured `stop_reason` field programmatically. `tool_use` means continue and execute; `end_turn` means stop. An iteration cap is still good practice, but as a safety ceiling, not the decision mechanism.

### 1.3 Tool Execution and Context Accumulation
After Claude requests a tool call, the harness executes it (locally, via MCP, via a custom function) and returns the result as a new message in the conversation. This is what lets Claude reason about the *outcome* of its own action on the next turn — the model doesn't have any persistent memory between calls; everything it "knows" about what just happened is exactly and only what's in the appended tool result.

**Exam-relevant nuance:** this is why tool result design matters so much (see Domain 4) — if a tool's error message is vague, Claude's next decision in the loop is only as good as that vague signal.

### 1.4 Model-Driven Decisions vs. Pre-Configured Decision Trees
The exam syllabus explicitly distinguishes: *"model-driven decision-making (Claude reasons about which tool to call next based on context) and pre-configured decision trees or tool sequences."*

- **Model-driven**: Claude looks at the current state and decides, fresh, which tool (if any) to call next. This is what makes something genuinely "agentic" rather than a scripted workflow.
- **Pre-configured**: A fixed sequence (call tool A, then B, then C, always in that order) — this is a **workflow**, not an agent, even if Claude is technically making the individual tool calls. See §2 for the precise distinction.

The exam's framing implies you should be able to look at a described system and classify which one it is — and recognize that **most production systems are hybrids**, with model-driven reasoning inside a workflow-shaped skeleton (see §2.4).

---

## 2. Agents vs. Workflows vs. Conversational Systems

### 2.1 The Three-Way Distinction
- **Conversational system**: a chat interface with no tool use — Claude responds based on context alone. No agentic loop.
- **Workflow**: a predetermined sequence of steps, possibly each powered by an LLM call, but the *order and branching logic* is fixed by the developer in code, not decided by the model at runtime.
- **Agent**: the model itself decides, turn by turn, what to do next — which tool to call, whether to call one at all, when to stop — based on reasoning over the current state.

### 2.2 When Workflows Outperform Agents
This is explicitly an exam topic (lecture title: *"When Workflows Outperform Agents"*). The general principle: **if the task's steps and branching logic are fully knowable in advance, a workflow is more reliable, more predictable in cost, and easier to debug than letting the model decide the path each time.** Agentic flexibility is valuable specifically when the path *can't* be fully predetermined — when the right next step depends on information only discoverable during execution (what a file actually contains, what an API actually returns, what edge case is actually present).

**The calibrated exam answer to "should this be an agent or a workflow":** name the specific source of unpredictability that justifies agentic reasoning, or name the absence of one that justifies a workflow. "This needs to be agentic because the right remediation depends on which of several possible error types the log actually shows, which we can't know until runtime" is a complete, correct-shaped answer. "We made it agentic because agents are more advanced" is not.

### 2.3 Choosing the Right Architecture — A Decision Framework
Given a scenario, the exam expects you to map requirements to architecture:

| Signal in the scenario | Points toward |
|---|---|
| Fixed, known sequence of steps; compliance/audit requirements for predictable execution | Workflow |
| The right next action depends on runtime-discovered information | Agent |
| Pure Q&A, no external state to act on | Conversational, no tools needed |
| Need for cost/latency predictability | Workflow (or agent with a hard iteration/turn cap) |
| Task naturally decomposes into independent, parallelizable subtasks | Multi-agent (see §4) |

### 2.4 The Hybrid Reality
In practice, most production Claude Code and Agent SDK systems are **workflows that call agentic subroutines**, or **agents constrained by workflow-like guardrails** (hooks enforcing deterministic checks at specific points — see Domain 2). The exam's framing rewards recognizing this nuance rather than treating "agent" and "workflow" as mutually exclusive categories.

---

## 3. Task Analysis and Decomposition

### 3.1 Decomposition Patterns
The exam syllabus names three explicitly: **hierarchical, sequential, and parallel.**
- **Hierarchical**: a high-level goal is broken into sub-goals, each of which may itself be broken down further — naturally maps onto an orchestrator delegating to subagents, which may delegate to their own nested subagents (Claude Code supports nested subagent spawning up to a fixed depth limit — see §4.3).
- **Sequential**: subtask B depends on the output of subtask A; must run in order. Maps onto a pipeline topology (§4.2).
- **Parallel**: subtasks are independent and can run simultaneously. Maps onto fan-out/fan-in (§3.3) or parallel subagent research (§4.2).

### 3.2 Sequential Pipelines — Design and Tradeoffs
A sequential pipeline trades latency (each stage waits for the prior one) for simplicity and clear data lineage (you always know exactly what fed into stage N). The tradeoff to name explicitly on the exam: a sequential pipeline is the wrong choice when stages are genuinely independent, since it adds latency for no correctness benefit.

### 3.3 Parallel Execution — Fan-Out, Fan-In, and Synchronization
**Fan-out**: dispatch the same or related work to multiple workers simultaneously (e.g., "research the authentication, database, and API modules in parallel using separate subagents" — a real example from Anthropic's own subagent documentation).
**Fan-in**: the results from parallel workers are collected and synthesized by a single coordinating agent.
**Synchronization**: the orchestrator must wait for all (or a defined subset of) parallel workers to complete before fan-in can proceed — a real design decision, since waiting for a straggler can dominate total latency.

**Exam-relevant constraint:** Anthropic's own documentation explicitly warns that running many subagents that each return detailed results can consume significant context in the parent conversation — fan-out is not free, and the exam expects you to know this cost exists, not just that the pattern exists.

### 3.4 Adaptive Planning, Replanning, and Ambiguity Handling
An agent's initial plan is not necessarily final. As execution proceeds and new information surfaces (a tool call fails, a file doesn't contain what was expected, a dependency turns out to be missing), the agent should be able to revise its plan rather than rigidly executing the original one. **Plan Mode** in Claude Code is the concrete, named feature most directly relevant here: it lets Claude research and present a plan *before* making any changes, which the user can approve, edit, or send back for re-planning — explicitly supporting an iterate-before-execute pattern rather than one-shot planning.

**Handling incomplete specifications:** when a task is under-specified, the exam's implied best practice (consistent with Claude Code's own `AskUserQuestion` tool and elicitation patterns) is to surface the ambiguity explicitly — asking a structured clarifying question — rather than silently guessing and proceeding, especially for consequential or hard-to-reverse actions.

---

## 4. The Orchestrator-Subagent Model

### 4.1 Why Delegate at All — The Two Real Reasons
Anthropic's own subagent documentation gives this precisely: delegate when **(1)** "a side task would flood your main conversation with search results, logs, or file contents you won't reference again" (context preservation), or **(2)** "you keep spawning the same kind of worker with the same instructions" (reuse). Everything else about subagent design follows from these two motivating reasons.

### 4.2 Topology Patterns — Hub-and-Spoke, Pipeline, Peer-to-Peer
- **Hub-and-spoke**: one orchestrator dispatches to multiple independent workers and synthesizes their results. Maps directly onto Claude Code's standard subagent model — the main conversation is the hub, each subagent invocation is a spoke, and subagents do not talk to each other directly.
- **Pipeline**: output of one agent becomes input to the next, in sequence — "use the code-reviewer subagent to find performance issues, then use the optimizer subagent to fix them" is the documented Claude Code example of this pattern.
- **Peer-to-peer**: agents communicate directly with each other rather than exclusively through a central orchestrator. This is the shape **Agent Teams** introduces (a distinct, more advanced Claude Code feature from standard subagents) — teammates can be talked to directly, assigned and claim tasks, and the lead can require plan approval from teammates, which is a materially different communication structure than the strict hub-and-spoke of ordinary subagents.

**Selecting the right topology — the exam-relevant judgment:** hub-and-spoke is the default and simplest; reach for pipeline when there's a genuine sequential dependency; reach for peer-to-peer / agent teams specifically when workers need to coordinate directly or when you need sustained parallelism beyond what a single orchestrator's context budget can absorb from fan-in (Anthropic's own docs note: "For tasks that need sustained parallelism or exceed your context window, agent teams give each worker its own independent context").

### 4.3 Context Isolation — The Single Most Important Subagent Property
**A subagent starts with a fresh, isolated context window. It does not see your conversation history, the skills you've already invoked, or the files Claude has already read.** This is the precise, verified mechanism — not an approximation. What a (non-fork) subagent's initial context actually contains:
- Its own system prompt (the subagent's defined prompt, plus environment details Claude Code appends) — **not** the full Claude Code system prompt.
- The delegation task message Claude composed when handing off.
- CLAUDE.md and the full memory hierarchy — **except** the built-in **Explore** and **Plan** subagents, which deliberately skip this to stay fast and cheap.
- A git status snapshot from the start of the parent session (again, except Explore/Plan).
- Any skills explicitly preloaded via the subagent's `skills` frontmatter field.

**Only the subagent's final text response (plus a small metadata trailer) returns to the parent conversation.** This is the entire mechanism behind the context-preservation benefit — a subagent might read 6,000 tokens of files and return a 400-token summary, and only that 400 tokens touches the orchestrator's context.

### 4.4 Built-In Subagents — Know These Three Cold
| Subagent | Model | Tools | Purpose |
|---|---|---|---|
| **Explore** | Haiku (fast, low-latency) | Read-only (no Write/Edit) | Codebase search and discovery without modification; skips CLAUDE.md and git status for speed |
| **Plan** | Inherits from main conversation | Read-only | Research during Plan Mode, before a plan is presented; also skips CLAUDE.md and git status |
| **general-purpose** | Inherits from main conversation | All tools | Complex multi-step tasks requiring both exploration and modification |

**The exam-relevant distinction to internalize:** Explore and Plan are deliberately *cheaper and faster* by skipping context most subagents load — this is a real architectural tradeoff (speed/cost vs. completeness of context) you should be able to name, not just a list of three names to memorize.

### 4.5 Subagent Scope and Authority — Tool and Permission Restriction
A subagent's capabilities are controlled via:
- **`tools` (allowlist)** or **`disallowedTools` (denylist)** in its frontmatter. If both are set, `disallowedTools` is applied first, then `tools` is resolved against what remains.
- **`permissionMode`** — `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, or `plan`. Critically: if the **parent** session uses `bypassPermissions` or `acceptEdits`, that takes precedence and the subagent's own `permissionMode` cannot override it — permission restriction flows downward and cannot be loosened by a child.
- **`mcpServers`** scoped specifically to that subagent — letting you give a subagent access to tools the main conversation doesn't have, *and* keep an MCP server's tool definitions out of the main conversation's context entirely if only a subagent needs them.

### 4.6 Spawning Limits — Nested Subagents
As of the current Claude Code version, a subagent can spawn its own nested subagents, up to a **fixed depth limit** (a subagent at depth five does not receive the Agent tool and cannot spawn further; this limit is not configurable). This matters for exam scenario questions about deeply hierarchical decomposition — there is a hard architectural ceiling, not infinite recursion.

---

## 5. Agent Communication, Handoffs, and State

### 5.1 Designing Agent-to-Agent Message Schemas
Since a subagent receives only a delegation task message (not the full parent conversation), **the quality of that handoff message is the single biggest lever on subagent success.** A vague delegation ("look into the auth stuff") forces the subagent to spend its own context rediscovering what the orchestrator already knew. A precise one (the specific bug, the specific files already identified, the specific constraint to respect) lets the subagent start working immediately.

**A specific, documented gotcha worth knowing:** Explore and Plan skip CLAUDE.md, so any project-specific rule that must reach them (e.g., "ignore the `vendor/` directory") has to be **restated explicitly in the delegation prompt** — it won't arrive any other way.

### 5.2 Handoff Protocols and Continuity
- **Resuming a subagent**: rather than starting a fresh instance, Claude can resume an existing subagent (by agent ID) to continue work with full prior context — but the built-in Explore and Plan agents are one-shot and **cannot** be resumed (they return no agent ID at all). Use `general-purpose` or a custom subagent when continuity matters.
- **Forking**: a fundamentally different mechanism from a named subagent — a fork inherits the **entire** parent conversation history rather than starting fresh, trading context isolation for continuity. Forks share the parent's prompt cache (cheaper for tasks needing the same context) but cannot spawn further forks themselves.

### 5.3 In-Context State vs. External Memory
- **In-context state**: everything currently in the conversation's token window — cheap to access, but vanishes on compaction (mostly) and doesn't survive across sessions.
- **External memory**: persisted to disk, survives across sessions. Claude Code's concrete mechanism is the **`memory` frontmatter field** on a subagent (`user`, `project`, or `local` scope), giving a subagent a durable directory it can read/write to accumulate knowledge (patterns, recurring issues, architectural decisions) across many separate invocations over time — genuinely different from simply having a long context window.

**Exam framing:** know that `project` scope is the documented recommended default (shareable via version control), `user` scope is for cross-project learnings, and `local` is project-specific but deliberately excluded from version control.

### 5.4 Session Continuity Across Turns and Failures
At the main-conversation level, sessions can be resumed (`--resume`, `--continue`, or `/resume`), and subagent transcripts persist independently of the main conversation — meaning **main conversation compaction does not affect subagent transcripts**, since they're stored in separate files entirely. This separation is a real, exam-relevant architectural fact: state durability for subagents is decoupled from the parent's own context-management lifecycle.

---

## 6. Error Handling, Fallback, and Escalation

### 6.1 Error Classification — Tool, Reasoning, and Environment Errors
The exam syllabus names this three-way split explicitly:
- **Tool errors**: the tool itself failed (an API call timed out, a file didn't exist, a permission was denied).
- **Reasoning errors**: the model's *decision* was wrong even though execution succeeded (it picked the wrong tool, misread an instruction, or drew an incorrect inference from a correct result).
- **Environment errors**: the surrounding system failed in a way unrelated to either the tool call or the model's reasoning (network outage, MCP server disconnected, rate limit hit).

**Why the distinction matters for design:** each category implies a different fix. A tool error often warrants a retry (see §6.2). A reasoning error warrants better prompting, tighter tool schemas, or an independent review step (see the ML/AI document's §4.6 on multi-instance review for the general principle) — retrying the same call with the same flawed reasoning won't help. An environment error often warrants a circuit-breaker/fallback rather than an immediate retry, since the underlying system may need time to recover.

### 6.2 Retry Logic — When to Retry, When to Abort
Claude Code's own MCP transport behavior is a concrete, documented model worth knowing precisely: for HTTP/SSE servers, reconnection uses **exponential backoff — up to five attempts, starting at a one-second delay and doubling each time** — and **authentication and not-found errors are explicitly not retried**, because they require a configuration change, not a transient-failure recovery. This is the general principle the exam wants you to apply: retry transient failures (5xx, timeouts, connection-refused); don't retry failures that need a human or a config change to fix, since retrying just wastes time and obscures the real problem.

### 6.3 Fallback Chains and Graceful Degradation
A fallback chain is an explicit, designed sequence of "if this fails, try this instead" — not an accident of retry logic. Claude Code's model-fallback feature is a direct, concrete example: configurable fallback model chains that activate automatically under specific triggers (e.g., the primary model being overloaded), with the system able to report what triggered the fallback. The exam-relevant generalization: **graceful degradation means the system continues to provide *some* value at reduced capability, rather than failing completely** — and good design makes the degraded state visible/loggable, not silent.

### 6.4 Why Prompts Are Not Sufficient for High-Stakes Paths
This is an explicit exam lecture title, and the underlying principle is one of the most important ideas in the whole agentic-architecture domain: **a system prompt instruction is a strong suggestion to the model, not an enforced guarantee.** For anything genuinely high-stakes (irreversible actions, financial transactions, destructive operations), the correct architecture uses **programmatic enforcement** — hooks that can deny a tool call outright (`PreToolUse` returning `permissionDecision: "deny"`), permission rules, or sandboxing — layered *on top of* prompted guidance, not as a substitute for it. A prompt saying "never delete production data" is advisory; a `PreToolUse` hook that returns `deny` for any `rm` command matching a protected path pattern is enforced. **The exam will almost certainly test whether you reach for the prompt-only answer or the programmatic-enforcement answer when the scenario describes something consequential.**

### 6.5 Escalation Trigger Design — Valid vs. Invalid Patterns
A well-designed escalation trigger fires on a **specific, detectable condition** tied to genuine risk or genuine uncertainty — e.g., a permission system's `"ask"` decision surfacing a confirmation dialog specifically when a hook can't confidently allow or deny, or `AskUserQuestion` firing when the model has identified a genuine ambiguity requiring user input. An invalid pattern escalates either too eagerly (interrupting for things that didn't need a human) or not eagerly enough (silently proceeding through something that should have paused). The calibrated, exam-correct framing: escalation should be reserved for genuine decision points — not used as a substitute for either good initial design (handling foreseeable cases without asking) or programmatic enforcement (handling unacceptable cases without asking either, by simply denying them).

### 6.6 Interruption Points and Review Workflows
Concrete, documented Claude Code mechanisms for human-in-the-loop review: **Plan Mode** (review and approve a plan before any execution happens), **checkpoints** (rewind to a prior state if something goes wrong, providing an undo path rather than requiring perfect foresight), and **`ExitPlanMode`** as a tool call that explicitly requires user approval before Claude leaves read-only planning and starts taking action. The exam's implied best practice: place interruption points *before* hard-to-reverse actions, not after, and prefer a designed review gate over relying on the user noticing a problem after the fact.

---

## 7. Multi-Agent Architecture — Evaluation and Design Judgment

### 7.1 The Core Question the Exam Asks: "Should This Be Multi-Agent at All?"
Multi-agent architecture has a real cost (more context overhead, more coordination complexity, more failure surface) and should be justified by a specific need: genuine parallelizability, the need for context isolation between unrelated subtasks, or the need for specialized tool/permission scoping per worker. The "Evaluate multi-agent architecture in the context of agentic architecture & orchestration" objective in the official syllabus implies scenario-based questions where the correct answer is sometimes **"a single agent would have been sufficient here — multi-agent adds coordination overhead without a corresponding benefit."**

### 7.2 Agent Teams vs. Subagents — A Precise, Exam-Relevant Distinction
These are genuinely different Claude Code features, not synonyms:
- **Subagents**: hub-and-spoke, isolated context, results return only to the orchestrator, no direct inter-agent communication.
- **Agent Teams**: teammates can be talked to directly, can be assigned and can claim tasks themselves, and a lead can require plan approval from teammates before they proceed — a fundamentally more peer-oriented structure. Best practices documented by Anthropic include giving teammates enough context, choosing an appropriate team size, sizing tasks appropriately, and explicitly avoiding file conflicts between simultaneously-working teammates (a real, documented failure mode).

### 7.3 Cost and Token Awareness in Multi-Agent Design
Multi-agent systems multiply token cost in ways a single-agent system doesn't — Anthropic's own documentation specifically calls out "Agent team token costs" as a distinct cost-management concern, separate from ordinary single-session usage. The exam-relevant takeaway: a multi-agent design proposal that doesn't address its own cost multiplication is an incomplete answer.

---

## 8. Quick-Reference Summary for This Domain

- **Loop termination**: `stop_reason` field, never text-parsing, never iteration-count-as-primary-mechanism.
- **Agent vs. workflow**: justified by a specific source of runtime unpredictability, not "agents are more advanced."
- **Subagent's #1 property**: fresh, isolated context window; only final text + metadata returns to parent.
- **Explore/Plan**: Haiku-or-inherited, read-only, skip CLAUDE.md and git status, cannot be resumed.
- **High-stakes paths**: programmatic enforcement (hooks, permissions) layered on top of prompts, never prompts alone.
- **Error type → fix**: tool error → retry (if transient) or surface; reasoning error → independent review, not blind retry; environment error → fallback/circuit-breaker.
- **Multi-agent justification**: parallelizability, context isolation, or specialized scoping — and always with an explicit cost story.

This domain carries 27% of the exam — the single largest share. If your study time is constrained, §1 (the loop and its anti-patterns), §4.3 (context isolation), and §6.4 (programmatic enforcement vs. prompts) are the three highest-density sections to over-prepare relative to the rest.
