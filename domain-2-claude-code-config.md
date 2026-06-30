# CCA-F Domain 2: Claude Code Configuration & Workflows (20% of exam)

**Source note:** Built directly from Anthropic's current Claude Code documentation (verified live, June 2026). This domain has had genuinely significant recent architecture changes — most importantly, **custom slash commands have been merged into the Skills system** — and this doc reflects that current state rather than treating them as separate, parallel features the way some older third-party study material may.

---

## 1. Claude Code Architecture — The Tool System and Execution Model

### 1.1 The Core Mental Model
Claude Code is the Agent SDK's agentic loop (Domain 1, §1) wrapped in a CLI/IDE/web interface, with a specific set of built-in tools (Bash, Read, Edit, Write, Glob, Grep, WebFetch, WebSearch, Agent, AskUserQuestion, ExitPlanMode, and a few others) plus whatever MCP tools and skills are configured. Every configuration mechanism covered in this domain — CLAUDE.md, settings, hooks, skills, subagents — is a way of shaping *what's in context* and *what's allowed to happen* around that same underlying loop.

### 1.2 What Loads at Session Start — The Concrete, Verified Order
Before you type anything, a session loads (in roughly this order): the system prompt, **auto memory** (first 200 lines or 25KB of `MEMORY.md`), environment info, MCP tool names (schemas deferred by default — see Domain 4), skill descriptions, `~/.claude/CLAUDE.md`, then project `CLAUDE.md`. This is verified, current behavior, not an approximation — and it's exam-relevant because **CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself**, which is precisely why CLAUDE.md instructions are advisory (Claude tries to follow them) rather than enforced (the way a hook or permission rule is).

---

## 2. CLAUDE.md — Hierarchy, Precedence, and Effective Writing

### 2.1 The Four Scopes, in Load Order (Broadest to Most Specific)
| Scope | Location | Shared with |
|---|---|---|
| **Managed policy** | OS-specific system path (e.g., `/etc/claude-code/CLAUDE.md` on Linux) | Entire organization; cannot be excluded by users |
| **User** | `~/.claude/CLAUDE.md` | Just you, all your projects |
| **Project** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team, via version control |
| **Local** | `./CLAUDE.local.md` | Just you, this project only (gitignored) |

**The precise loading mechanic worth knowing exactly:** Claude Code walks up the directory tree from your working directory, loading every `CLAUDE.md`/`CLAUDE.local.md` it finds. **All discovered files are concatenated — none override each other.** Within the tree, content is ordered root-to-working-directory (so instructions closer to where you launched are read *last*, i.e., most recently before your prompt), and within each directory, `CLAUDE.local.md` is appended after `CLAUDE.md`. Nested subdirectory `CLAUDE.md` files load on-demand, only when Claude reads a file in that subdirectory — not at launch.

### 2.2 `.claude/rules/` — Path-Scoped Instructions
For larger projects, instructions can be split into `.claude/rules/*.md` files. **Rules without a `paths` frontmatter field load unconditionally**, same priority as the project CLAUDE.md. **Rules with `paths` frontmatter load conditionally** — only when Claude reads a file matching the glob pattern. This is a direct, real lever for context efficiency: a `src/api/**/*.ts`-scoped rule about API conventions costs zero tokens until Claude is actually touching API code.

### 2.3 `AGENTS.md` Interop
Claude Code reads `CLAUDE.md`, **not** `AGENTS.md`. If a repo already standardizes on `AGENTS.md` for other coding agents, the documented pattern is a `CLAUDE.md` that imports it (`@AGENTS.md`) and appends Claude-specific instructions below — or a symlink, if no Claude-specific content is needed (though symlinks require elevated privileges on Windows, so `@AGENTS.md` import is the more portable choice).

### 2.4 Writing Effective Instructions — The Concrete Guidance
- **Size**: target under 200 lines per file. Longer files cost more context *and* reduce adherence — both costs are real and documented, not just a style preference.
- **Specificity**: "Use 2-space indentation" over "Format code properly"; "Run `npm test` before committing" over "Test your changes." Vague instructions are unreliable instructions, because CLAUDE.md is context, not enforced configuration.
- **Consistency**: contradictory instructions across CLAUDE.md/rules files cause Claude to "pick one arbitrarily" — there's no defined tiebreak for conflicting *content* (as opposed to precedence between files, which *is* defined).
- **Imports**: `@path/to/file` syntax, max depth of 4 hops, resolved relative to the importing file (not the working directory). Imported content still loads at launch — imports help organization, **not** context cost.

### 2.5 The Critical Exam Distinction: CLAUDE.md vs. Settings/Hooks
This is one of the most important distinctions in the entire domain, and it's stated explicitly in Anthropic's own documentation: **"Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer."** If a scenario requires something to happen *every time*, regardless of the model's judgment (block a specific command, run a linter after every edit, enforce a compliance rule) — the correct answer is a hook or a permission rule, not a CLAUDE.md instruction, no matter how emphatically worded. This is the Domain 2 instance of the same principle Domain 1's §6.4 establishes for high-stakes actions generally.

### 2.6 What Survives `/compact`
**Project-root CLAUDE.md is automatically re-read from disk and re-injected after compaction.** Nested subdirectory CLAUDE.md files are **not** automatically re-injected — they reload only the next time Claude reads a file in that subdirectory. A documented, real consequence: if you give Claude an instruction only in conversation (not written to any CLAUDE.md), it can be lost after compaction. The fix is to write durable instructions to CLAUDE.md, not just say them.

### 2.7 Auto Memory — The Other Persistence Mechanism
Distinct from CLAUDE.md: **auto memory is written by Claude itself**, not you, based on corrections and patterns it notices are worth remembering. Stored at `~/.claude/projects/<project>/memory/`, with a `MEMORY.md` index (first 200 lines/25KB loaded every session) plus optional topic files Claude reads on demand. **Exam-relevant distinction table:**

| | CLAUDE.md | Auto memory |
|---|---|---|
| Author | You | Claude |
| Content | Instructions/rules | Learnings/patterns Claude discovered |
| Loaded into context | Every session, in full | Every session, but only the first 200 lines/25KB |

---

## 3. Skills — The Unified Mechanism (Including What Used to Be "Commands")

### 3.1 The Single Most Important Current-State Fact for This Domain
**Custom slash commands have been merged into skills.** A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and behave the same way — old `.claude/commands/` files keep working, but skills are now the recommended, superset mechanism (supporting directories of supporting files, invocation-control frontmatter, and subagent execution that plain commands don't). **If exam material frames "commands" and "skills" as two separate, parallel topics, treat that framing as the historical state of the product — the current architecture is one unified system with two file-location patterns that produce the same result.**

### 3.2 The SKILL.md Anatomy
A skill is a directory containing `SKILL.md` (required) plus optional supporting files (templates, examples, scripts, reference docs) that are **not** loaded into context until the skill specifically references and needs them. This is the entire point of the supporting-files pattern: large reference material costs near-zero tokens until actually needed, unlike CLAUDE.md content, which is always loaded.

### 3.3 Where Skills Live and Precedence
| Location | Scope |
|---|---|
| Enterprise (managed settings) | All org users |
| `~/.claude/skills/<name>/` | All your projects |
| `.claude/skills/<name>/` | This project |
| `<plugin>/skills/<name>/` | Wherever the plugin is enabled |

When names collide: enterprise overrides personal, personal overrides project, and a skill at any of these overrides a same-named **bundled** skill. Plugin skills are namespaced (`plugin-name:skill-name`) and can't collide with anything. **If a skill and a command share a name, the skill wins.**

### 3.4 Controlling Who Invokes a Skill — A Frequently-Tested Distinction
Two frontmatter fields, with genuinely different effects:
- **`disable-model-invocation: true`** — only **you** can invoke it (via `/name`); Claude never triggers it automatically. Use for anything with a side effect you want to control the timing of — `/deploy`, `/commit`, `/send-slack-message`. **Critically, this also removes the skill's description from context entirely**, so it costs zero tokens until invoked.
- **`user-invocable: false`** — only **Claude** can invoke it; hidden from your `/` menu. Use for background knowledge (e.g., "how does our legacy system work") that isn't a meaningful manual action.

**The exam-relevant trap:** these are not opposites of a single toggle — a skill could theoretically have both set to restrict it from invocation by anyone except automatic matching in specific cases, and the default (neither set) allows both you and Claude to invoke it. Know the table precisely: default = both can invoke, description always in context; `disable-model-invocation` = only you, description **not** in context; `user-invocable: false` = only Claude, description still in context.

### 3.5 `context: fork` — Running a Skill in a Subagent
Setting `context: fork` in a skill's frontmatter runs the skill's content as the prompt for a subagent (default `general-purpose`, or a named agent via the `agent` field) rather than inline in the main conversation. **This is the direct bridge between Domain 1's subagent material and Domain 2's skills material**, and the exam will likely test the *inverse relationship* table:

| | System prompt comes from | Task comes from | Also loads |
|---|---|---|---|
| Skill with `context: fork` | The agent type | SKILL.md content | CLAUDE.md (unless agent is Explore/Plan) |
| Subagent with `skills:` field | Subagent's own markdown body | Claude's delegation message | Preloaded skills + CLAUDE.md |

A documented warning worth remembering precisely: `context: fork` only makes sense for skills with an explicit, actionable task — a skill that's just reference guidelines ("use these API conventions") forked into a subagent gives that subagent nothing to actually do.

### 3.6 Dynamic Context Injection — The `!`command`` Syntax
A skill can run a shell command *before* Claude ever sees the prompt, with the output substituted directly into the skill content. This is preprocessing, not something Claude executes — Claude only sees the final, rendered result. **Exam-relevant precision:** this only triggers when `!` appears at the start of a line or right after whitespace; `KEY=!`cmd`` does **not** trigger it, since `!` follows another character there.

### 3.7 Argument Passing
`$ARGUMENTS` (the full string), `$ARGUMENTS[N]`/`$N` (positional, 0-indexed), and `$name` (named, via the `arguments` frontmatter field, mapped positionally) are the three substitution patterns. If a skill is invoked with arguments but doesn't reference `$ARGUMENTS` anywhere, Claude Code appends `ARGUMENTS: <value>` automatically so the information isn't silently dropped.

### 3.8 Skill Content Lifecycle and Compaction
Once invoked, a skill's rendered content enters the conversation as a message and **is not re-read on later turns** — meaning a skill should write standing instructions ("always do X for the rest of this task"), not one-time-only steps that assume it'll be consulted again. After auto-compaction, the **most recent invocation of each skill is re-attached** (first 5,000 tokens, shared 25,000-token budget across all re-attached skills, filled starting from most-recently-invoked) — meaning skills invoked early in a long session can be dropped entirely after compaction if many other skills were invoked afterward.

---

## 4. Subagents in Claude Code — Configuration Recap (Cross-Reference to Domain 1)

Domain 1 §4–§5 covers subagent *architecture and orchestration* in depth. This domain's overlapping exam objective ("Subagents in Claude Code — configuration, scope, and delegation") is the *mechanics* layer: where files live (`.claude/agents/` project-scoped, `~/.claude/agents/` user-scoped, with project taking precedence on name collision when both exist — wait, precisely: **managed settings > `--agents` CLI flag > `.claude/agents/` > `~/.claude/agents/` > plugin agents**, a five-level precedence order worth memorizing exactly for the exam), the YAML frontmatter schema (`name`, `description` required; `tools`, `model`, `permissionMode`, `skills`, `mcpServers`, `hooks`, `memory`, `isolation`, `color`, and others optional), and the `/agents` command as the recommended creation/management interface.

---

## 5. Hooks — Lifecycle Automation (Cross-Reference, Configuration Focus)

Domain 1 uses hooks as the *mechanism for programmatic enforcement* (§6.4). This domain's exam objective ("hooks — lifecycle events, implementation, and exit code conventions") is about the configuration mechanics themselves:

### 5.1 The Configuration Shape
Three levels of nesting: an **event** (e.g., `PreToolUse`), a **matcher group** (filters which calls trigger it — by tool name, by `if` pattern using permission-rule syntax), and one or more **hook handlers** (`command`, `http`, `mcp_tool`, `prompt`, or `agent` type).

### 5.2 Exit Code Conventions — The Single Most Testable Mechanic
- **Exit 0**: success. Claude Code parses stdout for JSON output fields.
- **Exit 2**: blocking error. Stderr is fed back to Claude as the reason; the specific effect depends on the event (`PreToolUse` blocks the tool call; `UserPromptSubmit` rejects the prompt; `Stop` prevents Claude from stopping).
- **Any other non-zero exit**: non-blocking error for most events — **this is a common, real-world mistake to watch for**: Claude Code treats exit code 1 as *non-blocking*, even though it's the conventional Unix failure code. A hook meant to enforce policy must use exit code 2 specifically, not just "any failure."

### 5.3 Hook Locations and Scope
| Location | Scope | Shareable |
|---|---|---|
| `~/.claude/settings.json` | All your projects | No |
| `.claude/settings.json` | This project | Yes, committed |
| `.claude/settings.local.json` | This project | No, gitignored |
| Managed policy settings | Organization-wide | Yes, admin-controlled |
| Plugin `hooks/hooks.json` | While plugin enabled | Yes, bundled |
| Skill/agent frontmatter | While that component is active | Yes, in the component file |

### 5.4 Useful Hook Patterns Worth Having Ready for Scenario Questions
- **Auto-format after edits**: `PostToolUse` matching `Write|Edit`, running a formatter.
- **Block destructive commands**: `PreToolUse` matching `Bash`, with an `if` pattern like `Bash(rm *)`, returning `permissionDecision: "deny"`.
- **Re-inject context after compaction**: `PostCompact` or `SessionStart` with the `compact` matcher, re-adding information that doesn't survive compaction automatically (Domain 2 §2.6's nested-CLAUDE.md gap is a direct, real use case for this).
- **Notify on idle/need-for-input**: the `Notification` event firing on `permission_prompt` or `idle_prompt`.

---

## 6. Plan Mode — Safe Analysis Before Action

### 6.1 What It Is, Precisely
Plan Mode restricts Claude to read-only tools while it researches (often delegating to the **Plan** built-in subagent — see Domain 1 §4.4) and culminates in calling `ExitPlanMode`, a tool that **requires explicit user approval** before Claude can leave the read-only state and start making changes. This is the concrete Claude Code feature instantiating Domain 1's "interruption points before hard-to-reverse actions" principle.

### 6.2 When to Use It
Documented guidance: use Plan Mode for complex refactors, or any change where you want to review the approach before any file is touched. It can be configured as the default mode for a project (rather than something invoked ad hoc), which is the right call for codebases where unreviewed changes are a genuine risk.

### 6.3 Iterating on a Plan
A presented plan isn't a one-shot accept/reject — it can be approved, edited, or sent back for re-planning, directly supporting Domain 1's §3.4 adaptive-planning principle at the product-feature level.

---

## 7. CI/CD Integration — Non-Interactive Mode and the `-p` Flag

### 7.1 The Core Mechanism
`claude -p "<prompt>"` runs Claude Code non-interactively — the same Agent SDK loop, same tools, but no interactive terminal session. This is the literal foundation of every CI/CD integration (GitHub Actions, GitLab CI/CD, build scripts).

### 7.2 `--bare` — The CI-Specific Flag Worth Knowing Precisely
**`--bare` skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md** — meaning a CI run gets *exactly* what you explicitly pass via flags, with no dependency on whatever happens to be configured in a teammate's `~/.claude` or a project's `.mcp.json`. This is the documented, recommended mode for scripted/CI calls specifically because it guarantees the same result on every machine — a real, exam-relevant reliability property, and Anthropic's own docs note it will become the default for `-p` in a future release.

### 7.3 Structured Output for Scripted Consumption
`--output-format json` returns structured metadata (session ID, cost, usage) with the text result in a `result` field. Combined with `--json-schema`, you get a `structured_output` field conforming to a JSON Schema you define — the direct, CLI-level mechanism for Domain 3's structured-output concerns, just invoked from a build pipeline instead of the API directly.

### 7.4 Tool Auto-Approval for Non-Interactive Runs
Since there's no human to approve a permission prompt in CI, `--allowedTools "Bash,Read,Edit"` (or a permission mode like `acceptEdits` or the stricter `dontAsk`, which denies anything not explicitly allowed — the right default for locked-down CI) is required, or the run will abort the moment Claude attempts something not pre-approved. **`dontAsk` is the documented choice for locked-down CI specifically** — know this by name as the answer to "which permission mode for an untrusted/automated CI run."

### 7.5 Cost Tracking in Scripted Contexts
`--output-format json` responses include `total_cost_usd` and a per-model cost breakdown specifically so scripted callers can track spend per invocation without checking a separate dashboard — a real, designed feature for the exact use case of running Claude in a budget-conscious automated pipeline.

---

## 8. Quick-Reference Summary for This Domain

- **CLAUDE.md precedence**: managed > user > project > local, but files are **concatenated**, not overridden — content conflicts have no defined tiebreak, location precedence does.
- **CLAUDE.md vs. hooks/settings**: CLAUDE.md shapes behavior; hooks/settings enforce it regardless of what Claude decides. Know which one a scenario actually needs.
- **Commands = skills now.** Don't treat them as two separate systems on the exam.
- **`disable-model-invocation` vs. `user-invocable: false`**: the first blocks Claude from auto-invoking (and removes it from context until called); the second blocks *you* from manually invoking (but keeps the description visible to Claude).
- **Hooks**: exit 2 blocks, exit 0 + JSON gives fine-grained control, exit 1 (and anything else nonzero) is non-blocking — a common, testable trap.
- **CI/CD**: `--bare` for reproducibility, `dontAsk` for locked-down automated permission handling, `--output-format json` for structured, cost-aware scripted consumption.

This domain carries 20% of the exam. The CLAUDE.md-vs-enforcement distinction (§2.5) and the commands-are-now-skills architectural fact (§3.1) are the two points most likely to separate a stale mental model from a current, accurate one.
