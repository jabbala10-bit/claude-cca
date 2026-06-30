# CCA-F Domain 3: Prompt Engineering & Structured Output (20% of exam)

**Source note:** Built from Anthropic's official "Prompting best practices" reference (the current, consolidated single source for Claude prompting technique — it explicitly replaced what used to be several separate pages on clarity, examples, CoT, and XML tags), the Agent SDK's structured-outputs documentation, and Claude Code's model-configuration documentation for effort/thinking controls. Verified live, June 2026.

---

## 1. Core Prompt Engineering — Clarity, Specificity, and Instruction Design

### 1.1 The Foundational Mental Model
Anthropic's own framing, worth having close to verbatim: think of Claude as a brilliant but very new employee with no context on your norms, styles, or preferred ways of working. The more precisely you explain what you want, the better the result — this isn't a platitude, it's the organizing principle behind every technique in this domain.

### 1.2 The Golden Rule of Clear Prompting
**Show your prompt to a colleague with minimal context on the task and ask them to follow the instructions. If they're confused, Claude will likely be too.** This is Anthropic's own documented heuristic for testing prompt clarity before ever sending it to the model — and it's a genuinely good exam-answer framing for "how would you debug a prompt that's producing inconsistent results."

### 1.3 Instruction Decomposition and Constraint Design
Break compound instructions into discrete, individually-clear steps rather than one dense paragraph trying to convey several constraints at once. A prompt with five entangled requirements is harder for the model to satisfy *and* harder for a human to verify than the same five requirements stated as a numbered list — clarity and verifiability move together.

### 1.4 System Prompts — Purpose, Structure, and Role Assignment
The system prompt sets persistent context, persona, and ground rules for the whole interaction — distinct from a single turn's instructions. Role assignment ("You are a senior security engineer reviewing this code") is a legitimate, documented technique for steering tone, depth, and the specific lens Claude applies — it's not just cosmetic framing, it measurably shifts what the model attends to and how it phrases output.

---

## 2. XML Tags — Why They Work and When to Use Them

### 2.1 The Precise, Documented Rationale
**XML tags help Claude parse complex prompts unambiguously, especially when a prompt mixes instructions, context, examples, and variable inputs.** This is the exact condition that justifies reaching for XML structuring — not "always use XML," but specifically when a prompt has multiple distinct content types that could otherwise blur together. Claude was specifically trained to recognize this kind of structure, which is why it's measurably more reliable than equivalent plain-paragraph prompts for mixed-content cases.

### 2.2 Standard Tag Conventions (Not Reserved Keywords)
There's no fixed, required tag vocabulary — `<instructions>`, `<context>`, `<example>`/`<examples>`, `<document>`, `<thinking>`, `<answer>`, `<data>` are conventions, not syntax Claude validates. The exam-relevant judgment: **use semantically meaningful tag names that match the content's actual role**, and be consistent within a single prompt — inconsistent or generic tag names (`<a>`, `<b>`) undercut the entire benefit.

### 2.3 Structuring Multi-Document Prompts
For prompts referencing multiple documents, the documented pattern wraps each one in `<document>` tags with `<document_content>` and `<source>` (plus other metadata) as subtags — this is the precise, current convention, not an arbitrary choice, and it's worth using exactly this shape if a scenario question describes a multi-document RAG-adjacent prompt (see Domain 5 for the retrieval side of this).

### 2.4 Grounding in Quotes for Long-Document Tasks
For tasks involving long documents, the documented technique is to **ask Claude to quote the relevant parts of the document first, before carrying out the actual task** — this helps Claude cut through noise in the rest of the document and anchors its answer in something verifiable, rather than synthesizing an answer that sounds plausible but isn't traceable to the source.

---

## 3. Few-Shot (Multishot) Prompting

### 3.1 The Three Documented Quality Criteria
Anthropic's own framing for what makes a good few-shot example set, precisely:
- **Relevant**: mirrors your actual use case closely.
- **Diverse**: covers edge cases and varies enough that Claude doesn't pick up *unintended* patterns (a real, named risk — narrow examples teach narrow, wrong generalizations).
- **Structured**: wrapped in `<example>` tags (multiple examples inside an outer `<examples>` tag) so Claude can distinguish them from instructions.

### 3.2 The Documented Quantity Guidance
**Include 3–5 examples for best results.** This is a specific, current, citable number — not "more is always better" (more examples cost more tokens and can start reinforcing narrow patterns past a point), and not "one example is enough" (insufficient to establish a generalizable pattern reliably).

### 3.3 A Documented Meta-Technique: Using Claude to Improve Your Examples
You can ask Claude itself to evaluate your examples for relevance and diversity, or to generate additional ones based on an initial set — a real, sanctioned technique worth knowing exists, not just a clever trick.

### 3.4 Why Examples Often Beat Verbal Rules
For tasks like entity extraction, classification, or format transformation, a few well-chosen examples frequently outperform an equivalently-effortful paragraph of verbal description — examples show the pattern directly rather than requiring the model to infer it from a description, and they're easier for a human reviewer to verify against actual desired output too.

---

## 4. Chain-of-Thought (CoT) and Extended Thinking

### 4.1 The Three-Tier Documented Progression
Anthropic's own framing, ordered from least to most complex (and correspondingly, least to most token-expensive):
1. **Basic**: "Think step-by-step" — simple, but gives no guidance on *how* to think, which matters when a task has domain-specific reasoning structure.
2. **Guided**: explicitly outline the steps Claude should follow in its reasoning — better guidance, but reasoning and final answer aren't cleanly separated, making programmatic extraction harder.
3. **Structured**: use `<thinking>` and `<answer>` tags to cleanly separate reasoning from final output — the **documented best default** specifically because it lets you programmatically extract just the answer while retaining the full reasoning trace for debugging.

### 4.2 Multishot Examples Combine With Thinking
**Use `<thinking>` tags inside your few-shot examples to show Claude the reasoning pattern you want — it generalizes that style to its own extended thinking.** This is a real, documented composability between §3 and §4: don't treat few-shot and CoT as separate, unrelated techniques — a well-designed example set can *teach the reasoning style*, not just the output format.

### 4.3 Manual CoT as a Fallback When Extended Thinking Is Off
When extended thinking (the model-level reasoning feature — see §4.4) is disabled, you can still get step-by-step reasoning by explicitly asking for it in the prompt, using the same `<thinking>`/`<answer>` structuring. **A specific, current, model-sensitivity fact worth knowing precisely:** when extended thinking is off, certain Claude models are particularly sensitive to the literal word "think" and its variants — using alternatives like "consider," "evaluate," or "reason through" avoids unintended interaction with how the model parses the instruction.

### 4.4 Extended Thinking and Effort Levels — The Claude Code / Platform Mechanism
Distinct from prompted CoT (which is a prompting *technique*), **extended thinking is a model-level capability** the platform/Claude Code exposes as a configurable feature. In Claude Code specifically: **effort levels** (`low`, `medium`, `high`, `xhigh`, `max`, plus the Claude-Code-specific `ultracode` setting) control *adaptive reasoning* — letting the model decide whether and how much to think on each step, rather than using a fixed thinking budget. **The exam-relevant precision:** the effort scale is calibrated *per model* — the same level name does not represent the same underlying reasoning depth across different models, so "set effort to high" is not a portable, model-independent instruction in the way it might intuitively seem.

### 4.5 The Self-Check Technique
A simple, documented, reliably effective addition: append an instruction like **"Before you finish, verify your answer against [test criteria]."** This catches errors reliably, especially for coding and math tasks — worth knowing as a low-effort, high-value addition distinct from full multi-instance review (the more expensive pattern covered in the ML/AI architecture material).

---

## 5. Prompt Chaining — Decomposing Complex Tasks

### 5.1 When to Reach for It
When a single prompt is trying to do too much at once and Claude "drops the ball" on parts of a multi-step task, the documented fix is **prompt chaining**: break the task into distinct, sequential subtasks, each getting its own focused prompt, with the output of one becoming the input to the next.

### 5.2 The Three Documented Benefits, Precisely
- **Accuracy**: each subtask gets Claude's full attention, reducing errors from juggling too much at once.
- **Clarity**: simpler subtasks mean simpler, clearer instructions per step.
- **Traceability**: you can pinpoint exactly which link in the chain is producing a problem, rather than debugging one large, opaque prompt.

### 5.3 Design Principles for Chains
- **Single-task goal per link**: each subtask should have one clear objective, not several.
- **XML for handoffs**: use XML tags to structure what passes from one prompt to the next, for the same parsing-clarity reasons as §2.
- **Isolate and debug per-link**: if Claude misses a step or underperforms, isolate just that step into its own prompt to tune it without redoing the entire chain.

### 5.4 The Architectural Connection to Domain 1
Prompt chaining at the single-prompt level is the direct conceptual ancestor of the **sequential pipeline** topology covered in Domain 1 §3.2/§4.2 — the same "output of stage N feeds stage N+1" idea, just operating at the level of individual prompts rather than full agent/subagent invocations. Recognizing this as one idea showing up at two different levels of system granularity is good, exam-rewarded synthesis.

---

## 6. Structured Output — From Prompted JSON to Schema-Validated Output

### 6.1 Why This Matters: The Concrete Problem It Solves
Free-form text output requires your application to parse out structure (titles, numbers, lists) from inconsistent natural-language formatting. **Structured outputs let you define the exact shape of data you want, and get back validated JSON matching that shape** — directly usable by application logic, a database, or UI components, with no brittle parsing layer in between.

### 6.2 The Mechanism, Precisely (Agent SDK)
Define a **JSON Schema** describing the desired output shape, and pass it via the `outputFormat` (TypeScript) / `output_format` (Python) option on `query()`, with `type: "json_schema"` and a `schema` field. **Critically: the agent can still use any tools it needs during the task — structured output applies to the final result, not a constraint on the whole interaction.** This is the exam-relevant distinction from a single-turn structured-output call: a TODO-extraction agent can use Grep to search and Bash to run `git blame`, autonomously, and still return one final, schema-validated JSON object combining everything it found.

### 6.3 Type-Safe Schemas via Zod / Pydantic
Rather than hand-writing JSON Schema, **Zod** (TypeScript, via `z.toJSONSchema()`) or **Pydantic** (Python, via `.model_json_schema()`) can generate the schema from a typed class/object definition — giving you full type inference, runtime validation (`safeParse()` / `model_validate()`), and reusable, composable schema definitions. This is the documented, recommended path for production use over hand-rolled JSON Schema.

### 6.4 Validation and Retry Behavior — The Precise Failure Mode
**The SDK validates output against your schema and re-prompts automatically on a mismatch.** If validation doesn't succeed within the retry limit, the result is an explicit error, not silently-wrong data. The exact, documented `subtype` values on the result message:
- `success` — output generated and validated.
- `error_max_structured_output_retries` — the agent couldn't produce valid output after multiple attempts.

**This is a frequently-testable mechanic**: a well-designed integration checks `subtype` explicitly and has a defined fallback (retry with a simpler prompt, fall back to unstructured output, surface an error to the user) — silently assuming success and reading `structured_output` without checking `subtype` first is exactly the kind of brittle integration mistake the exam is likely to present as a flawed scenario to identify.

### 6.5 Documented Tips for Avoiding Validation Failures
- **Keep schemas focused** — deeply nested schemas with many required fields are harder for the agent to reliably satisfy; start simple.
- **Match schema to task reality** — if the task might not have all the information a field requires, make that field optional rather than required. (The TODO-extraction example makes `author`/`date` optional specifically because git blame information isn't always available.)
- **Use clear prompts** — an ambiguous prompt makes it harder for the agent to know what output to aim for, compounding with schema complexity rather than being independent of it.

### 6.6 Structured Output via Tool Definitions — The Alternative Path
Beyond the dedicated `outputFormat` mechanism, structured data extraction can also be achieved by defining a tool whose input schema *is* the desired output shape, and having Claude "call" that tool with the extracted data as its arguments — a pattern that predates the dedicated structured-output feature and is still relevant for single-turn API use without the Agent SDK's loop. Know both exist; the dedicated `outputFormat` mechanism is the more current, more directly validated path when using the Agent SDK specifically.

---

## 7. Validation Loops, Hallucination Prevention, and Evaluation

### 7.1 Validation Gates and Retry Strategy
Beyond the SDK's automatic schema-retry (§6.4), a robust pipeline often layers an *additional*, application-level validation gate — checking not just "is this valid JSON matching the schema" but "does this data make domain sense" (e.g., a returned date is plausible, a returned ID actually exists). Schema validation catches structural errors; it does not catch semantically wrong-but-well-formed output — that's a distinct validation layer.

### 7.2 Multi-Pass Review for Large/Complex Extraction
For large extraction or review tasks, splitting into **per-file local analysis passes plus a separate cross-file integration pass** avoids both attention dilution (trying to hold too much in view at once) and contradictory findings across a task that should have been decomposed. This directly echoes Domain 1's general decomposition principles, applied specifically to extraction/review-style prompt engineering tasks.

### 7.3 Building a Prompt Evaluation Workflow
A documented, sound evaluation workflow: generate a **test dataset** of representative inputs, run the prompt against each, and use **model-based grading** (a separate Claude call evaluating the first call's output against criteria) to score results at scale, rather than only spot-checking by hand. This is the prompt-engineering-domain instance of the same "independent review instance is more effective than self-review" principle from the ML/AI document's multi-instance review material — a separate grading call, without the generating call's reasoning context, catches issues a single self-reviewing pass would miss.

### 7.4 Test Dataset Generation
Claude itself can generate plausible test inputs for an evaluation set (directly paralleling §3.3's "ask Claude to generate more examples" technique) — useful for bootstrapping an eval set before you've accumulated enough real production examples to test against.

---

## 8. Quick-Reference Summary for This Domain

- **XML tags**: use when a prompt mixes multiple content types (instructions + context + examples + data) — not a universal default, a fix for a specific structural problem.
- **Few-shot**: 3–5 examples, relevant + diverse + structured in `<example>` tags — the specific documented number, not a vague "a few."
- **CoT progression**: basic → guided → structured (`<thinking>`/`<answer>`), with structured being the documented default for anything you need to programmatically parse.
- **Structured output**: schema-validated JSON via `outputFormat`/`output_format`, auto-retried on mismatch, explicit `error_max_structured_output_retries` failure mode — always check `subtype`, never assume success.
- **Prompt chaining**: reach for it when a single prompt is doing too much; gains are accuracy, clarity, and traceability, in that specific documented order.
- **Evaluation**: model-based grading on a test dataset beats hand-spot-checking at any meaningful scale, and independent grading instances catch more than self-review.

This domain carries 20% of the exam. §6.4's exact `subtype` failure-mode values and §3.2's "3–5 examples" figure are the kind of precise, citable facts most likely to appear as direct recall questions rather than scenario judgment — worth memorizing exactly, not just approximately.
