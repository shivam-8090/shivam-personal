# Marketing Agent Documentation

## 1. System Overview
The Marketing Agent is a FastAPI-based GenAI service that supports Pramerica marketing workflows and insurance-related communication tasks.  
It provides:
- Normal chat
- File-aware chat (documents + images where supported)
- Optional updated output file generation
- Optional base64 return of generated files
- Provider routing (Gemini, Bedrock, Vertex, OpenAI-like)
- Context retention across sessions
- Cost and usage observability

The platform is built for production usage with logging, guardrails, provider fail-fast checks, and MongoDB-based telemetry.

## 2. Functional Summary
Primary business use cases:
- Campaign ideation and taglines
- Marketing copy generation and rewrites
- Product positioning and customer-friendly insurance messaging
- Summarization of uploaded content
- Q&A over uploaded documents/images
- File transformation output (txt/json/pptx/docx/xlsx) and optional base64 response

Supported interaction patterns:
1. Text-only interaction (`/v1/chat`, JSON)
2. File + text interaction (`/v1/chat`, multipart/form-data)
3. Backward-compatible file endpoint (`/v1/chat-with-files`)

## 3. High-Level Architecture
1. Client calls API (`/v1/chat`)
2. Request is validated and normalized
3. Guardrail pre-check runs
4. Context state is prepared (retain context by default)
5. Provider and model are selected by router
6. LLM chain executes (direct file mode or text context mode)
7. Usage + cost are calculated
8. Query log and cost log are written to MongoDB
9. Response is returned with metadata, and optional output file/base64

## 4. Core Components

### 4.1 API Layer
Key files:
- `marketing_agent/main.py`
- `marketing_agent/api/routes_chat.py`
- `marketing_agent/api/routes_cost.py`

Responsibilities:
- Endpoint routing
- Request/response shaping
- OpenAPI schema generation
- No-cache response headers
- CORS enablement

### 4.2 Routing and LLM Orchestration
Key files:
- `marketing_agent/llm/router.py`
- `marketing_agent/llm/chains.py`
- `marketing_agent/llm/prompt_registry.py`

Responsibilities:
- Resolve provider (`requested_provider` or default)
- Resolve model (`requested_model` or provider default)
- Build and invoke chain
- Use direct file mode if provider supports file attachments

### 4.3 Context Management
Key file:
- `marketing_agent/llm/context_manager.py`

Responsibilities:
- Pull recent conversation turns for same `user_id + session_id`
- Attach prior summary if present
- Estimate token load and trigger rollover if context size exceeds threshold
- Preserve continuity via `session_summary` documents

### 4.4 Guardrails
Key file:
- `marketing_agent/llm/guardrails.py`

Current checks:
- SQL injection-like patterns
- Out-of-scope content heuristics (non-business/no Pramerica context)
- Safe refusal templates from prompt config

### 4.5 File Processing and Storage
Key files:
- `marketing_agent/storage/file_service.py`
- `marketing_agent/storage/backends.py`

Capabilities:
- Ingest files from local/S3 backend
- Extract text for prompt context (including PDF extraction)
- Create output files in `txt/json/pptx/docx/xlsx` (with fallback behavior)
- Return output as base64 without local persistence when configured

Storage modes:
- Local filesystem backend
- S3 backend via configurable bucket and prefixes

### 4.6 Provider Layer
Key files:
- `marketing_agent/llm/providers/gemini.py`
- `marketing_agent/llm/providers/bedrock.py`
- `marketing_agent/llm/providers/vertex.py`
- `marketing_agent/llm/providers/openai_like.py`

Bedrock-specific highlights:
- Supports direct file converse (`document` and `image` content blocks)
- Supports model provider resolution for ARNs/IDs
- Supports CA bundle for TLS trust
- Supports Bedrock bearer token env flow

### 4.7 Observability and Cost
Key files:
- `marketing_agent/observability/query_logger.py`
- `marketing_agent/llm/cost_tracking.py`
- `marketing_agent/api/routes_cost.py`

Features:
- Query success/failure logging
- Cost log per request with token usage
- Cost summary endpoints with recalculation fallback when stored cost is null

### 4.8 Data Layer
Key files:
- `marketing_agent/db/mongo.py`
- `marketing_agent/db/schemas.py`

Collections:
- `users`
- `query_logs`
- `cost_logs`
- `vector_docs` (reserved for future RAG)

## 5. API Specification

### 5.1 `POST /v1/chat`
Single primary endpoint supporting two content types:

1. `application/json`
- Standard chat + optional staged file references

2. `multipart/form-data`
- Direct file upload under `files`
- Supports all major chat parameters

Core request fields:
- `user_id` (required)
- `session_id` (required)
- `message` (required)
- `retain_context` (default: `true`)
- `requested_provider`
- `requested_model`
- `temperature`
- `input_files` (optional references)
- `produce_updated_file` (bool)
- `return_updated_file_b64` (bool)
- `output_file_path` (optional persistence path)
- `output_format` (txt/json/pptx/docx/xlsx)
- `metadata`

Core response fields:
- `request_id`
- `reply`
- `provider`, `model`
- `route_decision`
- `latency_ms`
- `usage` (input/output/total tokens)
- `cost_usd`
- `output_file_uri`
- `output_file_type`
- `output_file_name`
- `output_file_b64`

### 5.2 `POST /v1/chat-with-files`
Backward compatibility endpoint for file form uploads. Internally routes to the shared chat execution path.

### 5.3 `GET /v1/health`
Health payload includes:
- overall status
- mongo readiness
- provider readiness map
- LangSmith configuration indicator

### 5.4 `GET /v1/history`
Returns historical query log records filtered by `user_id` and/or `session_id`.

### 5.5 `GET /v1/files`
Single endpoint for:
- listing input/output files from chat metadata
- downloading a selected input/output file

### 5.6 Cost Endpoints
- `GET /v1/cost/summary`
- `GET /v1/cost/models`

## 6. File Input/Output Behavior

### 6.1 Input files
- Files are staged via backend (local/S3)
- For providers with direct file support (example: Bedrock), attachments are passed as native file content blocks
- For providers without direct file support, text extraction is used as context

### 6.2 Output files
File generation is triggered when:
- `produce_updated_file=true`, or
- output settings are provided, or
- message appears to request edits and input files are present

Output format resolution:
1. Explicit `output_format` (highest priority)
2. Inferred from message keywords (presentation/json/excel/text)
3. Fallback to `txt`

### 6.3 Base64-only output mode
If:
- `return_updated_file_b64=true`
- and `output_file_path` is not provided  

Then output is rendered in memory and returned as base64 without persisting to local output path.

## 7. Context and Session Behavior
- `retain_context=true` by default
- Same `session_id` continues conversation memory
- Context manager pulls recent successful turns (`CONTEXT_HISTORY_TURNS`)
- If estimated context exceeds `CONTEXT_MAX_TOKENS`, session rollover summary is generated

## 8. Prompting and Domain Scope
Prompt policy is maintained in:
- `marketing_agent/prompts/prompts.yaml`

Intent:
- Marketing-first assistant behavior
- Insurance-domain support for communication tasks
- Guardrails for safety, business relevance, and data handling

## 9. Costing Model
Cost is computed from token usage and configured pricing:
- Env: `MARKETING_MODEL_PRICING_JSON`
- Key format supported:
  - `provider:model`
  - `model`

If price is missing:
- `cost_usd` may be null
- Cost APIs can recompute from current pricing config when possible

## 10. Configuration Matrix (Important Env Variables)

Platform:
- `MARKETING_API_PREFIX`
- `ENABLE_QUERY_LOGGING`

Providers:
- `MARKETING_DEFAULT_PROVIDER`
- `ENABLE_PROVIDER_GEMINI`
- `ENABLE_PROVIDER_BEDROCK`
- `ENABLE_PROVIDER_VERTEX`
- `ENABLE_PROVIDER_OPENAI_LIKE`

Bedrock:
- `AWS_REGION`
- `MARKETING_BEDROCK_MODEL_ID`
- `MARKETING_BEDROCK_API_KEY` (if used in your environment)
- `AWS_CA_BUNDLE` / `REQUESTS_CA_BUNDLE` (if TLS trust chain required)

Mongo:
- `MONGODB_URI` or `MONGO_CNXN_STRING`
- `MONGODB_DB_NAME` or `DB_PREFIX`
- optional collection overrides/prefixes

Context:
- `CONTEXT_MAX_TOKENS`
- `CONTEXT_HISTORY_TURNS`

Storage:
- `MARKETING_STORAGE_BACKEND` (`local` or `s3`)
- `MARKETING_LOCAL_STORAGE_ROOT`
- `MARKETING_S3_BUCKET`
- `MARKETING_S3_INPUT_PREFIX`
- `MARKETING_S3_OUTPUT_PREFIX`

Observability:
- `ENABLE_LANGSMITH_TRACING`
- `LANGSMITH_API_KEY`
- `LANGSMITH_PROJECT`

## 11. Security and Governance Controls
1. SQL injection pattern blocking before model call
2. Out-of-scope request detection
3. Strict request/response schema validation via Pydantic
4. Metadata-aware logging for auditability
5. Sensitive URI masking in health output
6. Path constraints for local file reads/writes
7. Optional no-persistence mode via base64 response
8. Provider configuration gating (fail if not configured)
9. Prompt-level business scope and safety instructions
10. Request-level trace context in Mongo operation logs

## 12. Non-Functional Characteristics
- Horizontal API scalability through stateless FastAPI workers
- Centralized persistence via MongoDB
- Pluggable provider strategy for failover/multi-vendor
- Operational transparency through route decisions, latency, and usage logging

## 13. Known Operational Dependencies
- MongoDB availability is mandatory for healthy state
- Provider SDKs and credentials must be present for active provider usage
- Optional libraries improve file fidelity:
  - `python-pptx` for PPTX
  - `python-docx` for DOCX
  - `openpyxl` for XLSX

Fallback behavior exists if optional libraries are unavailable.

## 14. Deployment and Run
From project root:

```powershell
cd c:\Users\P043123\Work_Projects\Agentic-Ai
.\venv\Scripts\Activate.ps1
python -m uvicorn marketing_agent.main:app --reload --port 8004
```

## 15. Testing Checklist
1. Health endpoint readiness (`/v1/health`)
2. JSON normal chat (`/v1/chat`)
3. Multipart chat with file upload (`/v1/chat`)
4. Context continuity over same session
5. Updated file generation in each format
6. Base64-only output mode without file persistence
7. Cost summary/model endpoints after sample traffic
8. `/v1/files` list and download workflows

## 16. Suggested Future Enhancements
1. Add RAG retrieval against `vector_docs` for marketing knowledge packs
2. Add fine-grained role-based access and policy controls
3. Add async batch processing for large file transformations
4. Add prompt/version registry with A/B evaluation
5. Add SLO dashboards for latency, error rates, and cost drift
