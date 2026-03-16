# Prompt Engineering in Production — Notes

---

## What is a Prompt? (Production Definition)

### The Tutorial Definition vs The Production Definition

The tutorial definition: *"A prompt is a set of instructions given to an LLM."*

That is not wrong. But it is incomplete — and the gap between those two is where production systems break.

---

### What a Prompt Actually Is in Production

A prompt in production is **not one thing**. It is an assembled object made of multiple layers that gets constructed dynamically at runtime, sent to the model, and discarded. You never see it as a whole unless you log it explicitly.
```
┌─────────────────────────────────────┐
│         SYSTEM PROMPT               │  ← Static. Lives in a file/registry.
│   "You are a helpful assistant..."  │     Set by the engineer. Rarely changes.
├─────────────────────────────────────┤
│         FEW-SHOT EXAMPLES           │  ← Semi-static. Curated examples.
│   Input: X → Output: Y              │     May vary by use case or user type.
├─────────────────────────────────────┤
│         RETRIEVED CONTEXT           │  ← Dynamic. Changes every request.
│   "Here is the document: {chunk}"   │     Pulled from vector DB or memory.
├─────────────────────────────────────┤
│         USER MESSAGE                │  ← Dynamic. Changes every request.
│   "Summarise this for me..."        │     What the end user actually typed.
└─────────────────────────────────────┘
```

All four layers get **concatenated into a single string** and sent to the model as one token sequence. The model sees no boundaries between them.

---

### Why "Instructions in a File" Breaks at Scale

Storing a prompt in a `.md` or `.txt` file works for a prototype. Here is what breaks it in production:

- **No versioning** — you edit the file, the old behaviour is gone forever, you don't know what changed
- **No metadata** — you don't know which model it was written for, who wrote it, or when
- **No testing** — you pushed a change and have no idea if outputs got better or worse
- **No dynamic assembly** — the file is static but your context, user input, and retrieved data change every single call

---

### The Production Definition

> A prompt is a **versioned, dynamically assembled input contract** between your application and a language model — composed of static instructions, curated examples, and runtime-injected context — that must be treated with the same discipline as application code.

Three words matter most: **versioned, assembled, contract.**

| Word | Why It Matters |
|---|---|
| Versioned | Changing a prompt changes your system's behaviour — treat it like a code change |
| Assembled | Built at runtime from multiple sources, not read from one static file |
| Contract | The model's output is entirely determined by what you put in |

---

### What Happens When a Layer Bloats

Every model has a context window — a hard token limit for the entire assembled prompt plus the response. When retrieved context brings in 3x more text than usual:

**Scenario 1 — You hit the limit:**
The API throws an error. Request fails. You notice immediately.

**Scenario 2 — You're just under the limit:**
The model receives everything but your system prompt gets **diluted**. Instructions don't disappear — but the model drifts from intended behaviour. No error. No alert. Quietly worse outputs.

Scenario 2 is the dangerous one. You don't find out until a user screenshots something broken.

---

### Key Takeaways

- A prompt has 4 layers: System Prompt, Few-Shot Examples, Retrieved Context, User Message
- It is assembled at runtime — not read from a single file
- Bloating any layer silently degrades the entire system
- Treating prompts like code — versioned, tested, monitored — is the foundation of production prompt engineering

---

## Topic 2 — System Prompts vs User Prompts vs Few-Shot Examples

### The Three Layers — Production Reality

---

### System Prompt — What It Actually Controls

The system prompt is the **engineer's contract with the model**. The user never sees it, never touches it, and in a well-built system — never knows it exists.

It controls three things:

- **Identity** — what role the model plays (*"You are a financial analyst assistant"*)
- **Behaviour rules** — what it must and must not do (*"Never speculate. Always cite sources."*)
- **Output format** — how responses must be structured (*"Always respond in JSON with keys: summary, confidence, sources"*)

In production this is the most sensitive layer:
- If it **leaks** to the user — your entire product behaviour is exposed
- If it gets **overridden** by a clever user message — your system is compromised

Both are real attack vectors. Covered in Topics 9 and 10.

---

### User Prompt — What It Actually Controls

The user prompt is what the end user sends. But in production it is rarely *just* what the user typed. It is almost always **augmented before it hits the model:**

Raw user input:  "Summarise this contract"

Actual user prompt sent to model:
"Summarise this contract. Output must be under 150 words.
Format: bullet points. Language: formal.

Contract text:
{retrieved_document}"

The user typed 3 words. The model received 800 tokens. This is **prompt augmentation** — it happens in almost every production system.

---

### Few-Shot Examples — What They Actually Control

Few-shot examples don't just show format — they **set the model's behavioural baseline** for the entire session. The model pattern-matches from examples more strongly than it follows written instructions.

This means:
- **Bad examples teach bad behaviour** — consistently and reliably
- **Example quality > instruction quality** — a well-chosen example outperforms a perfectly written instruction
- **3-5 examples is the production sweet spot** — beyond that you burn tokens with diminishing returns

```
# Weak — instruction only
"Always respond concisely and professionally."

# Strong — example demonstrates it
User: What is RAG?
Assistant: RAG is a technique that retrieves relevant documents
at runtime and injects them into the prompt before generation.
It reduces hallucination by grounding responses in source material.
```

---

### The Production Hierarchy

| Layer | Who Controls It | Changes Per Request | Override Risk |
|---|---|---|---|
| System Prompt | Engineer | No | High — if injection succeeds |
| Few-Shot Examples | Engineer | Rarely | Low |
| User Prompt | End User | Every request | N/A — it is the input |

---

### Intra-Prompt Conflict — The Silent Bug

When your few-shot examples **contradict** your system prompt instructions, the model follows the examples — not the instructions. No error is thrown. Output just drifts.

```
System Prompt: "Always respond in formal English."

Few-Shot Example shows:
Assistant: "Hey! So RAG is basically when you grab some docs
and shove them into the prompt — pretty neat right?"
```

The model sees demonstrated behaviour and matches it. Your instruction becomes decoration.

This is **intra-prompt conflict** — one of the most common silent bugs in production prompts.

---

### Two Distinct Failure Modes

| Failure Mode | Cause | Who Breaks It |
|---|---|---|
| Intra-prompt conflict | Examples contradict instructions | You, the engineer |
| Prompt injection | User input overrides system prompt | External attacker |

Both result in the model ignoring your system prompt — for completely different reasons, requiring completely different fixes.

---

### Why Models Follow Examples Over Instructions

LLMs are trained to **predict the next token based on patterns** — not to follow rules logically. When you give an example, you show the model a pattern. When you write an instruction, you give it text to interpret.

**Pattern recognition beats text interpretation. Every time.**

---

### The Fine-Tuning Connection

Few-shot examples are essentially **runtime fine-tuning** — steering behaviour through demonstration without touching model weights.

```
Few-shot examples          Fine-tuning
in the prompt        →     on a dataset
─────────────────────────────────────────
Temporary behaviour        Permanent behaviour
Per-request cost           One-time training cost
Limited examples (3-5)     Thousands of examples
No model change            Model weights change
```

The more examples you need to get consistent behaviour — the stronger the signal that you should be fine-tuning instead.

---

### The One Rule

> **Never write an instruction for something you can demonstrate with an example.**
> **Never use an example when a hard rule is non-negotiable.**

- Examples teach: style, tone, format, reasoning pattern
- Instructions enforce: boundaries, restrictions, safety rules

---

### Key Takeaways

- System prompt is the engineer's contract — most sensitive, highest attack surface
- User prompt is always augmented in production — users never send raw input to the model
- Few-shot examples control behaviour more powerfully than instructions
- Contradicting your own layers is a silent bug — no error, just drift
- The line between few-shot prompting and fine-tuning is the number of examples needed for consistency

---
