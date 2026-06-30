# CCA-F Domain 5: Context Management & Reliability (15% of exam)

**Source note:** Built from Anthropic's current prompt-caching and batch-processing documentation (verified live, June 2026), the original Liu et al. (2023) "Lost in the Middle" research and its 2025-2026 refinements, and Claude Code's own context-window documentation. This is the smallest-weighted domain, but several of its facts (exact pricing multipliers, exact TTLs, exact thresholds) are precise enough to be efficient, high-confidence recall questions.

---

## 1. Context Window Architecture — Tokens, Limits, Tradeoffs

### 1.1 What Actually Shares the Budget
The context window is not "your conversation" — it's the system prompt, tool definitions, CLAUDE.md/memory, skill descriptions, and the full conversation history (including every tool result ever returned), **all sharing one token budget simultaneously.** A session that feels like "just a chat" can be dominated by tool-output accumulation rather than the visible conversation text — this is exactly why Domain 2's skill/CLAUDE.md size discipline and Domain 4's deferred-tool-loading (tool search) both exist: they're different levers on the same shared budget.

### 1.2 Current Window Sizes — Know the Current State, Not an Old Number
Claude's current generation of models support a standard **200,000-token** context window, with select models (Opus and Sonnet at the 4.5+ generation) additionally supporting a **1-million-token** window option. **The pricing nuance worth knowing precisely:** for models that include the full 1M window at standard pricing (current Opus and Sonnet generations), a 900K-token request is billed at the *same per-token rate* as a 9K-token request — there is no automatic premium just for being a large request on those models. (Earlier-generation long-context offerings did carry a premium-rate tier past 200K; the current state for the latest model generations does not — this is a fast-moving specific number worth re-verifying close to your exam date.)

### 1.3 The Real Lesson From the 1M Window: More Context Is Not Strictly Better
This is the single most important conceptual point in this domain, and it directly motivates everything in §2: **a bigger context window does not mean you should fill it.** Empirically, most real sessions never approach even the 200K limit before becoming unwieldy, and retrieval accuracy measurably degrades as a window fills, independent of whether the window's *maximum* size is large. The 1M window's genuine, well-targeted use case is loading one large, cohesive artifact (a full codebase, a large document set) for cross-referencing in a single request — **not** as a way to avoid ever having to manage context in a long, turn-by-turn conversation, where stale early context can actively hurt rather than just sit inertly unused.

---

## 2. The "Lost in the Middle" Effect — Evidence and Mitigation

### 2.1 The Original Finding, Precisely
Liu et al.'s 2023 "Lost in the Middle" paper (the foundational, citable research underlying this entire topic) found that retrieval accuracy is highest when relevant information sits at the very start or end of the input context, and **degrades, sometimes sharply, when the relevant information is in the middle** — a U-shaped accuracy curve as a function of position, not the flat, position-independent accuracy you'd intuitively expect from a model that "reads everything."

### 2.2 The Current, More Precise Refined Picture
A meaningfully more sophisticated finding than the basic U-shape is worth knowing for depth: more recent analysis (2025-era replications) found the *exact shape* of the degradation depends on how full the context window already is — **below roughly 50% utilization, the classic U-shape holds (middle tokens are lost worst); above roughly 50% utilization, the pattern shifts to favor recency, with degradation increasing by distance from the end** (i.e., the *earliest* tokens become the most at-risk once the window is more than half full, not strictly the middle ones). Knowing this refinement — that the effect isn't a single static curve but one that shifts shape as utilization grows — is a stronger, more current answer than reciting the original 2023 U-shape as if it were the complete picture.

### 2.3 Why This Happens — The Architectural, Not Just Empirical, Explanation
The effect is a structural property of how transformer architectures encode positional information (the rotary position embedding scheme used in most current models introduces a decay effect for tokens far from both edges of the sequence), **not a bug specific to any one model or a simple training-data artifact** — meaning it cannot be fully "trained away" by any single model release, only reduced in severity. Stating this — that the cause is architectural, not incidental — is what separates a deep answer from a surface-level "models sometimes miss stuff in long documents" answer.

### 2.4 Mitigation Strategies — Concrete and Actionable
- **Position your most important content at the start or end, never buried in the middle**, when you control prompt construction directly.
- **In RAG specifically: order retrieved chunks deliberately, not just by raw relevance rank** — a documented, concrete pattern places the single highest-confidence chunk first, the second-highest last, and lower-ranked/lower-confidence chunks in the middle, deliberately exploiting the same bias rather than fighting it.
- **Retrieve fewer, better chunks rather than more, broader ones** — every additional mediocre chunk is more middle-ground for the genuinely important content to get lost in; a tighter reranking step that returns 3-5 high-confidence chunks beats returning 20 loosely-relevant ones.
- **Ask Claude to quote the relevant part of a long document before answering** (directly cross-referencing Domain 3 §2.4) — this is a second, independent mitigation: it forces the model to explicitly locate and surface the relevant span rather than silently relying on attention alone to find it.

---

## 3. RAG Architecture — Chunking, Embedding, and the Retrieval Pipeline

### 3.1 When RAG Is Still the Right Architecture, Even With Large Context Windows
A precise, current decision framework, given that §1.3 already established "just load everything into a 1M window" isn't a universal answer: reach for RAG when your **total corpus exceeds the context window** (RAG remains necessary, not optional, once data exceeds even 1M tokens), when your **data updates frequently** and stale information loaded once into a long-lived context would go stale, when you need **low-latency, sub-200ms-class responses** (loading a huge document into context on every call is slower than a fast vector lookup), or when you're running **high query volume** and want to minimize the cost of reprocessing a huge context on every single call by retrieving only what's relevant per query.

### 3.2 The Core Pipeline Stages
- **Chunking**: splitting source documents into retrievable units — chunk size is a real, consequential tradeoff (too small loses context within a chunk; too large reintroduces the "lost in the middle"-adjacent problem of diluting a chunk's own signal, and wastes retrieval budget on irrelevant surrounding text).
- **Embedding**: converting chunks (and queries) into vector representations for similarity search.
- **Retrieval**: querying the vector index for the most relevant chunks to a given query.
- **Reranking** (often a distinct, additional stage): a second, often more expensive pass that re-scores the initially-retrieved candidates for genuine relevance before they're handed to the generation step — directly the §2.4 "fewer, better chunks" mitigation, implemented as a pipeline stage.

### 3.3 Semantic Search, BM25, and Hybrid Retrieval
- **Semantic (embedding-based) search**: finds conceptually similar content even without exact keyword overlap — strong for paraphrased or conceptual queries, weaker for queries needing an exact term/code/ID match.
- **BM25 (keyword/lexical search)**: a classical, exact-term-matching algorithm — strong for precise terminology, product codes, or proper nouns that an embedding model might conflate with semantically similar-but-wrong alternatives.
- **Hybrid retrieval**: combining both (commonly via a weighted or reciprocal-rank-fusion combination of each method's results) — the documented, generally-recommended production pattern specifically because semantic and lexical search have complementary, non-overlapping failure modes; relying on either alone leaves a real, predictable gap the other one would have covered.

### 3.4 Multi-Index RAG and Production Hardening
For larger or more heterogeneous corpora, a single flat index is often insufficient — multi-index designs (separate indices per document type, source, or freshness tier, with query-time routing to the relevant index or indices) are the production-scale pattern. **Citation** — surfacing exactly which retrieved chunk/source backed each part of a generated answer — is a documented production-hardening practice for the same reason Domain 3 §2.4 emphasized quoting: it makes the answer's grounding independently verifiable rather than asking the user to trust an unattributed synthesis.

---

## 4. Prompt Caching — Eligibility, Placement, and Cost Impact

### 4.1 The Core Mechanism, Precisely
Prompt caching lets you mark a point in your prompt (a **cache breakpoint**, via `cache_control: {"type": "ephemeral"}`) such that everything *up to and including* that point can be reused by a subsequent request with an identical prefix, skipping reprocessing of that content entirely. **Cache prefixes are constructed in a fixed order: tools, then system, then messages** — a breakpoint placed on the last system block caches both the tools and system sections together, since everything before it in that fixed order is included automatically.

### 4.2 The Exact, Current Pricing Multipliers — Worth Memorizing Precisely
- **Cache write**: billed at a **1.25x** multiplier versus standard input pricing for that content (a real, small premium for the *first* request that establishes the cache entry).
- **Cache read** (a subsequent request hitting the cached prefix): billed at **0.1x** standard input pricing — a 90% discount on that portion of the input.
- **Net effect documented by Anthropic**: this combination can reduce costs by up to 90% and reduce latency by more than 2x for prompts with substantial reusable content.

### 4.3 TTL — Two Options, and the Tradeoff Between Them
**5-minute TTL** (the default) versus **1-hour TTL** (explicitly opt-in, at a higher write-cost multiplier than the 5-minute default, though still far cheaper than no caching at all over repeated reads) — the 1-hour option is the documented recommendation specifically for batch processing workloads (§5) and any workflow where requests are spaced out further than 5 minutes but still want to share a cache. **The TTL resets on every cache read** — a session that keeps reading the cache regularly stays warm indefinitely; only a genuine gap longer than the TTL forces a full recomputation and a fresh write.

### 4.4 Up to Four Breakpoints — Why You'd Want More Than One
You can define **up to 4 cache breakpoints** per request, specifically to cache different sections that change at *different frequencies* (e.g., tool definitions that almost never change, a system prompt that changes occasionally, and a daily-refreshed context block that changes more often than that, each as its own breakpoint) — a single breakpoint forces everything before it to share one all-or-nothing invalidation behavior, which is wasteful when parts of the prefix are genuinely more stable than others.

### 4.5 The Single Most Common, Costly Mistake — Worth Knowing Exactly
**Placing the cache breakpoint on a block that changes every request** (a timestamp, the latest user message, any per-request variable content) silently defeats caching entirely: the prefix hash never matches between requests, so every single request becomes a fresh, full-price cache *write* that's never subsequently *read* — you pay the 1.25x write premium repeatedly and never collect the 0.1x read discount that's supposed to offset it. **The fix is structural, not a parameter to tune**: place the breakpoint on the *last block that stays genuinely identical across the requests you want to share a cache* — typically the end of a static prefix, immediately before the first per-request-varying content begins. This is a "silent failure" in the precise sense that the request still succeeds and returns a correct answer — the only visible symptom is that `cache_creation_input_tokens` is repeatedly nonzero while `cache_read_input_tokens` stays at zero, which requires actually checking usage fields to notice at all.

### 4.6 Automatic vs. Explicit Caching
**Automatic caching**: a single top-level `cache_control` field, with the system automatically placing the breakpoint on the last cacheable block and sliding it forward as a conversation grows — the documented, recommended default for ordinary multi-turn conversations. **Explicit breakpoints**: placed directly on individual content blocks, needed when different sections of the prompt change at different frequencies (§4.4) or when automatic placement would land on the wrong (frequently-changing) block, exactly as described in §4.5's failure case.

### 4.7 What Invalidates a Cache — A Concrete, Testable List
Any byte-level difference in the prefix up to and including the breakpoint invalidates that cache entry — concretely and non-exhaustively: a changed system prompt, a different or reordered tool list, a different `tool_choice` setting, adding or removing an image, or switching models mid-session. **In Claude Code specifically**: whether an MCP server change invalidates the cache depends on whether that server's tools are deferred by tool search (Domain 4 §4.5) or loaded directly into the prefix — deferred tools only *append* new content when discovered and don't disturb what's already cached, while prefix-loaded tools (the case when tool search is unavailable, e.g. on certain models, or for `alwaysLoad`-marked tools) invalidate the cache on any change, since they sit inside the very prefix the breakpoint covers. This is a genuinely deep, current, and directly testable cross-domain connection between Domain 4's tool-loading architecture and Domain 5's caching behavior.

---

## 5. Batch API — Cost Savings, the Processing Window, and Workload Design

### 5.1 The Core Offer, Precisely
The Message Batches API processes large volumes of independent requests **asynchronously, at a flat 50% discount on both input and output tokens**, compared to standard synchronous API pricing — **the discount is unconditional**, with no volume threshold required to unlock it.

### 5.2 The Processing Window — Exact, Current Numbers
- Most batches complete in **under one hour**, though Anthropic's own documentation is explicit that processing time is best-effort, not guaranteed.
- The **hard ceiling is 24 hours** — a batch that hasn't finished by then expires, and any still-pending requests within it are marked expired rather than completed.
- Results remain available for **29 days** after batch creation; after that window, the batch record itself may still be viewable, but results are no longer downloadable.
- A single batch can hold **up to 10,000 requests**; for larger workloads, the documented pattern is submitting multiple sequential batches, not waiting for a single oversized one.

### 5.3 The Right Workload Shape for Batch
Anthropic's own examples are precise and worth having ready: **large-scale evaluations** (running thousands of test cases), **content moderation** (analyzing large volumes of user-generated content with no real-time requirement), and bulk document processing/classification generally. **The defining characteristic that makes a workload batch-appropriate**: it does not require an immediate response, and the requests are genuinely independent of each other (each is processed on its own, not as turns in a shared conversation).

### 5.4 Batch + Prompt Caching — They Stack, With a Caveat
The two discounts **combine** — caching's 90%-off reads layer on top of batch's flat 50% off, providing substantially deeper savings than either alone for workloads with shared, reusable context across many batch requests. **The precise, documented caveat**: because batch requests are processed asynchronously and concurrently rather than strictly in the order submitted, cache hits within a batch are **best-effort, not guaranteed** — real observed hit rates range from roughly 30% to 98% depending on traffic pattern, and the documented mitigation is to use the **1-hour TTL** specifically for batch workloads (since the default 5-minute TTL is too short to reliably survive a batch's concurrent, non-sequential processing) and to include identical `cache_control` placement across every request in the batch that's meant to share a cache entry.

### 5.5 What's Not Supported in Batch Requests
A small number of standard Messages API parameters are explicitly unsupported in batch mode, and **streaming is not supported at all** for batch requests — a structural consequence of the asynchronous, poll-for-results model, not an oversight. A scenario requiring real-time streamed output is, by definition, not a batch-appropriate workload, regardless of cost pressure.

---

## 6. Long-Conversation Coherence, Compaction, and Session Memory

### 6.1 The Core Tension This Section Addresses
Everything in §1-§2 establishes that filling the context window is itself a quality risk, independent of hitting any hard limit. Long-running sessions need an active strategy for managing accumulated context, not just a passive "wait until the limit is hit" approach.

### 6.2 Auto-Compaction — The Default, Reactive Mechanism
Claude Code triggers automatic compaction (summarizing conversation history into a condensed form) once context utilization crosses a high threshold (commonly cited around 80-95%, with the exact figure subject to change across CLI versions — treat any specific percentage as approximate and version-dependent rather than a fixed constant to memorize precisely). **The documented limitation worth naming explicitly**: auto-compaction summarizes without specific guidance about what's actually important to preserve for *your* task — it's a generic, one-size-fits-all compression, which is precisely why proactive, manual intervention (see §6.3) is the more reliable practice for any session where precision about what survives actually matters.

### 6.3 Proactive (Manual) Compaction vs. Reactive Auto-Compaction
The documented best practice is intervening **before** the auto-compact threshold, at a point in the session that corresponds to a natural logical boundary (after completing a unit of work, not mid-implementation) — manual `/compact`, optionally with a **targeted, focused prompt** ("keep only the auth flow decisions, discard the UI discussion") rather than a generic, undirected summarization. This is the single biggest lever available for controlling exactly what a compaction step preserves versus discards — generic auto-compaction cannot target-preserve anything, since it has no way to know what you specifically still need.

### 6.4 `/compact` vs. `/clear` — A Precise Decision Rule
- **`/compact`**: summarizes and retains a condensed version of prior context — use when continuity matters and some prior decisions need to survive into the next phase of work.
- **`/clear`**: a hard reset with **no recovery option** — wipes session history entirely, with no summary at all (though CLAUDE.md still reloads automatically, since it's read from disk fresh each session, not carried as conversation history). Use when prior context has **negative** value — it's actively wrong, stale, or irrelevant to a new, unrelated task — since in that case, paying to retain and re-attend to it (even in compacted form) is worse than starting clean.
- **The documented warning worth repeating precisely**: `/clear` is irreversible. Anything not already persisted to CLAUDE.md or external memory before running it is genuinely, permanently gone from that session.

### 6.5 Session Memory Across Compaction and Restarts — Tying Back to Domain 1 and Domain 2
This section is the direct, cross-domain synthesis point for the whole document: **Domain 1 §5.3-5.4** established that subagent transcripts persist independently of main-conversation compaction, and that durable memory requires explicit external persistence (the `memory` frontmatter field). **Domain 2 §2.6-2.7** established that project-root CLAUDE.md auto-reloads after compaction but nested CLAUDE.md and conversational-only instructions do not. **The unifying principle across all of it**: the context window is fundamentally a *temporary working space*, and anything that needs to survive compaction, a session restart, or a version upgrade (which, per current Claude Code documentation, forces a full cache invalidation and full reprocessing of conversation history on the next request) must be **deliberately externalized** — to CLAUDE.md, to auto/structured memory, or to a RAG-retrievable store — rather than assumed to simply persist because it's "in the conversation."

---

## 7. Quick-Reference Summary for This Domain

- **Bigger context ≠ better results.** The 1M window is for one large cohesive artifact, not a replacement for active context management in long, turn-by-turn sessions.
- **Lost in the middle has a refined, current shape**: below ~50% utilization, the classic U-curve (middle lost worst); above ~50%, recency-favored (earliest tokens become the most at-risk).
- **Cache pricing, exact**: 1.25x write, 0.1x read — and the most common real failure is a breakpoint on a per-request-varying block, which silently produces write-without-read forever.
- **Batch**: flat 50% off, 24-hour hard ceiling, 10,000 requests/batch, no streaming, stacks with caching (use the 1-hour TTL specifically for batch workloads).
- **`/compact` vs `/clear`**: compact when prior context still has positive value; clear when it's actively wrong or irrelevant — and clear is irreversible.

This domain carries 15% of the exam — the smallest share, but with an unusually high density of exact, precisely-citable numbers (1.25x/0.1x, 4 breakpoints, 10,000 requests, 24 hours, 29 days, 50%) that make for efficient, high-confidence recall questions if memorized exactly rather than approximately.
