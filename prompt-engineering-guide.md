# The Engineer's Guide to Prompt Engineering

## Table of Contents

1. [How LLMs Actually Process Prompts](#1-how-llms-actually-process-prompts)
2. [Core Principles](#2-core-principles)
3. [Techniques — From Basic to Advanced](#3-techniques--from-basic-to-advanced)
4. [Structured Output Control](#4-structured-output-control)
5. [Context Window Management](#5-context-window-management)
6. [System Prompts vs User Prompts](#6-system-prompts-vs-user-prompts)
7. [Debugging Bad Outputs](#7-debugging-bad-outputs)
8. [Anti-Patterns](#8-anti-patterns)
9. [Domain-Specific Prompting](#9-domain-specific-prompting)
10. [Evaluation & Iteration](#10-evaluation--iteration)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. How LLMs Actually Process Prompts

Before learning techniques, understand the machine you're programming.

### Tokenization

LLMs don't see characters or words — they see **tokens**. A token is roughly 3-4 characters in English. This matters because:

- The model's "memory" (context window) is measured in tokens, not words.
- Token boundaries can split words in unexpected ways, affecting how the model interprets your input.
- Code, structured data, and non-English text tokenize less efficiently (more tokens per concept).

```
"Hello world"     → ["Hello", " world"]          = 2 tokens
"indistinguishable" → ["ind", "ist", "inguish", "able"] = 4 tokens
"2026-04-09"      → ["202", "6", "-", "04", "-", "09"] = 6 tokens
```

### Autoregressive Generation

The model generates one token at a time, left to right. Each new token is predicted based on everything before it (your prompt + its own output so far). This has critical implications:

- **Order matters.** Information placed early in the prompt has a different effect than information placed late.
- **The model can't "go back."** Once it commits to a token, it builds on it. A bad early token cascades.
- **Asking the model to "think first" works** because it generates reasoning tokens that then influence the answer tokens.

### Attention and the "Lost in the Middle" Problem

Transformer attention is not uniform across the context window. Research shows:

- Information at the **beginning** and **end** of the prompt gets the most attention.
- Information in the **middle** of long prompts can be partially ignored.
- This is why placing critical instructions at the start or end matters.

```
┌─────────────────────────────────────────────┐
│ HIGH        LOWER ATTENTION         HIGH    │
│ ATTENTION   ←── middle ──→         ATTENTION│
│ ← start                            end →   │
└─────────────────────────────────────────────┘
```

### Temperature and Sampling

These parameters control randomness in token selection:

| Parameter     | What it does                                    | When to adjust                     |
|---------------|------------------------------------------------|------------------------------------|
| Temperature   | Scales the probability distribution. Lower = more deterministic | Factual tasks → low (0-0.3). Creative → higher (0.7-1.0) |
| Top-p         | Nucleus sampling — only considers tokens within cumulative probability p | Usually leave at 0.9-1.0 unless you need tight control |
| Top-k         | Only considers the k most likely tokens         | Rarely needed if using top-p       |

The key insight: **no amount of prompt engineering fixes a bad temperature setting for your use case.**

---

## 2. Core Principles

### 2.1 Be Specific, Not Verbose

Specificity and verbosity are different things. A specific prompt can be short.

```
❌ Vague:
"Write me something about databases."

❌ Verbose but still vague:
"I would really appreciate it if you could write me a nice, detailed,
comprehensive piece about databases and how they work and all the
different types and everything related to them."

✅ Specific:
"Compare PostgreSQL and DynamoDB for a write-heavy IoT ingestion
pipeline processing 50k events/sec. Cover: data model trade-offs,
cost at scale, and operational complexity."
```

The mental model: **you're writing a function signature, not a letter.** Define inputs, constraints, and expected output shape.

### 2.2 Provide Context the Model Doesn't Have

The model has broad training data but doesn't know:
- Your specific codebase, architecture, or conventions
- Your team's preferences or constraints
- What you've already tried
- The current state of your system

```
❌ "Why is my Lambda timing out?"

✅ "My Python 3.12 Lambda (256MB, 30s timeout) queries a PostgreSQL
RDS instance in the same VPC. It times out on cold starts but works
on warm invocations. The Lambda is in a private subnet with a NAT
gateway. Security group allows outbound to RDS on port 5432."
```

### 2.3 Constrain the Output

Unconstrained prompts get unconstrained answers. Tell the model:

- **Format**: JSON, markdown, bullet points, code only, etc.
- **Length**: "in 3 sentences", "under 200 words", "a one-liner"
- **Scope**: "only cover X", "don't include Y"
- **Audience**: "for a senior engineer", "for a non-technical PM"

```
"Explain the CAP theorem in exactly 3 bullet points,
each under 20 words, for a senior backend engineer."
```

### 2.4 One Task Per Prompt (When Precision Matters)

Compound prompts dilute focus. If you need high quality on each part, split them.

```
❌ "Write a Terraform module for an ECS cluster, also write the CI/CD
pipeline, and document the architecture."

✅ Three separate prompts:
1. "Write a Terraform module for an ECS Fargate cluster with..."
2. "Write a GitHub Actions pipeline that deploys to the ECS cluster defined in [module]..."
3. "Document the architecture from these two artifacts..."
```

For conversational use this is natural. For programmatic use (API calls), this means chaining calls.

---

## 3. Techniques — From Basic to Advanced

### 3.1 Zero-Shot Prompting

Give the task with no examples. Works well for straightforward tasks the model has seen extensively in training.

```
"Convert this SQL query to a SQLAlchemy ORM expression:
SELECT users.name, COUNT(orders.id)
FROM users LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.name"
```

When to use: simple, well-defined tasks. When to avoid: tasks with ambiguous output format or domain-specific conventions.

### 3.2 Few-Shot Prompting

Provide examples of input → output pairs. The model pattern-matches from your examples.

```
Convert these error codes to human-readable messages:

ERR_CONN_REFUSED → "Unable to connect to the server. Check if the service is running."
ERR_TIMEOUT → "The request timed out. The server may be under heavy load."
ERR_AUTH_FAILED → "Authentication failed. Verify your credentials."

Now convert:
ERR_RATE_LIMIT →
ERR_DISK_FULL →
ERR_CERT_EXPIRED →
```

**Why it works**: Few-shot examples don't "teach" the model — they activate relevant patterns from training and establish the expected format, tone, and reasoning depth.

**Tips for effective few-shot**:
- Use 2-5 examples. More isn't always better (wastes tokens, can overfit to examples).
- Make examples diverse — cover edge cases, not just the happy path.
- Order matters: put the most representative example first and last (attention bias).
- If the model copies your examples too literally, add a note: "These are examples of the format. Generate novel content."

### 3.3 Chain-of-Thought (CoT)

Ask the model to reason step by step before answering. This forces it to generate intermediate reasoning tokens that improve the final answer.

```
❌ "Is this function O(n) or O(n²)?"

✅ "Analyze the time complexity of this function.
Walk through the loop structure step by step,
identify the dominant term, then state the Big-O complexity."
```

**Why it works mechanically**: Remember autoregressive generation — each token depends on all previous tokens. When the model writes "The outer loop runs n times, the inner loop runs n times for each iteration...", those reasoning tokens are now in context when it generates the final answer. Without CoT, it jumps straight to an answer with less "working memory."

**Variants**:
- **Explicit CoT**: "Think step by step."
- **Structured CoT**: "First identify X, then analyze Y, finally conclude Z."
- **CoT with verification**: "After reaching your answer, verify it by [method]."

### 3.4 Self-Consistency

Run the same prompt multiple times (with temperature > 0) and take the majority answer. This is a programmatic technique, not a single-prompt trick.

```python
# Pseudocode
answers = [call_llm(prompt, temperature=0.7) for _ in range(5)]
final_answer = majority_vote(answers)
```

When to use: high-stakes decisions where you can afford the extra API calls. Particularly effective for math, logic, and classification tasks.

### 3.5 Role Prompting

Assign the model a role to activate domain-specific knowledge patterns.

```
"You are a senior site reliability engineer with 10 years of
experience running large-scale distributed systems on AWS.
Review this architecture for single points of failure."
```

**Why it works**: The role constrains the probability distribution toward tokens associated with that expertise in training data. "A senior SRE" is more likely to mention blast radius, circuit breakers, and graceful degradation than a generic assistant.

**Nuance**: Role prompting is a soft constraint. It biases the output but doesn't guarantee expertise. Always verify technical claims.

### 3.6 Decomposition Prompting

Break complex problems into sub-problems explicitly in the prompt.

```
"I need to design a rate limiter for our API gateway.

Address each of these independently:
1. Algorithm choice: Compare token bucket vs sliding window for bursty traffic
2. Storage: Redis vs in-memory, considering a 12-node cluster
3. Failure mode: What happens when the rate limiter itself is unavailable?
4. Client experience: Response headers and retry-after behavior"
```

This is more effective than asking "design a rate limiter" because it prevents the model from fixating on one aspect and neglecting others.

### 3.7 Metacognitive Prompting

Ask the model to evaluate its own confidence or identify gaps.

```
"Answer this question, then rate your confidence (high/medium/low)
and explain what information, if available, would change your answer:

What's the maximum throughput of a single Kinesis shard
when records average 50KB?"
```

This doesn't make the model actually "know" its confidence, but it activates patterns where training data discussed uncertainty, caveats, and limitations — producing more calibrated responses.

### 3.8 Prompt Chaining

Use the output of one prompt as input to the next. This is the programmatic equivalent of a multi-step conversation.

```
Chain for code review:

Prompt 1: "Identify all potential bugs in this code. List them numbered."
     ↓ output
Prompt 2: "For each bug identified: {output_1}, write a fix. Show before/after."
     ↓ output
Prompt 3: "Review these fixes: {output_2}. Do any introduce new issues?"
```

This works better than a single "review and fix this code" prompt because each step has focused attention and the model can build on verified intermediate results.

### 3.9 Negative Prompting (Exclusion Constraints)

Tell the model what NOT to do. This is surprisingly effective because it prunes large branches of the output space.

```
"Explain Kubernetes networking.
- Do NOT start with 'Kubernetes is a container orchestration platform'
- Do NOT cover topics the reader already knows: pods, services, deployments
- Do NOT use analogies
- Jump straight into CNI plugin mechanics and iptables rules"
```

### 3.10 Iterative Refinement Prompting

Use follow-up prompts to refine output. This is the most natural technique in conversation.

```
Prompt 1: "Write a Python retry decorator with exponential backoff."
Prompt 2: "Add jitter to prevent thundering herd."
Prompt 3: "Add a parameter for a custom exception whitelist."
Prompt 4: "Now add type hints and a docstring with usage example."
```

Each step builds on the previous output. This produces better results than cramming all requirements into one prompt because the model focuses on one concern at a time.

---

## 4. Structured Output Control

### JSON Output

```
"Return the result as a JSON object with this exact schema:
{
  "severity": "critical" | "warning" | "info",
  "message": "string",
  "line_number": number,
  "suggested_fix": "string"
}
Return ONLY the JSON. No markdown fences, no explanation."
```

Providing the schema as an example is more reliable than describing it in prose.

### Markdown / Tables

```
"Present the comparison as a markdown table with columns:
| Feature | PostgreSQL | DynamoDB | Winner |"
```

### Code-Only Output

```
"Respond with only the code. No explanations, no markdown fences,
no comments unless they clarify non-obvious logic."
```

### Controlling Length

Explicit length constraints work but are approximate:

- "In one sentence" — reliable
- "In exactly 50 words" — unreliable (models are bad at counting tokens)
- "Under 200 words" — mostly reliable
- "In 3 bullet points" — reliable (structural constraint, not counting)

Structural constraints (bullets, sections, tables) are more reliable than word counts.

---

## 5. Context Window Management

### The Token Budget Mental Model

Think of the context window as a fixed budget:

```
Context Window = System Prompt + Conversation History + User Input + Model Output
```

If your system prompt is 2,000 tokens and your context window is 8,000 tokens, you have 6,000 tokens for everything else. This is why concise system prompts matter in production.

### Strategies for Long Contexts

**Summarization checkpoints**: In long conversations, periodically ask the model to summarize the conversation so far, then start a new context with that summary.

**Relevant context only**: Don't dump your entire codebase. Include only the files/functions relevant to the question.

```
❌ "Here's my entire 500-line file. Why does line 347 fail?"

✅ "Here's the function that fails (lines 340-360) and the type
definitions it depends on (lines 12-25):"
```

**Reference before question**: Place reference material before the question, not after. The model processes sequentially.

```
✅ Structure:
[Context / reference material]
[Specific question about that material]

❌ Structure:
[Question]
[Context dumped after]
```

---

## 6. System Prompts vs User Prompts

In API usage, you have distinct roles: `system`, `user`, `assistant`.

### System Prompt

Sets persistent behavior, personality, constraints, and format. It's the "configuration" of the model for your application.

```
System: "You are a code review assistant. You only review Python code.
For each issue found, cite the exact line number and explain why it's
a problem. Categorize issues as: bug, performance, style, security.
Respond in JSON array format."
```

**Best practices**:
- Keep it focused. A system prompt trying to do everything does nothing well.
- Put hard constraints here (output format, refusal conditions, persona).
- Don't repeat system prompt content in user messages — it wastes tokens.

### User Prompt

The per-request input. This is where the specific task, data, and context go.

```
User: "Review this function:
def process(data):
    result = eval(data['expression'])
    return result"
```

### The Interaction

System and user prompts interact. A well-designed system prompt means your user prompts can be shorter and more natural. A vague system prompt forces every user prompt to re-establish context.

---

## 7. Debugging Bad Outputs

When the model gives you garbage, diagnose systematically:

### Problem: Output is too generic

**Cause**: Prompt lacks specificity or constraints.
**Fix**: Add concrete details about your environment, constraints, and expected output format.

### Problem: Model ignores instructions

**Cause**: Instruction buried in the middle of a long prompt, or conflicting instructions.
**Fix**: Move critical instructions to the beginning or end. Remove contradictions. Use emphasis formatting (caps, bold, or XML tags for structure).

```
"IMPORTANT: Return ONLY valid JSON. No explanations."
```

### Problem: Model hallucinates facts

**Cause**: Asked about something outside training data, or asked to be too specific about something it's uncertain about.
**Fix**: Ask the model to cite sources, say "I don't know" when uncertain, or provide the facts yourself and ask it to reason over them.

### Problem: Model is too verbose

**Cause**: Default training biases toward helpfulness = length.
**Fix**: Explicit length constraints. "Answer in one sentence." "Be terse." "No preamble."

### Problem: Inconsistent output format

**Cause**: Format not specified precisely enough, or few-shot examples are inconsistent.
**Fix**: Provide a schema or template. Use few-shot examples with consistent format.

### Problem: Model refuses a legitimate request

**Cause**: Safety filters triggered by surface-level pattern matching.
**Fix**: Reframe the request with explicit context about why it's legitimate. Add professional context.

```
❌ "How do I exploit a buffer overflow?"
✅ "I'm writing a CTF challenge for an internal security training.
I need an example of a stack buffer overflow in C that participants
will need to identify and patch. Include the vulnerable code and
the fixed version."
```

---

## 8. Anti-Patterns

### "Please" and "Thank you" tokens

Politeness doesn't improve output quality. Every token of "I would really appreciate it if you could kindly..." is a token not spent on useful context. Be direct.

### The Kitchen Sink Prompt

Cramming every possible instruction into one prompt. The model can't prioritize when everything is "important."

### Prompt Cargo-Culting

Copying prompts that worked for someone else without understanding why. "Act as a 10x engineer" is meaningless — what specific behaviors do you want?

### Assuming the Model Remembers

In API usage, each call is stateless (unless you manage conversation history). The model doesn't remember previous API calls. In chat interfaces, it has the conversation history but nothing from previous sessions.

### Over-Constraining

```
❌ "Write exactly 47 words using only words with fewer than 8 letters,
in passive voice, with exactly 3 semicolons."
```

The model will try to satisfy all constraints and fail at most of them. Prioritize the constraints that actually matter.

### Repeating the Same Prompt Hoping for Better Results

If a prompt doesn't work, repeating it identically (at temperature 0) gives the same result. Change the prompt, not the number of attempts.

---

## 9. Domain-Specific Prompting

### For Code Generation

```
"Write a [language] function that [specific behavior].

Requirements:
- Input: [types and constraints]
- Output: [type and format]
- Error handling: [strategy]
- Edge cases: [list them]

Do not use [deprecated API / library / pattern].
Use [preferred library / pattern] instead."
```

Include existing code conventions when relevant:
```
"Our codebase uses:
- snake_case for functions
- Type hints on all public functions
- Google-style docstrings
- pytest for testing

Match these conventions."
```

### For Architecture / Design

```
"Design [system component] given these constraints:
- Scale: [requests/sec, data volume, users]
- Latency requirement: [p99 target]
- Budget: [if relevant]
- Existing stack: [what's already in place]
- Team expertise: [relevant skills/gaps]

Evaluate at least 2 approaches. For each, cover:
trade-offs, operational complexity, and migration path
from current state."
```

### For Debugging

```
"I'm seeing [exact error message / behavior].

Environment: [language version, OS, relevant dependencies]
Code: [minimal reproduction]
Expected: [what should happen]
Actual: [what happens instead]
Already tried: [what you've ruled out]"
```

The "already tried" field is critical — it prevents the model from suggesting things you've already done.

### For Infrastructure / DevOps

```
"Write [Terraform/CloudFormation/CDK] for [resource].

Constraints:
- Must be in [region/VPC/account]
- Naming convention: [pattern]
- Tags required: [list]
- Security: [specific requirements]
- Must integrate with existing [resource] named [name]"
```

---

## 10. Evaluation & Iteration

### The Prompt Development Loop

```
1. Write initial prompt
2. Test with representative inputs (not just the happy path)
3. Identify failure modes
4. Hypothesize why it fails (refer to Section 7)
5. Modify prompt to address root cause
6. Re-test with same inputs + new edge cases
7. Repeat until quality threshold met
```

### A/B Testing Prompts

For production systems, test prompt changes like code changes:

- Keep a test suite of input/expected-output pairs.
- Run both old and new prompts against the suite.
- Measure: accuracy, format compliance, latency, token usage.
- Don't ship a prompt change without regression testing.

### Versioning Prompts

Treat prompts as code:

```
prompts/
├── code-review/
│   ├── v1.md
│   ├── v2.md        ← added JSON output format
│   ├── v3.md        ← added severity classification
│   └── current.md   → symlink to v3.md
└── summarization/
    ├── v1.md
    └── current.md
```

Track what changed and why, just like code commits.

---

## 11. Quick Reference Cheat Sheet

| Goal | Technique |
|------|-----------|
| Simple well-defined task | Zero-shot with clear constraints |
| Establish output format | Few-shot (2-3 examples) |
| Complex reasoning | Chain-of-thought |
| High-stakes accuracy | Self-consistency (multiple runs) |
| Domain expertise | Role prompting |
| Multi-faceted problem | Decomposition |
| Calibrated confidence | Metacognitive prompting |
| Multi-step pipeline | Prompt chaining |
| Eliminate unwanted patterns | Negative prompting |
| Progressive improvement | Iterative refinement |

### The 5-Second Prompt Checklist

Before sending a prompt, verify:

- [ ] **What** — Is the task unambiguous?
- [ ] **Context** — Does the model have what it needs?
- [ ] **Format** — Did I specify the output shape?
- [ ] **Constraints** — Did I set boundaries (length, scope, exclusions)?
- [ ] **Verification** — Can I tell if the output is correct?

---

## Further Reading

- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [Anthropic Prompt Engineering Documentation](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Google — Prompt Engineering for Developers](https://cloud.google.com/vertex-ai/docs/generative-ai/learn/introduction-prompt-design)
- "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models" (Wei et al., 2022)
- "Lost in the Middle: How Language Models Use Long Contexts" (Liu et al., 2023)
