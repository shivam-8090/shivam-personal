# Marketing Agent Architecture

This document explains the end-to-end Marketing Agent solution implemented in
this folder. The service is intentionally modular so Gemini can run today while
Bedrock, Vertex AI, and OpenAI-compatible providers can be enabled later without
rewriting the API layer.

## 1. High-Level Flow

The current v1 flow is:

1. A client calls the FastAPI API.
2. FastAPI validates the request using Pydantic DTOs.
3. The LLM router selects a provider. Gemini is the default active provider.
4. The router builds and invokes a LangChain LCEL marketing chain.
5. LangSmith captures chain-level traces when enabled.
6. The response, route metadata, latency, token usage, and cost estimate are
   written to MongoDB.
7. FastAPI returns the model response and usage metadata to the client.

RAG is not wired into the main flow yet. A vector document collection and client
placeholder exist so future RAG work can be added without changing the main API
contract.

## 2. Main Components

### FastAPI Layer

Location:

- `main.py`
- `api/routes_chat.py`
- `api/routes_cost.py`
- `models/dto.py`

Responsibilities:

- Exposes the HTTP API.
- Validates incoming request and outgoing response shapes.
- Provides health checks.
- Returns chat responses to the client.
- Exposes cost monitoring endpoints backed by MongoDB aggregation.

Current endpoints:

- `GET /v1/health`
- `POST /v1/chat`
- `POST /v1/response/`
- `GET /v1/cost/summary`
- `GET /v1/cost/models`

`POST /v1/response/` is a compatibility alias for projects that expect an
advisor-style endpoint. The preferred endpoint for the new service is
`POST /v1/chat`.

### Configuration Layer

Location:

- `core/config.py`

Responsibilities:

- Loads `.env` / `.env.development` / `.env.production`.
- Centralizes API keys, MongoDB settings, feature flags, provider settings, and
  pricing config.
- Keeps the rest of the code from reading environment variables directly.

Important environment variables:

```env
GEMINI_API_KEY=...
MONGO_CNXN_STRING=...
DB_PREFIX=...
ENABLE_LANGSMITH_TRACING=true
LANGSMITH_API_KEY=...
LANGSMITH_PROJECT=marketing-agent-local
MARKETING_DEFAULT_PROVIDER=gemini
ENABLE_PROVIDER_GEMINI=true
```

Optional pricing:

```env
MARKETING_MODEL_PRICING_JSON={"gemini:gemini-2.0-flash":{"input_per_1k":0.0,"output_per_1k":0.0}}
```

### LLM Provider Layer

Location:

- `llm/providers/base.py`
- `llm/providers/gemini.py`
- `llm/providers/bedrock.py`
- `llm/providers/vertex.py`
- `llm/providers/openai_like.py`

Responsibilities:

- Defines one standard provider interface.
- Wraps each provider-specific LangChain chat model.
- Extracts token usage metadata from provider responses.
- Makes the router independent of provider-specific SDK details.

Current provider status:

- Gemini: implemented and intended for current use.
- Bedrock: implemented behind config and package availability, disabled by
  default.
- Vertex AI: implemented behind config and package availability, disabled by
  default.
- OpenAI-compatible: implemented behind config and package availability,
  disabled by default.

### Router And Orchestration Layer

Location:

- `llm/router.py`
- `llm/chains.py`

Responsibilities:

- Selects the provider for each request.
- Builds the LangChain marketing chain.
- Invokes the chain with LangSmith metadata.
- Returns a normalized provider result to the API layer.

Current routing strategy:

- If `requested_provider` is supplied in the request, use that provider.
- Otherwise use `MARKETING_DEFAULT_PROVIDER`.
- If `requested_model` is supplied, use it for that provider call.
- If the provider is disabled or missing credentials, return a clear error.

Current chain strategy:

- Uses a LangChain `ChatPromptTemplate`.
- Pipes the prompt into the selected chat model using LCEL.
- Applies a corporate marketing assistant system prompt.

Future router improvements:

- Add task-based routing.
- Add a classifier chain for provider/model selection.
- Move to LangGraph if the flow becomes multi-step with tools, RAG, approval
  gates, or retries.

### Observability Layer

Location:

- `observability/langsmith_tracing.py`
- `observability/query_logger.py`

Responsibilities:

- Configures LangSmith tracing from environment variables.
- Adds trace metadata such as user, session, department, provider, and model.
- Writes product-level logs to MongoDB.

LangSmith and MongoDB serve different needs:

- LangSmith is for chain-level debugging, prompt inspection, and future
  evaluation workflows.
- MongoDB is for operational reporting, usage analytics, cost monitoring, and
  audit-style request history.

### MongoDB Layer

Location:

- `db/mongo.py`
- `db/schemas.py`
- `db/vector_client.py`

Responsibilities:

- Creates the MongoDB client.
- Provides named collection accessors.
- Defines expected document shapes.
- Provides a placeholder vector client for future RAG.

Collections:

- `users`
- `query_logs`
- `cost_logs`
- `vector_docs`

Indexes can be created once after MongoDB is reachable:

```powershell
python -c "from marketing_agent.dependencies import get_store; get_store().ensure_indexes()"
```

## 3. MongoDB Collection Details

### users

Stores the latest known user/session metadata.

Important fields:

- `user_id`
- `department`
- `name`
- `session_id`
- `updated_at`

### query_logs

Stores request and response history.

Important fields:

- `request_id`
- `user_id`
- `session_id`
- `query_text`
- `response_text`
- `provider`
- `model`
- `route_decision`
- `latency_ms`
- `status`
- `error`
- `metadata`
- `timestamp`

### cost_logs

Stores token and cost telemetry for cost APIs.

Important fields:

- `request_id`
- `user_id`
- `session_id`
- `provider`
- `model`
- `tokens_used.input_tokens`
- `tokens_used.output_tokens`
- `tokens_used.total_tokens`
- `cost_usd`
- `pricing_source`
- `timestamp`

### vector_docs

Provisioned for future RAG.

Important fields:

- `doc_id`
- `embedding`
- `metadata`
- `source`
- `created_at`

The main chat route does not read from `vector_docs` in v1.

## 4. Request Lifecycle

### Chat Request

Client sends:

```json
{
  "user_id": "test_user_001",
  "session_id": "session_001",
  "department": "marketing",
  "user_name": "Test User",
  "message": "Create a campaign idea for young professionals.",
  "requested_provider": "gemini",
  "requested_model": "gemini-2.0-flash",
  "temperature": 0.4,
  "metadata": {
    "client": "local-test"
  }
}
```

Only `user_id`, `session_id`, and `message` are required.

### Processing Steps

1. `routes_chat.py` creates a `request_id` and starts a timer.
2. `LLMRouter.route()` chooses the provider.
3. `build_marketing_chain()` creates the LCEL chain.
4. The selected LangChain chat model is invoked.
5. Usage metadata is extracted from the model response.
6. `CostCalculator` computes cost from configured pricing when possible.
7. `QueryLogger` writes `users`, `query_logs`, and `cost_logs`.
8. FastAPI returns `ChatResponse`.

### Chat Response

Example response shape:

```json
{
  "request_id": "generated-uuid",
  "user_id": "test_user_001",
  "session_id": "session_001",
  "reply": "Generated marketing response...",
  "provider": "gemini",
  "model": "gemini-2.0-flash",
  "route_decision": "provider=gemini;strategy=requested_or_default",
  "latency_ms": 1234,
  "usage": {
    "input_tokens": 50,
    "output_tokens": 120,
    "total_tokens": 170
  },
  "cost_usd": null
}
```

`cost_usd` is `null` when pricing is not configured or token metadata is not
available.

## 5. Cost Monitoring Flow

The cost APIs read from `cost_logs`; they do not call the LLM provider.

### User/Model Summary

Endpoint:

```text
GET /v1/cost/summary?user_id=test_user_001
```

Behavior:

- Filters by user if `user_id` is supplied.
- Filters by date if `from_timestamp` or `to_timestamp` is supplied.
- Groups by `user_id`, `provider`, and `model`.
- Returns total cost, total tokens, and request count.

### Model Summary

Endpoint:

```text
GET /v1/cost/models
```

Behavior:

- Groups by `provider` and `model`.
- Useful for dashboards showing spend by LLM provider/model.

## 6. LangSmith Flow

When enabled, the app sets LangSmith environment variables during startup:

- `LANGCHAIN_TRACING_V2=true`
- `LANGSMITH_API_KEY`
- `LANGCHAIN_API_KEY`
- `LANGSMITH_PROJECT`
- `LANGCHAIN_PROJECT`

The router passes metadata into the LangChain invocation:

- `user_id`
- `session_id`
- `department`
- `provider`
- `model`

Expected LangSmith trace:

1. Prompt template run.
2. Gemini chat model run.
3. Final LCEL chain output.

If traces do not appear:

- Confirm `ENABLE_LANGSMITH_TRACING=true`.
- Confirm `LANGSMITH_API_KEY` is valid.
- Confirm `LANGSMITH_PROJECT` is the project you are checking.
- Restart the API after editing `.env`.
- Send a fresh `/v1/chat` request.

## 7. Error Handling

Provider setup errors return HTTP `503`.

Examples:

- Provider requested but disabled.
- Provider enabled but missing API key.
- Bedrock enabled without AWS region or credentials.

Unexpected runtime failures return HTTP `500` and are logged in `query_logs`
with status `error` when query logging is enabled.

## 8. Future Extension Points

### Add Bedrock

1. Install `langchain-aws`.
2. Configure AWS credentials and region.
3. Set:

```env
ENABLE_PROVIDER_BEDROCK=true
AWS_REGION=...
MARKETING_BEDROCK_MODEL_ID=...
```

4. Test with:

```json
{
  "requested_provider": "bedrock",
  "message": "Create a campaign idea..."
}
```

### Add Vertex AI

1. Install `langchain-google-vertexai`.
2. Configure Google Cloud credentials.
3. Set:

```env
ENABLE_PROVIDER_VERTEX=true
GOOGLE_PROJECT_ID=...
VERTEX_LOCATION=us-central1
MARKETING_VERTEX_MODEL=...
```

### Add OpenAI-Compatible Providers

1. Install `langchain-openai`.
2. Set:

```env
ENABLE_PROVIDER_OPENAI_LIKE=true
OPENAI_API_KEY=...
OPENAI_BASE_URL=...
MARKETING_OPENAI_MODEL=...
```

### Add RAG

RAG should be added in the chain layer, not directly inside the API route.

Recommended path:

1. Finalize vector DB choice.
2. Implement real search in `db/vector_client.py`.
3. Add a retrieval chain next to `llm/chains.py`.
4. Extend router decisions to choose general marketing chat vs RAG-assisted
   marketing response.
5. Log retrieved document IDs in `query_logs.metadata`.

## 9. Local End-To-End Test Checklist

1. Set Gemini and MongoDB environment variables.
2. Optionally set LangSmith environment variables.
3. Start the API:

```powershell
python -m uvicorn marketing_agent.main:app --reload --port 8004
```

4. Check health:

```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:8004/v1/health" -Method Get
```

5. Send a chat request:

```powershell
$body = @{
  user_id = "test_user_001"
  session_id = "session_001"
  department = "marketing"
  user_name = "Test User"
  message = "Create a 3-line life insurance campaign for young professionals."
} | ConvertTo-Json

Invoke-RestMethod `
  -Uri "http://127.0.0.1:8004/v1/chat" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

6. Check MongoDB Compass collections.
7. Check LangSmith project if tracing is enabled.
8. Check cost summary:

```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:8004/v1/cost/summary" -Method Get
```

