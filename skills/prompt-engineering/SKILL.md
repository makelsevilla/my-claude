---
name: prompt-engineering
description: Engineer high-quality prompts tuned for Claude, either from scratch (given a task description) or by rewriting an existing prompt that isn't working well. Use whenever the user wants to create, improve, fix, refine, rewrite, debug, or "engineer" a prompt — including phrasings like "help me write a prompt to...", "this prompt isn't working", "make this prompt better", "how should I phrase this for Claude", "turn this into a prompt", or any request to produce instructions/system messages intended for an LLM. Trigger even when the user doesn't say "prompt engineering" explicitly — if the deliverable is text meant to be fed to an LLM, this skill applies.
---

# Prompt Engineering

A skill for producing well-structured, Claude-tuned prompts on demand. Two modes:

- **Build from scratch**: User describes a task → produce a ready-to-use prompt.
- **Improve existing**: User pastes a prompt that isn't working → diagnose and rewrite.

The final deliverable is **only the engineered prompt**, in a single fenced code block, ready to copy-paste. Don't append explanations, technique breakdowns, or commentary unless the user explicitly asks.

---

## Workflow

### 1. Classify the request

Read the user's message and decide:

- **Build mode**: They described a task they want a prompt for.
- **Improve mode**: They pasted an existing prompt and want it better.
- **Ambiguous**: It's unclear what task the prompt should do, who the end user is, or what input/output looks like.

### 2. Gather the minimum info you need (only if missing)

Before writing the prompt, you need to know:

- **The task** — what should the resulting prompt make Claude do?
- **The input shape** — what will be fed in at runtime? (a document, a user message, structured data, nothing?)
- **The output shape** — plain text, JSON, XML, bullet list, classification label, code?
- **Any hard constraints** — length limits, tone, what to do on edge cases, what NOT to do.

If any of these are genuinely unclear and you can't reasonably assume, ask **one focused round of clarifying questions** (no more than 3). If you can make reasonable assumptions, just make them and note them as placeholders in the prompt — don't bounce the user around with questions.

For improve mode, also diagnose: what's wrong with the current prompt? Common failure modes:
- Vague task description ("summarize this" with no audience/length/focus)
- No examples for a task that needs them
- No output format specified (so the model picks one inconsistently)
- Instructions and input data mixed together with no structural separation
- Conflicting instructions
- Missing edge-case handling
- Wrong technique for the task (e.g., few-shot when chain-of-thought would help more)

### 3. Choose the right techniques

Pick from the toolbox below based on task type. Don't apply every technique to every prompt — over-engineering is a real failure mode. Match technique to need.

### 4. Produce the prompt

Output the prompt in a single fenced code block. Use `{{VARIABLE_NAME}}` placeholders (double-curly, screaming snake case) for anything that will be filled in at runtime. Nothing before the code block except a one-line header, nothing after.

---

## Technique toolbox (Claude-tuned)

### Structure

- **XML tags** are the primary structural tool for Claude. Use them to separate instructions, examples, input data, and output specs. Common tags: `<instructions>`, `<context>`, `<example>`, `<document>`, `<input>`, `<format>`, `<rules>`, `<thinking>`. Tag names should describe the content.
- **Role assignment** in a system-prompt-style opening line ("You are an experienced X who...") gives Claude useful context. Use it when the task benefits from a specific perspective or expertise. Skip it for generic tasks where it adds noise.
- **Section order matters**: long input documents go near the top, instructions near the bottom — Claude attends more to content near the end of the prompt for instruction-following.

### Clarity

- **Be specific and detailed**. "Summarize this article in 3 bullets focused on financial implications for retail investors" beats "summarize this".
- **Tell Claude what to do, not just what to avoid.** Positive instructions outperform negative ones.
- **Number multi-step instructions** so Claude can track them.

### Demonstrations

- **Few-shot examples** are the single highest-leverage technique for non-trivial tasks. Include 2–5 examples in `<example>` tags showing the exact input → output shape you want. Examples beat instructions for teaching format and edge cases.
- Cover edge cases in examples (e.g., one example of "no action items found" if that's a valid output).

### Reasoning

- **Chain of thought**: For tasks involving analysis, classification with nuance, math, or multi-step reasoning, tell Claude to think first. Either with `<thinking>` tags ("Think through this in `<thinking>` tags, then provide your final answer") or step-by-step instructions ("First, identify X. Then, evaluate Y. Finally, output Z.").
- Don't add chain-of-thought to simple lookups or extractions — it adds latency without benefit.

### Format control

- **Specify output format explicitly.** If you want JSON, show the schema. If you want a specific structure, show it in an example.
- **Prefilling**: When the prompt will be used via API, mention that the user can prefill the assistant turn with the opening of the desired output (e.g., `{` for JSON, `<analysis>` for tagged output) to lock format.
- **One output, one format.** If chain-of-thought is used, put the reasoning in `<thinking>` and the final output in a separate tag so it's easy to parse.

### Advanced patterns

- **Prompt chaining**: For complex workflows, split into multiple prompts (e.g., extract → classify → summarize). Each prompt does one thing well. Mention this as an alternative if the user's task is genuinely multi-step and a single prompt would be unwieldy.
- **Tool use / agentic prompts**: If the prompt is for an agent that will call tools, include a clear description of when to use each tool, how to handle tool failures, and when to stop. Encourage the agent to plan before acting.
- **Structured output via schema**: For strict JSON output, include the schema in `<format>` tags and an example that exactly matches. Mention that the API's tool-use mode or structured output parameter is more reliable than pure prompting for production use.
- **Extended thinking**: If the task is genuinely hard (complex math, multi-step reasoning, code with tricky logic), mention that Claude's extended thinking mode may help — but the prompt itself should still be well-structured.

---

## Anatomy of a strong Claude prompt

Most prompts benefit from some subset of these sections, in roughly this order:

1. **Role** (one line, optional) — "You are a…"
2. **Task** (one short paragraph) — what Claude is doing and why
3. **Context** (optional) — background the model needs
4. **Instructions** (`<instructions>` tag, numbered if multi-step) — the how
5. **Rules / constraints** (`<rules>` tag) — must-dos and must-not-dos
6. **Examples** (`<example>` tags) — 2–5 input/output demonstrations
7. **Input data** (named tag like `<document>`, `<ticket>`, `<email>`) — placeholder for runtime input
8. **Output format** (`<format>` tag or inline instruction) — what the final output should look like
9. **Final nudge** — one line at the end pointing Claude at the immediate action ("Now produce the X for the input above.")

Not every prompt needs every section. A simple classification prompt might be role + task + examples + input + format. An agentic prompt might add tool descriptions and a planning step.

---

## Improve mode: diagnostic checklist

When rewriting an existing prompt, run through these silently before writing:

- Is the task one sentence and unambiguous? If not → rewrite the task statement.
- Is there structural separation between instructions and input data? If not → add XML tags.
- Is the output format specified? If not → specify it, ideally with an example.
- Are there examples for non-trivial tasks? If not → add 2–3.
- Are instructions positive ("do X") or negative ("don't do Y, don't do Z…")? Convert to positive where possible.
- Are there conflicting instructions? Resolve.
- Are edge cases handled? ("What if the input is empty / ambiguous / contains X?")
- Is chain-of-thought needed and missing? Or present and unnecessary?
- Is the role appropriate or just decorative?

Then rewrite. Keep what works; don't change things for the sake of changing them.

---

## Output requirements (strict)

- Final output is **only the engineered prompt**, in **one fenced code block**.
- Use a one-line header above the code block: `**Engineered prompt:**` (or `**Rewritten prompt:**` for improve mode).
- Use `{{PLACEHOLDER}}` syntax (double-curly, screaming snake case) for any runtime variables. Don't fill them in with the user's example content.
- Do not append explanations, technique lists, or "I used X technique because Y" commentary unless the user explicitly asks for it.
- If you had to make significant assumptions, you may add **one short line** under the code block flagging them (e.g., "Assumed output is plain text; let me know if you need JSON.") — but keep it to one line.

---

## When to push back

If the user's request would lead to a clearly weaker prompt (e.g., "make the prompt shorter" when the prompt's length is doing real work, or "remove the examples" when the examples are what makes it function), produce the requested version but add a one-line note pointing out the tradeoff. Don't lecture.

---

## Reference

Anthropic's official prompt engineering documentation is the source of truth for techniques and any Claude-specific behavior: https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview
