flowchart TD
    subgraph API Layer
      A[Auth Service] --> B[Rate Limiter]
      B --> C[Request Validator]
      C --> D[Model Router]
    end

    subgraph Retrieval Layer
      D --> R1[Vector DB (Pinecone)]
      D --> R2[SQL DB / Graph DB]
      R1 & R2 --> E[Re-Ranker (MiniLM Cross-Encoder)]
    end

    subgraph Inference Layer
      E --> M1[7B LLM (8-bit)]
      E --> M2[13B LLM (4-bit+LoRA)]
      E --> M3[70B LLM (INT8+FlashAttn)]
      M1 & M2 & M3 --> O[Post-Processor]
    end

    subgraph Tool Layer
      O --> T1[Market Data API]
      O --> T2[Python Sandbox]
      O --> T3[SQL Engine]
      T1 & T2 & T3 --> O
    end

    subgraph Feedback & Monitoring
      O --> F1[Telemetry Collector]
      O --> F2[Explicit Feedback UI]
      F1 & F2 --> G[Feedback Aggregator]
      G --> H[Data Pipeline / RAG Index]
      G --> I[Prompt / Model Adjustment Pipeline]
    end
