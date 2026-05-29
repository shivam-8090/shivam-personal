# Marketing Agent Flowchart

```mermaid
flowchart TD
    A[Client / Postman / UI] --> B{Client-side cache<br/>TTL: 6 hours}
    B -- Cache hit --> A
    B -- Cache miss --> C[FastAPI<br/>marketing_agent.main]

    C --> D[Request validation<br/>ChatRequest DTO]
    D --> E[Create request_id<br/>Start latency timer]
    E --> F[LLM Router<br/>llm/router.py]

    F --> G{Provider selection}
    G -- requested_provider or default --> H[Gemini Provider<br/>Active now]
    G -. future .-> I[AWS Bedrock Provider<br/>Stub/disabled]
    G -. future .-> J[Vertex AI Provider<br/>Stub/disabled]
    G -. future .-> K[OpenAI-compatible Provider<br/>Stub/disabled]

    H --> L[LangChain LCEL Chain<br/>PromptTemplate -> Chat Model]
    I -. later .-> L
    J -. later .-> L
    K -. later .-> L

    L --> M[Gemini API call]
    M --> N[Model response<br/>text + usage metadata]

    L -. trace metadata .-> O[LangSmith<br/>Project: agentic-ai]
    O -. stores traces .-> O1[Prompt, model call,<br/>latency, metadata]

    N --> P[Cost Calculator<br/>llm/cost_tracking.py]
    P --> Q{Pricing configured?}
    Q -- Yes --> R[Calculate cost_usd]
    Q -- No --> S[cost_usd = null<br/>pricing_source = missing]

    R --> T[Query Logger<br/>observability/query_logger.py]
    S --> T

    T --> U[(MongoDB)]
    U --> U1[users]
    U --> U2[query_logs]
    U --> U3[cost_logs]
    U -. provisioned only .-> U4[vector_docs]

    T --> V[ChatResponse DTO]
    V --> W[FastAPI Response]
    W --> A

    X[Cost Monitor API<br/>/v1/cost/summary<br/>/v1/cost/models] --> U3
    U3 --> Y[Mongo aggregation<br/>by user/provider/model]
    Y --> Z[Cost summary response]

    AA[Future RAG Flow] -. not wired in v1 .-> U4
    U4 -. later retrieval .-> F
```

## End-To-End Sequence

```mermaid
sequenceDiagram
    participant Client
    participant API as FastAPI
    participant Router as LLM Router
    participant LCEL as LangChain Chain
    participant Gemini
    participant LS as LangSmith
    participant Mongo

    Client->>API: POST /v1/chat
    API->>API: Validate ChatRequest
    API->>Router: route(request)
    Router->>Router: Select provider = Gemini
    Router->>LCEL: Invoke marketing chain
    LCEL-->>LS: Send trace metadata
    LCEL->>Gemini: Prompt + user message
    Gemini-->>LCEL: Response + usage metadata
    LCEL-->>Router: AI response
    Router-->>API: ProviderResult
    API->>API: Calculate latency and cost
    API->>Mongo: Upsert users
    API->>Mongo: Insert query_logs
    API->>Mongo: Insert cost_logs
    API-->>Client: ChatResponse
```

## Component Map

| Component | File / Module | Purpose |
|---|---|---|
| API entrypoint | `main.py` | Creates FastAPI app and mounts routes |
| Chat API | `api/routes_chat.py` | Handles `/v1/chat`, `/v1/response/`, `/v1/health` |
| Cost API | `api/routes_cost.py` | Handles cost summary endpoints |
| Settings | `core/config.py` | Loads env values, feature flags, provider config |
| Mongo access | `db/mongo.py` | MongoDB client and collection accessors |
| DTOs | `models/dto.py` | Request/response schemas |
| Router | `llm/router.py` | Chooses provider and invokes LangChain chain |
| Chain | `llm/chains.py` | Marketing prompt and LCEL chain |
| Gemini provider | `llm/providers/gemini.py` | Active Gemini model adapter |
| Future providers | `llm/providers/bedrock.py`, `vertex.py`, `openai_like.py` | Provider adapters for later use |
| Cost tracking | `llm/cost_tracking.py` | Calculates cost from token usage and pricing config |
| LangSmith | `observability/langsmith_tracing.py` | Enables tracing and project config |
| Mongo logging | `observability/query_logger.py` | Writes users, query logs, and cost logs |
| Vector placeholder | `db/vector_client.py` | Future RAG/vector DB extension |

