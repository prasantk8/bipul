```mermaid
graph TD
    subgraph "Ingestion & Input"
      A1[User / External Event] --> A2[API Gateway / Pre-Filter]
      A2 --> A3[Input Guardrails (PII/Injection Filter)]
    end

    subgraph "Core Processing"
      A3 --> B1[Context Construction]
      B1 --> B2[RAG Retriever / KB Fetch]
      B2 --> B3[Tool Controller (API Calls, Python, SQL)]
      B1 --> B4[Prompt Assembler]
      B3 --> B4[Prompt Assembler]
      B4 --> B5[LLM Inference]
      B5 --> B6[Output Guardrails (Schema, Safety Check)]
    end

    subgraph "Post-Processing & Actions"
      B6 --> C1[Action Executor (DB writes, notifications)]
      B6 --> C2[Response to User]
    end

    subgraph "Observability & Feedback"
      C2 --> D1[Usage Telemetry (latency, tokens)]
      B6 --> D1
      D1 --> D2[Dashboards & Alerts]
      C2 --> D3[User Feedback (explicit/implicit)]
      D3 --> D4[Feedback Processor]
      D4 --> B1
    end

    subgraph "CI/CD & Governance"
      E1[Code / Prompt Change] --> E2[Evaluation Pipeline (Chaps 3–9)]
      E2 --> E3[Approval / Rollout]
      E3 --> A2
    end
