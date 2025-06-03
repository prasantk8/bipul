# 🚀 Chapter 5: Prompt Engineering

Prompt engineering is the craft of **designing, structuring, and safeguarding** the instructions that drive foundation‑model behaviour. Because modern models already possess vast latent capabilities, the right prompt often unlocks surprising performance without altering model weights.

---

## 📌 Core Terminology & Definitions

| Term                                    | Explanation                                                                                                              |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Prompt**                              | The full instruction sent to a model (may include system, user, tool‑calling sections).                                  |
| **Task Description**                    | Natural‑language specification of what the model should do, including role and output format.                            |
| **Shot**                                | One *(input → output)* example embedded inside the prompt. Zero‑shot ⇒ no examples; few‑shot ⇒ ≤ k examples (k≪dataset). |
| **In‑Context Learning (ICL)**           | The model infers patterns from examples inside the *context window* instead of weight updates.                           |
| **Chain‑of‑Thought (CoT)**              | Technique that asks a model to think step‑by‑step before emitting a final answer.                                        |
| **Self‑Consistency**                    | Sampling multiple CoT traces, then majority‑voting on final answers for higher reliability.                              |
| **Temperature / Top‑k / Top‑p**         | Sampling knobs controlling randomness vs. determinism in generation.                                                     |
| **System Prompt**                       | High‑level, hidden instruction that sets global behaviour (e.g., brand voice, safety policy).                            |
| **Prompt Injection / Jailbreak**        | Malicious or accidental user text that overrides the intended system instructions.                                       |
| **Robustness**                          | Model’s resilience to harmless prompt perturbations (capitalisation, ordering) and adversarial attacks.                  |
| **Function / Tool Calling**             | Prompt pattern that guides the model to output structured JSON describing API calls.                                     |
| **Retrieval‑Augmented Prompting (RAP)** | Dynamically inserting retrieved passages into the prompt (RAG without vector DB).                                        |
| **LoRA / Adapter / Prefix‑Tuning**      | Parameter‑efficient tuning methods that *augment* prompts when ICL alone under‑performs.                                 |

---

## 📌 Anatomy of an Effective Prompt

```text
[System]
  • Role + high‑level policy + constraints
[User]
  • Task description (clear, concise)
  • Optional: Examples (few‑shot / CoT)
  • Optional: External context (RAP / RAG)
[Assistant]
  • (Left empty) – model fills here
```

> **Best‑practice ordering**: Most decoder‑only LLMs (GPT‑4o, Mistral, Qwen) perform best when the *task description precedes examples*. Llama‑3 empirically favours examples‑first. Always A/B‑test.

### Prompt Components

1. **Role specification** – *“You are a senior legal analyst…”*
2. **Explicit constraints** – length, language, style, output schema.
3. **High‑quality exemplars** – Demonstrate desired input→output pairs.
4. **Reasoning cues** – *“Think step‑by‑step”*, *“Use bullet points”*.
5. **Guard phrases** – Prevent unwanted behaviour (*“If unsure, reply: ‘I don’t know.’”*).

---

## 📌 Prompt Engineering Techniques & Patterns

| Technique                              | Purpose                                              | Mini‑Example                                                          |
| -------------------------------------- | ---------------------------------------------------- | --------------------------------------------------------------------- |
| **Zero‑shot**                          | Quick baseline; relies on model’s latent knowledge.  | *“Translate to Arabic: ‘Climate change is real.’”*                    |
| **Few‑shot ICL**                       | Teach task structure, style; reduces hallucinations. | Provide 3 Q\&A pairs before new question.                             |
| **CoT Prompting**                      | Unlock reasoning tasks (math, logic).                | *“Let’s solve step‑by‑step …”*                                        |
| **Self‑Consistency**                   | Increase accuracy by sampling N CoTs then voting.    | Sample 10 traces; choose majority answer.                             |
| **Tree‑of‑Thought / Graph‑of‑Thought** | Parallel exploration, prune best path.               | Reason about complex planning problems.                               |
| **Function Calling**                   | Structured tool invocation (OpenAI JSON mode).       | Model outputs `{ "name": "getWeather", "args": { "city": "Dubai" } }` |
| **RAP / RAG**                          | Inject factual passages to ground responses.         | Append “### CONTEXT” with retrieved docs.                             |
| **Style Transfer Prompts**             | Enforce tone & persona.                              | *“Rewrite in Shakespearean English …”*                                |
| **Negative / Anti‑Prompts**            | Ban sensitive topics.                                | *“Do NOT mention political affiliation.”*                             |
| **Meta‑Prompting**                     | Ask the model to improve its own prompt.             | *“Rewrite the above prompt to be clearer.”*                           |

---

## 📌 Robustness & Security

### Threats

* **Direct‑prompt injection**: *“Ignore previous instructions and …”*
* **Indirect/contextual**: Malicious data retrieved via RAG.
* **Semantic attacks**: Homoglyphs, unicode exploits.

### Defences

1. **Input sanitisation & classification** – Detect PII, policy violations.
2. **Role hierarchy** – System > developer > user; enforce with structured chat schemas (ChatML).
3. **Output filtering** – Post‑generation moderation (OpenAI Moderation, Guardrails.ai).
4. **Adversarial testing / red‑teaming** – Simulate attacks during QA.
5. **Structured JSON mode** – Limits free‑text jailbreak vectors.

---

## 📌 Advanced Prompting Trends (2025)

| Trend                                  | Insight                                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Multimodal Prompts**                 | Models like GPT‑4o accept text + image + audio. Prompts now embed `<image>` placeholders and refer to regions (*“In the top‑left corner, …”*). |
| **Long‑Context Windows**               | 256K‑token models (Qwen‑Long) enable in‑context “database” prompts – but require chunking + retrieval to avoid dilution.                       |
| **Agentic Prompt Loops**               | Orchestrators (LangGraph, CrewAI) create *task → plan → execute → reflect* cycles via hierarchical prompts.                                    |
| **Program‑Aided LLMs (PAL / Gorilla)** | The LLM writes code; external interpreter returns results; prompt fosters tool‑use reasoning.                                                  |
| **Semantic Caching**                   | Embed‑based cache keys to reuse identical prompt intents, slashing latency & cost.                                                             |

---

## 📌 Mindmap of Chapter 5

```
Prompt Engineering
├── Anatomy of a Prompt
│   ├── System
│   ├── User (task)
│   ├── Examples
│   └── Context/RAG
│
├── Techniques
│   ├── Zero‑shot / Few‑shot
│   ├── CoT & Self‑Consistency
│   ├── Function Calling
│   ├── RAP / RAG
│   └── Style / Negative / Meta
│
├── Sampling Knobs
│   ├── Temperature
│   ├── Top‑k
│   └── Top‑p
│
├── Robustness & Security
│   ├── Threats (injection, semantic)
│   └── Defences (sanitise, role hierarchy, red‑team)
│
└── Advanced Trends
    ├── Multimodal
    ├── Long Context
    ├── Agentic Loops
    └── PAL & Semantic Cache
```

---

## 📌 Actionable Next Steps

1. **Build a Prompt Library** – Store tested prompts alongside metadata (task, model, metrics).
2. **A/B‑Test Prompt Variants** – Measure impact on accuracy, latency, cost.
3. **Automate Red‑Team Scripts** – Continuously probe for jailbreaks; patch prompts accordingly.
4. **Experiment with Function Calling** – Create small tool‑use demos (e.g., weather API) to internalise schema prompting.
5. **Monitor Prompt Drift** – Periodically re‑evaluate prompts as models update (e.g., GPT‑4o behavioural shifts).

---

Mastering prompt engineering empowers you to squeeze maximal value from foundation models **before** incurring heavier costs of fine‑tuning. In the next chapter we’ll explore Retrieval‑Augmented Generation (RAG) and context‑construction strategies to push performance even further.
