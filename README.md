# CCA-F Master Index & Exam Strategy

**This is the front door to the five domain docs, not a substitute for them.** Read this first, use it to navigate, and return to it the week of your exam as the final review pass.

---

## 1. The Domain Weighting — Where to Spend Your Time

| Domain | Weight | Doc |
|---|---|---|
| D1: Agentic Architecture & Orchestration | **27%** | `domain-1-agentic-architecture.md` |
| D2: Claude Code Configuration & Workflows | **20%** | `domain-2-claude-code-config.md` |
| D3: Prompt Engineering & Structured Output | **20%** | `domain-3-prompt-engineering.md` |
| D4: Tool Design & MCP Integration | **18%** | `domain-4-tool-design-mcp.md` |
| D5: Context Management & Reliability | **15%** | `domain-5-context-reliability.md` |

**The arithmetic that should drive your study plan**: D1 alone is worth more than D4 and D5 combined. If your time is genuinely constrained, over-index on D1 and D2 (47% combined) before polishing D5's precise numeric recall facts — getting D1's *judgment-based* scenario questions right is harder to cram than D5's *precise-fact* recall, so D1 deserves earlier, deeper, more repeated study.

---

## 2. The Single Idea That Recurs in All Five Domains

**Advisory vs. enforced.** This exact distinction shows up, restated for a different layer, in every domain:
- **D1 §6.4**: prompts are advisory; hooks/permissions are enforced — for high-stakes agentic actions.
- **D2 §2.5**: CLAUDE.md is advisory; settings/hooks are enforced — for configuration.
- **D4 §3.5**: tool annotations are advisory metadata; the `tools` availability list and permission system are enforced — for tool access.
- **D5 (implicitly, via §6.5)**: the context window itself is temporary/advisory in the sense that nothing in it is guaranteed to survive compaction; CLAUDE.md/external memory is the enforced, durable layer.

**If you internalize one cross-cutting lens for the whole exam, this is it.** A huge fraction of scenario-based questions across all five domains are secretly testing whether you reach for the advisory (prompt/instruction) layer or the enforced (hook/permission/structural) layer when a scenario describes something that genuinely needs to happen reliably, every time, regardless of the model's judgment in the moment.

---

## 3. Cross-Domain Connections Worth Having Ready

These are the "aha, these two domains are actually the same idea" connections that read as genuine synthesis on a scenario question, not memorized trivia:

- **D1 subagent context isolation ↔ D2 skill `context: fork`** — a skill forked into a subagent and a subagent with preloaded skills are mirror images of the same underlying mechanism, with the system prompt source and task source swapped (D2 §3.5's table is the exact reference).
- **D1 sequential pipeline topology ↔ D3 prompt chaining** — the same "output of N feeds N+1" idea, at two different levels of granularity (full agent invocations vs. individual prompts).
- **D1 error classification (tool/reasoning/environment) ↔ D4 error handling contract** — D4's throw-vs-`isError` mechanic is the concrete implementation of how a *tool* error specifically gets surfaced into the loop for D1's classification scheme to even operate on.
- **D3 structured output retry (`error_max_structured_output_retries`) ↔ D1 retry/fallback design** — the SDK's own structured-output mechanism is a specific, concrete instance of the general retry-vs-abort judgment D1 §6.2 covers.
- **D4 tool search / deferred loading ↔ D5 cache invalidation** — D5 §4.7's precise point that deferred MCP tools don't invalidate the cache (because they only append) while prefix-loaded tools do, is one of the highest-value, least-obvious cross-domain facts in the entire set.
- **D3 "lost in the middle" mitigation (quote-then-answer) ↔ D5 "lost in the middle" mitigation (chunk ordering)** — the same underlying phenomenon, addressed at the single-prompt level (D3) and the retrieval-pipeline level (D5).

---

## 4. The "Current State, Not Stale Mental Model" Checklist

A meaningful fraction of this exam's value, and risk, is that the underlying product moves fast. These are the specific points where an outdated mental model gives a *confidently wrong* answer rather than just an incomplete one — review this list specifically if any of your study material predates mid-2026:

- **Commands have been merged into skills** (D2 §3.1) — don't treat them as two parallel systems.
- **Tool search / deferred tool loading is on by default** (D4 §4.5) — don't assume every connected MCP server's full tool list always loads upfront into context.
- **SSE is deprecated as an MCP transport** (D4 §4.3) — HTTP is current/recommended for remote servers.
- **The "lost in the middle" effect has a refined, utilization-dependent shape** (D5 §2.2) — it's not a single static U-curve; the pattern shifts above ~50% context utilization.
- **Some current models include the full 1M context window at standard (not premium) pricing** (D5 §1.2) — verify this is still true for whatever specific model the exam references, since pricing structures are exactly the kind of thing that changes across model generations.

---

## 5. Exam-Taking Strategy (Not Just Content)

### 5.1 Scenario Questions Want a Diagnosis, Not a Definition
Across every domain, the strongest-pattern question type presents a scenario (a system behaving a certain way, a design under consideration) and asks you to identify the right architecture, the right failure-mode classification, or the right fix. The recurring shape of a strong answer: **name the specific mechanism at play, not just the category.** "This needs a hook" is weaker than "this needs a `PreToolUse` hook returning `permissionDecision: deny`, because a CLAUDE.md instruction alone wouldn't guarantee enforcement." Specificity is what separates a partial-credit answer from a full-credit one in a well-designed scenario question.

### 5.2 Watch for Questions Testing the Anti-Pattern Directly
D1 §1.2 named this explicitly for loop-termination, but it's a pattern across the whole exam: a question describes a plausible-sounding but actually-flawed design (text-parsing for loop termination, a prompt-only guardrail for a destructive action, an `unordered_map`-style "should be fine" assumption) and the correct answer is identifying *why* it's flawed, not just picking the option that sounds most sophisticated.

### 5.3 Precise Numbers Are Free Points — If You Actually Memorize Them Exactly
Unlike the judgment-heavy D1/D2 material, D4 and D5 in particular have a cluster of exact, citable facts (the 1.25x/0.1x cache multipliers, the 10,000-request/24-hour/29-day batch parameters, the 3-5 few-shot example guidance, the four tool annotations and their defaults) that are either right or wrong — there's no partial credit for "approximately 50% off" if the real answer requires distinguishing batch's flat 50% from caching's 90%. Spend your last study session specifically drilling these numbers cold, since they're the highest-ROI-per-minute-of-study content in the whole exam.

### 5.4 When Two Options Both Sound Plausible, Check the Layer
Given the §2 "advisory vs. enforced" lens, a huge fraction of "which is correct" ambiguity collapses once you ask: *is this scenario asking what shapes the model's behavior, or what guarantees an outcome regardless of the model's behavior?* If the scenario says "must always," "regardless of what the model decides," "even if the model misunderstands," or describes a compliance/safety/financial stake — the enforced-layer answer (hook, permission, schema validation, programmatic check) is almost always correct over the advisory-layer answer (prompt instruction, CLAUDE.md, tool description), no matter how well-worded the advisory option sounds in the question.

---

## 6. Final Pre-Exam Drill (One Pass Through All Five Domains)

Do this from memory, in order, out loud, before you sit the exam:

1. **(D1)** State the two correct loop-termination signals and the three anti-patterns to avoid. State Explore/Plan's three shared properties (model/tools, what they skip, resumability).
2. **(D2)** State the four CLAUDE.md scopes in precedence order, and explain why "precedence" doesn't mean "override" for this specific mechanism. State the two skill-invocation-control frontmatter fields and what each actually does.
3. **(D3)** State the documented few-shot example count and the three quality criteria. State the three-tier CoT progression and which tier is the documented default for programmatic parsing.
4. **(D4)** State the exact throw-vs-`isError` consequence. State current MCP transport status (which is deprecated, which is recommended for what).
5. **(D5)** State the two cache pricing multipliers exactly. State the `/compact` vs `/clear` decision rule in one sentence each.

If all five come back fluently and precisely — not approximately — you're genuinely prepared, not just familiar. Good luck on the exam.
