# 🚀 Chapter 7 — **Finetuning & Parameter‑Efficient Adaptation**

> **Objective** – equip you with a *blueprint* to adapt foundation models to your domain **safely, cheaply, and scalably**.  We cover classic full‑model finetuning, modern PEFT techniques (LoRA, QLoRA, Adapters, Prompt‑Tuning), practical hardware tips, and decision frameworks (RAG vs Finetune).

---

## 1️⃣  Quick Glossary

| Term                                      | Working Definition                                                               | Prod Relevance                                |
| ----------------------------------------- | -------------------------------------------------------------------------------- | --------------------------------------------- |
| **Finetuning**                            | Further training a pre‑trained model on task‑specific data (updates weights).    | Unlock domain style, format adherence.        |
| **SFT (Supervised Finetuning)**           | Train on `(instruction, response)` pairs.                                        | First stage of most instruction models.       |
| **RLHF**                                  | Reinforcement Learning from Human Feedback; optimise responses ranked by humans. | Aligns model with human preference & safety.  |
| **Reward Model (RM)**                     | Model predicting human preference score for a response.                          | Used as reward fn in RLHF or DPO.             |
| **PEFT (Parameter‑Efficient Finetuning)** | Adapt model by training a small subset/adapter of parameters.                    | 10‑100× cheaper vs full fine‑tune.            |
| **LoRA**                                  | Inject low‑rank matrices into attention layers; back‑prop only these.            | GPU mem ↓, training time ↓.                   |
| **QLoRA**                                 | Combine 4‑bit quantisation + LoRA; fit 65B on single GPU.                        | State‑of‑practice for < \$100 budgets.        |
| **Adapters / IA3**                        | Lightweight MLP/attention scalars trained per‑task.                              | Enables multi‑task “adapter zoo”.             |
| **Prefix / Prompt Tuning**                | Learn virtual tokens prepended to every input; base weights frozen.              | Fast hyper‑personalisation (marketing copy).  |
| **BitFit**                                | Update **only** bias terms of layers.                                            | Memory‑minimal hack; good for some NLU tasks. |
| **Catastrophic Forgetting**               | Model loses original capabilities during finetune.                               | Mitigate by low LR, adapter isolation.        |

---

## 2️⃣  Finetune vs RAG Decision Matrix

| Scenario                                      | Recommend           | Rationale                                        |
| --------------------------------------------- | ------------------- | ------------------------------------------------ |
| Need **private knowledge** that changes daily | **RAG**             | Cheaper to refresh index than retrain.           |
| Need **strict output format / code style**    | **Finetune (LoRA)** | Model must learn pattern, not just recall facts. |
| Need **low‑latency, on‑device**               | **PEFT + quantise** | Avoid external retrieval; smaller model.         |
| Heavy reasoning on **static domain** corpus   | **SFT + RLHF**      | Embed chain‑of‑thought & style.                  |

**Hybrid** is common: finetune for style, *plus* RAG for freshness.

---

## 3️⃣  Finetuning Pipelines

### 3.1  Classic 3‑Stage (OpenAI “Instruct” recipe)

```
Base Model  ➜ 1. SFT ➜ 2. RM training ➜ 3. RLHF (PPO) ➜ Aligned Model
```

* *SFT*: 5‑100 K high‑quality pairs.
* *RM*: train on `(prompt, ‑preferred, +preferred)` triples.
* *RLHF*: optimise with PPO or DPO (Direct Preference Optimisation 2023).

### 3.2  PEFT Workflow (LoRA / QLoRA)

```
Data Prep → Quantised Checkpoint → Inject LoRA adapters → Train adapters (FP16 optimizer) → Merge & Save (optional)
```

* Use **bitsandbytes 0.43+** for 4‑bit NF4 quant.
* **LoRA Config**: r=16, α=32, dropout=0.05 ; tune on eval‑set.

> 🔧 **Hardware tip** – A100 40 GB fits QLoRA of 70 B (group size 128) batch = 4.

---

## 4️⃣  Memory & Compute Optimisations

| Technique              | GPU Mem ↓                | Comment                    |
| ---------------------- | ------------------------ | -------------------------- |
| 4‑bit (NF4) quant      | ≈ 4×                     | Keep FP16 for LayerNorm.   |
| LoRA rank 8            | 1‑2 % params             | Interpolates well.         |
| DeepSpeed ZeRO‑3       | 4‑8× (shard opt & grads) | Multinode, slower startup. |
| FlashAttention‑2       | 1.2‑1.5× speed           | Compile with SM 8.0+.      |
| Gradient Checkpointing | 50 % mem                 | Adds compute overhead.     |

---

## 5️⃣  Data Engineering for Finetune

### 5.1  Data Schema

| Stage | Expected Format                                       |
| ----- | ----------------------------------------------------- |
| SFT   | `{ "instruction": str, "input": str, "output": str }` |
| RM    | `{ "prompt": str, "chosen": str, "rejected": str }`   |
| DPO   | same as RM                                            |

### 5.2  Quality Signals

* Deduplicate near‑duplicate pairs (MinHash).
* Filter profanity / PII (Presidio, OpenAI mod).
* Prefer *longer* answers for reasoning tasks (CoT).
* Balance class labels for classification to avoid bias.

### 5.3  Synthesis Strategies when Lacking Data

* **Self‑Instruct** (Wang 2023) – LLM generates tasks & answers, humans curate 5‑10 %.
* **Data Augmentation** – paraphrase inputs via back‑translation.
* **Tool‑Assisted Labeling** – use regex/search to auto‑label obvious negatives.

---

## 6️⃣  Evaluation & Overfitting Checks

| Metric Bucket           | Example Metric           | Pass Criteria |
| ----------------------- | ------------------------ | ------------- |
| **Task**                | EM / F1 on held‑out set  |               |
| **Style/Format**        | JSON schema parse rate   | ≥ 0.98        |
| **Safety**              | Toxicity (Perspective)   | ≤ 0.01        |
| **Catastrophic Forget** | LAMBADA perplexity delta | Δ < 5 %       |
| **Compute**             | Throughput tokens/s      |               |

* Use **gpt‑eval** + **human spot‑checks**.
* Track eval every epoch; early‑stop when val loss plateaus.

---

## 7️⃣  Deployment Patterns

### 7.1  Merge‑and‑Serve (LoRA → Base)

Pros: single checkpoint.
Cons: lose modularity; need FP16 for merge.

### 7.2  Adapter Stack

Load base weights frozen; hot‑swap adapters per tenant.
Pros: multi‑tenant SaaS w/ MB‑level adapters.

### 7.3  On‑Device Personal LoRA

Mobile/edge; 8‑bit base + 4‑bit LoRA.
Used by Canva Magic Design offline.

---

## 8️⃣  Security & Governance

| Risk                            | Mitigation                                          |
| ------------------------------- | --------------------------------------------------- |
| **Data licensing violation**    | Audit dataset provenance; maintain SPDX manifest.   |
| **Weights exfiltration**        | Encrypt checkpoints at rest; KMS keys.              |
| **Finetune causing toxic bias** | Include adversarial eval set; apply reward shaping. |
| **Catastrophic forgetting**     | Low LR, L2‑reg, freeze LayerNorm/embeds.            |

---

## 9️⃣  Cost Planning Cheat Sheet

| Model Size | Full FP16 Cost\* | QLoRA Cost\* | AWS p4d/hr |
| ---------- | ---------------- | ------------ | ---------- |
| 7 B        | ≈ \$20           | \$4          | 1 GPU 5 h  |
| 13 B       | \$60             | \$8          | 1 GPU 12 h |
| 70 B       | \$650            | \~\$45       | 8 GPU 24 h |

\*rough at \$3/hr per A100 40G

---

## 🔟  Advanced Research Trends (2025)

1. **MoE‑Adapter Fusion** – selective routing of tokens to finetuned experts.
2. **Differentiable Memory Patches** – attach KV memory trained for recall tasks.
3. **Synthetic SFT via Agents** – self‑play task generation >200 K examples.
4. **Sparse Finetune (LoRA‑SV)** – zero out <1 % least‑used adapter rows at runtime.
5. **Cross‑Lingual Adapter Re‑use** – train on English, transfer to 30 langs via small delta.

---

## 🪜  Step‑by‑Step Implementation Checklist

1. **Define Success Metrics** (ex: JSON validity ≥ 99 %).
2. **Collect & Clean 2‑20 K exemplars.**
3. **Choose PEFT method** (LoRA rank 16, 4‑bit).
4. **Spin‑up training env** (ft‑script + weights & biases).
5. **Run 1‑2 epoch pilot** – validate loss curve.
6. **Evaluate on held‑out + safety set.**
7. **Hyper‑tune r, α, LR.**
8. **Merge/Deploy adapter; A/B test vs base.**
9. **Monitor drift; schedule monthly recoil.**

---

## 🔚  Key Takeaways

* **PEFT delivers 90 % of gains for 10 % of cost** – default choice.
* **Data quality trumps model size**.
* **Evaluate breadth**– style, safety, forgetting.
* **Merge adapters only when necessary** – keep flexibility.

> “Train small, ship fast, iterate always.”
