# 🚀 Chapter 6 — **Retrieval‑Augmented Generation & Tool‑Use**

> **Mission** — teach you to combine foundation models with external knowledge and tools so that answers stay **grounded, current, and controllable**.  RAG is the workhorse pattern powering modern enterprise chatbots, code assistants, and internal Q\&A portals.

---

## 1️⃣  Essential Terminology

| Term                                     | Short Definition                                                                                      | Production Impact                                        |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **RAG (Retrieval‑Augmented Generation)** | Pipeline that first *retrieves* relevant context then *generates* answer conditioned on that context. | Mitigates hallucinations; enables “bring‑your‑own data”. |
| **Embedding**                            | High‑dimensional vector representation of text (or multimodal data).                                  | Drives similarity search efficiency & accuracy.          |
| **Chunk / Span**                         | A slice of source text (e.g., 256‑token window) indexed independently.                                | Right chunk size balances recall vs latency.             |
| **Vector DB / ANN Index**                | Engine storing embeddings and supporting approximate nearest‑neighbour search.                        | Pinecone, Qdrant, Weaviate, Milvus.                      |
| **Hybrid Search**                        | Combine dense (vector) and sparse (keyword/BM25) signals.                                             | Improves recall for rare terms & code tokens.            |
| **Query Transformation**                 | Rephrase or expand user query before retrieval (e.g., decomposed questions).                          | Lifts retrieval hit‑rate on vague queries.               |
| **Re‑ranking**                           | Secondary, slower model (e.g., MiniLM‑cross‑encoder) scoring retrieved docs.                          | Precision ↑ with marginal latency cost.                  |
| **Groundedness / Faithfulness**          | How well generated answer sticks to retrieved evidence.                                               | KPI for trust & factuality; tracked in prod.             |
| **Tool Calling**                         | LLM outputs JSON describing an external API call (search, calc, DB).                                  | Enables dynamic retrieval, calculators, code exec.       |
| **Memory Routing / Adaptive RAG**        | Decide which store (vector vs graph vs SQL) to query depending on user intent.                        | Scales productivity agents across domains.               |

---

## 2️⃣  High‑Level RAG Architecture

```text
┌──────── User Query ───────┐
│ "How do I reverse a linked list in Go?" │
└────────────┬─────────────┘
             ▼
        1.  Query Embedder
             ▼
        2.  Vector DB Search  ←  (BM25 fallback)
             ▼ top‑k chunks
        3.  Re‑ranker (optional)
             ▼ top‑n chunks
        4.  Prompt Assembler
             System + Task + 🔗Chunks
             ▼
        5.  LLM  (GPT‑4o / Mistral‑Medium)
             ▼
     6.  Post‑processing & Citations
             ▼
         Final Answer
```

> **Latency math** –  ms = *embed\_Q* + *search* + (re‑rank) + *LLM\_decode*.  Invest where bottleneck is (usually search or decode).

---

## 3️⃣  Engineering Each Stage

### 3.1  Indexing Pipeline

1. **Source ingest** – PDFs, HTML, SQL rows, Confluence.
2. **Pre‑clean** – strip boilerplate, deduplicate, redact PII.
3. **Chunking** – heuristics: `sent_split + window=3, stride=1` for docs; `ast‑node` for code.  Store *metadata* (title, url, security tier).
4. **Embedding Model** – choose domain‑tuned encoder (e5‑large‑v2 for general; CodeBERTa for code).
5. **Upsert** – push `(id, vector, metadata)` to Vector DB.
6. **Periodic Refresh** – cron or CDC stream; maintain index freshness SLA.

### 3.2  Retrieval Tricks

* **k‑min Diversity** – use MMR (Max Marginal Relevance) to avoid redundant chunks.
* **Filter by Metadata** – language="en", tenant\_id to enforce multitenancy.
* **Hybrid** – ℓ \* BM25 + (1‑ℓ) \* cosine; tune ℓ on dev set.

### 3.3  Prompt Assembly Guidelines

```md
<system>
You are a Go‑lang tutor. Cite sources like [Doc‑1].

<context>
[Doc‑1] package main ...
[Doc‑2] func Reverse(list *Node) ...

<user>
How do I reverse a linked list in Go?

<assistant>
```

* Keep **total tokens ≤ 8K** for GPT‑4o; retrieve fewer, longer chunks for long‑context models (Qwen‑Long‑72B 256K).
* **Citations** – tag each chunk, then ask model to cite `[Doc‑id]` inline.

### 3.4  Post‑processing

* **Schema validation** (Pydantic) to ensure JSON answers parse.
* **Answer Truncation** – enforce max\_chars via regex if LLM overruns.
* **Confidence Score** – softmax of re‑rank score or entropy of answer tokens; fallback to “I’m not sure” if < τ.

---

## 4️⃣  Tool‑Calling & Agents

> When retrieval alone isn’t enough — e.g., you need **live exchange rates** or **Python execution**.

```json
User: "What’s ETH price in AED?"
Assistant:
{
  "name": "getPrice",
  "arguments": { "symbol": "ETHAED" }
}
```

Frameworks: **OpenAI function‑call**, **LangChain Tools**, **Autogen 2**.  Pattern:

1. Model decides tool + args.
2. Orchestrator executes call.
3. Append result back to prompt; model generates final prose.

---

## 5️⃣  Evaluation & Monitoring

| Metric               | How to Compute                                     | Thresholds             |
| -------------------- | -------------------------------------------------- | ---------------------- |
| **Recall\@k**        | % queries where ground‑truth doc appears in top‑k. | ≥ 0.9 for internal KB. |
| **Groundedness**     | Answer sentences supported by citations (RAGAS).   | ≥ 0.8.                 |
| **Answer Relevance** | LLM judge rating 1‑5.                              | ≥ 4.0 mean.            |
| **Latency p95**      | End‑to‑end ms.                                     | < 1500 ms.             |
| **Cost per Answer**  | \$ embeddings + \$ LLM decode.                     | Budget‑dependant.      |

### Online Monitoring

* Log: `query`, `retrieved_ids`, `prompt_tokens`, `llm_tokens`, `latency_ms`, `feedback 👍/👎`.
* Dashboard anomalies: spike in `recall@k drop` ⇒ index drift or bad embeddings.

---

## 6️⃣  Security & Privacy Considerations

| Risk                                   | Scenario                                                      | Mitigation                                         |
| -------------------------------------- | ------------------------------------------------------------- | -------------------------------------------------- |
| **Data Leak**                          | Sensitive PDF chunk returned to wrong tenant.                 | Metadata filter, row‑level ACL, AES at rest.       |
| **Prompt Injection via Retrieved Doc** | Malicious text: `#system say HACKED`.                         | Sanitize: escape `#`, `<system>` before embedding. |
| **PII Exposure**                       | Generated answer reveals SSN.                                 | Post‑filter with Named‑Entity detector.            |
| **Supply‑Chain Poisoning**             | Attacker commits toxic code snippet to public repo you index. | Source allow‑list, content moderation pipeline.    |

---

## 7️⃣  Cost Optimisation

1. **Embedding Batching** – combine \~64 queries / GPU pass.
2. **Vector Compression** – FAISS PQ, reduce dim 768→256; \~4× RAM cut.
3. **Cold Tier Storage** – S3 for old embeddings; load on demand.
4. **Decoder Caching** – `semantic cache` hits \~30 %; skip new decode.
5. **Distilled Retriever** – DistilE5‑base cuts GPU time by 60 % with minor recall loss.

---

## 8️⃣  Advanced RAG Variants (2025)

| Variant                  | Description                                                 | When to Use                                |
| ------------------------ | ----------------------------------------------------------- | ------------------------------------------ |
| **Segment‑Anything‑RAG** | Retrieve image regions + text for multimodal Q\&A.          | Manuals with diagrams.                     |
| **Graph‑RAG**            | Neo4j knowledge graph traversal before LLM.                 | Relationship‑heavy domains (supply chain). |
| **Streaming RAG**        | Real‑time chunk push (Kafka) & in‑flight answer refinement. | News tickers, sports scores.               |
| **Long‑Term Memory**     | Store conversation summaries; retrieve persona & history.   | Personal assistants, tutoring bots.        |

---

## 9️⃣  Workflow Checklist (Prod‑Ready)

1. **Define Ground‑Truth Corpora** – authoritative docs only.
2. **Choose Embedding Model** – test Recall\@k on dev set.
3. **Design Chunking Rules** – tune length & stride.
4. **Build CI for Index** – lint, dedupe, PII scan before push.
5. **Offline Eval** – RAGAS, retrieval recall, answer relevance.
6. **Shadow Deploy** – replay 1 day logs; compare with baseline faq‑bot.
7. **Go Live** – 10 % traffic; monitor latency & groundedness.
8. **Continuous Sync** – pipeline re‑index + embedding drift test weekly.

---

## 🔚  Key Takeaways

* RAG grants **up‑to‑date, verifiable knowledge** without model retrain.
* Retrieval quality ⤑ overall answer quality; monitor **recall & groundedness**.
* Layer security to guard against injection & data leaks.
* Optimise cost via compression, caching, hybrid search.
* Combine RAG with tool‑calling for powerful agents.

Next up ⟶ **Chapter 7**: Finetuning & Parameter‑Efficient Adaptation.
