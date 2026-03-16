# Prompt Engineering in Production — Notes

---

## Topic 1 — What is a Prompt? (Production Definition)

### The Tutorial Definition vs The Production Definition

The tutorial definition: *"A prompt is a set of instructions given to an LLM."*

That is not wrong. But it is incomplete — and the gap between those two is where production systems break.

---

### What a Prompt Actually Is in Production

A prompt in production is **not one thing**. It is an assembled object made of multiple layers that gets constructed dynamically at runtime, sent to the model, and discarded. You never see it as a whole unless you log it explicitly.

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
