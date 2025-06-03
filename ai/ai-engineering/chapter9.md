# 🚀 Chapter 9 — **Inference Optimisation & Cost Engineering**

> **Why it matters** — a brilliant model that costs \$0.50/query or responds in 10 s will never hit production scale. This chapter delivers concrete techniques to **speed‑up**, **scale‑out**, and **cost‑down** inference, blending modern research (2024‑2025) with hands‑on lessons from companies like OpenAI, Mistral, and Netflix.

---

## 1️⃣  Key Concepts & Metrics

| Term                  | Definition                                        | Typical Target                           |
| --------------------- | ------------------------------------------------- | ---------------------------------------- |
| **Latency (p95)**     | 95th‑percentile time from request → first token.  | < 500 ms for chat UI.                    |
| **Throughput**        | Tokens/sec/GPU or queries/sec/pod.                | ≥ 150 tok/s for 70 B @ A100.             |
| **Tokens per Dollar** | Generated tokens divided by infra \$ cost.        | > 1 M tok/\$ for 7 B QLoRA.              |
| **Compute‑Bound**     | Limited by FLOPS capability.                      | Use larger GPU clusters.                 |
| **Memory‑BW‑Bound**   | Limited by RAM ↔ GPU copy bandwidth.              | Use fused‑kernel libs, KV‑cache pinning. |
| **Batch Size**        | Concurrent sequences per forward pass.            | Auto‑tune (v‑shaped vs latency).         |
| **KV Cache**          | Attention key‑value tensors reused across tokens. | Pin in HBM to cut decode cost 3‑5×.      |

---

## 2️⃣  Bottleneck Identification Cheatsheet

| Symptom                                            | Likely Cause                                | Fix                                         |
| -------------------------------------------------- | ------------------------------------------- | ------------------------------------------- |
| Steady 300 ms *per* new token, regardless of batch | **Decode‑bound** (attention scaling O(n²)). | Flash‑Attention‑2; spec‑distil to 4‑8K ctx. |
| Big jump on first token, fast thereafter           | Embedding lookup, network overhead.         | Pre‑load vocab on GPU; HTTP/2 keep‑alive.   |
| GPU util < 40 %                                    | Small batch, I/O wait.                      | Enable dynamic batching (vLLM, TGI).        |
| p95 latency spikes nightly                         | Cold‑start pods or autoscaler churn.        | Pre‑warm min‑replicas; scheduled scale‑ups. |

---

## 3️⃣  Model‑Level Optimisations

### 3.1  Quantisation Matrix

| Bits            | Speed ↑ | Memory ↓ | Quality △ | Prod Example                      |
| --------------- | ------- | -------- | --------- | --------------------------------- |
| **16‑bit FP16** | 1.0×    | 1.0×     | 0         | OpenAI GPT‑3.5 prod.              |
| **8‑bit Int**   | 1.3×    | 1.8×     | −0.5 pp   | Mistral Instruct‑8bit API.        |
| **4‑bit NF4**   | 1.8×    | 4×       | −1 pp     | Hugging Face QLoRA models.        |
| **GPTQ 3‑bit**  | 2.4×    | 5‑6×     | −2‑3 pp   | Geekbench Llama‑3‑in‑your‑pocket. |

> **Rule‑of‑thumb** — accept ≤ 1 pp quality drop for 2‑4× cost cut. Always evaluate on *your* task.

### 3.2  Operator Fusion & Flash Attention

* **FlashAttention‑2** (Dao 2024) → 1.5‑1.9× speed on A100/H100.
* **SMoEbert** gating fused kernels → 3 % util gain.
* Use **TRT‑LLM** or **vLLM Triton** autopatched kernels.

### 3.3  Low‑Rank Adapter Merging at Serve‑Time

* Merge LoRA into base, then quantise → avoid extra matmuls.
* Netflix MediaGen saw 18 % latency reduction.

---

## 4️⃣  Service‑Level Optimisations

### 4.1  Modern Inference Runtimes

| Runtime                      | Killer Feature                                          | When to Use                  |
| ---------------------------- | ------------------------------------------------------- | ---------------------------- |
| **vLLM**                     | Paged‑attention KV cache, dynamic batching auto‑engine. | Chat apps, high fan‑out.     |
| **TGI (Text Gen Inference)** | Multi‑GPU tensor‑parallel, secure JWT.                  | Enterprise on‑prem clusters. |
| **TRT‑LLM**                  | TensorRT graph optim + quant pipelines.                 | Nvidia GPU datacentres.      |
| **ONNX‑Runtime‑LLM**         | Cross‑vendor CPU/GPU; Windows edge.                     | Local desktop, Xbox.         |

### 4.2  Dynamic Batching Logic (Pseudocode)

```python
while queue.not_empty():
    batch = gather(max_tokens=2048, timeout=5ms)
    run_inference(batch)
```

* **Trade‑off** — bigger batch ⇒ throughput ↑, latency ↑ until you cross p95 SLO.

### 4.3  KV‑Cache Sharding

* For 32‑sequence batch of Llama‑70B, KV = 23 GB context.
* vLLM pages inactive cache to host pinned memory, retaining 90 % speed.

---

## 5️⃣  Hardware & Cluster Tuning

| Lever            | Observation                               | Guideline                                     |
| ---------------- | ----------------------------------------- | --------------------------------------------- |
| **GPU Type**     | A100 vs H100 → H100 \~1.7× decode TPS.    | If SLO <500 ms and traffic >20 QPS, use H100. |
| **CPU‑GPU PCIe** | PCIe 3.0 halves KV copy speed v 4.0.      | Ensure PCIe Gen4+ or NVLink.                  |
| **Networking**   | Token streaming relies on TCP push.       | Enable TCP\_NODELAY; gRPC streaming.          |
| **Autoscaling**  | Sudden viral load at OpenAI (March 2025). | Predictive K‑means on last 7‑days traffic.    |

---

## 6️⃣  Real‑World Case Studies

### 6.1  Airbnb AI Concierge (2024)

* Problem: 40 K RPS spikes every Friday.
* Solution: vLLM + 8‑bit quant + multi‑region cache.
* Result: p90 latency 820→290 ms; infra bill −47 %/month.

### 6.2  BloombergGPT‑FinServe (2025 private cloud)

* Mixtral‑8x22B MoE with TRT‑LLM int4.
* Deployed on 32 H100 DGX nodes.
* Achieved 4.2 M tok/\$ vs 1.6 M baseline.

### 6.3  Meta Threads AI

* On‑device Llama‑3‑8B.GGUF 3‑bit (iOS).
* Distilled vocabulary to 35 K; runtime ≤ 80 ms.

---

## 7️⃣  Cost‑Control Playbook

1. **Quantise first, distil second.**
2. **Enable semantic cache** (>25 % hit saves decode cost).
3. **Use smaller expert model for first‑tokens** (Speculative decoding ala Chen 2024).
4. **Leverage Spot/Spare Instances** for traffic < SLO; Netflix saved 30 %.
5. **Monitor *tokens/user* metric** — drives cost more than QPS.

---

## 8️⃣  Monitoring & Alerting

| Metric      | Alert if                  | Tool                  |
| ----------- | ------------------------- | --------------------- |
| p95 Latency | > SLO 800 ms 5 min window | Prometheus + Grafana. |
| GPU Util    | < 50 % 10 min             | Nvidia DCGM exporter. |
| Error‑Rate  | > 0.1 %                   | Sentry.               |
| Cache Hit % | ↓20 % vs baseline         | Custom exporter.      |

Dashboard sample (PromQL):

```promql
histogram_quantile(0.95, rate(llm_request_duration_seconds_bucket[5m]))
```

---

## 9️⃣  Frontier Research (2025)

* **Speculative Decoding v2** — 5× throughput using 2‑stage student–teacher (DeepMind Gemini‑Spec).
* **Fused Multi‑Query MoE Attention** — reduces KV size by factor of experts (Ref: Microsoft NARU 2025).
* **Confidential Inference (TEEs)** — Azure Cobalt; enclaves add <7 % overhead for regulated data.
* **Liquid‑Cooling Micro‑pods** — 35 % energy savings (Meta EU Datacentres).

---

## 🔚  Key Takeaways

1. **Measure first** — profile to know if you are compute‑bound, mem‑bw‑bound, or I/O‑bound.
2. **Quant + FlashAttn + Dynamic Batch** → 2‑4× cost/perf wins with minimal quality drop.
3. **Choose runtime wisely** — vLLM for chat, TRT‑LLM for GPU‑dense clusters, ONNX for edge.
4. **Monitor & auto‑scale** — demand is spiky; guard p95 latency SLO.
5. **Iterate** — new kernels (Flash‑Attn 3) or quant methods arrive monthly.

> “Inference optimisation is **continuous DevOps**, not a one‑off tuning sprint.”
