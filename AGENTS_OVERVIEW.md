# Agentic-Ai Architecture and Agent Overview

This repository contains multiple agents organized as separate FastAPI applications. The top-level `app.py` acts as a router and exposes the combined API surface.

## 1. Overall Architecture

- `app.py` (root)
  - Creates a FastAPI app with CORS enabled.
  - Mounts four sub-applications:
    - `/genai` → `advisor/main.py`
    - `/customer` → `customer_service/main.py`
    - `/Intranet` → `Intranet/main.py`
    - `/master-agent` → `master_agent/app.py`
  - Exposes a shared health endpoint at `/health`.

- Each mounted service is its own FastAPI app with its own request models, business logic, and environment configuration.
- The `master_agent` app acts both as a router and as a fallback conversational assistant.

---

## 2. `master_agent` — Query Router and General Assistant

### Key file: `master_agent/app.py`

### Purpose

The `master_agent` app receives a user query, classifies it, and forwards it to one or more specialized services.

### Main flow

1. Load environment and authentication settings from `.env.development`.
2. Initialize a deterministic Gemini LLM via `ChatGoogleGenerativeAI(model="gemini-2.5-flash", temperature=0)`.
3. Expose two main endpoints:
   - `/login` — returns a JWT token for demo users.
   - `/route/` — accepts a routed query and requires a valid Bearer token.
4. In `/route/`:
   - Validate and screen the request payload.
   - Call `classify_query(question)` to decide which sub-agent(s) should handle the query.
   - Supported agents: `Advisor`, `Customer Service`, `Intranet`, `Master Agent`.
   - Route requests to:
     - `route_to_advisor()`
     - `route_to_customer_service()`
     - `route_to_intranet()`
     - `master_agent_response()` for general/chat queries.
   - Combine multiple agent responses into one consolidated message.

### Classification behavior

- Uses an LLM prompt that defines allowed agents and business context.
- Returns one or more agent names, comma-separated.
- If multiple agents are selected, it removes `Master Agent` unless it is the only option.
- Falls back to `Master Agent` if classification fails.

### Master Agent fallback

- Uses `master_agent_response(question)`.
- Maintains a global in-memory `conversation_history` list.
- Provides generic corporate-style conversational responses when no specialist agent is needed.

### Routing details

- `route_to_advisor()` sends the request to `advisor.main.assistant.process_query()`.
- `route_to_customer_service()` handles login vs authenticated queries using `customer_service.main.login` and `customer_service.main.assistant.process_query()`.
- `route_to_intranet()` sends the request to `Intranet.main.assistant.process_query()`.

---

## 3. `advisor` — Insurance advisor and product assistant

### Key files
- `advisor/main.py`
- `advisor/services.py`
- `advisor/VectorDB/` (FAISS vector store files)

### Purpose

Handles insurance advisor questions related to Pramerica products, policy status, earnings, and other advisor-level queries.

### Main flow

1. Load environment with `advisor.config.load_environment()`.
2. Initialize MongoDB via `advisor.dependencies.get_db()`.
3. Create a Gemini model `gemini-2.0-flash` for assistant tasks.
4. Initialize `PramericaAssistant` with:
   - LLM instance
   - MongoDB database
   - FAISS vector store from `advisor/VectorDB`
5. Expose `/response/` endpoint that requires `Authorization: Bearer ...`.
6. In `QA_bot()`:
   - Validate the bearer token header format.
   - Call `assistant.process_query(...)`.
   - Return structured answer metadata including cache IDs, base64 artifacts, and response time.

### Business logic in `advisor/services.py`

- `PramericaAssistant.process_query()` is the main entry.
- It classifies the incoming query into a taxonomy of insurance intents.
- Supported categories include:
  - `PRODUCT_INFO`
  - `LAPSED_POLICIES_INFO`
  - `GRACE_PERIOD_POLICIES_INFO`
  - `UPCOMING_RENEWAL_POLICIES_INFO`
  - `MATURED_POLICIES_INFO`
  - `CANCELLED_POLICIES_INFO`
  - `TERMINATED_POLICIES_INFO`
  - `ACTIVE_POLICIES_INFO`
  - `NEW_BUSINESS_EARNING_INFO`
  - `POLICY_INFO`
  - `PREMIUM_CERTIFICATE`
  - `PREMIUM_RECEIPT`
  - `UNIT_STATEMENT`
  - `WIP_REPORT`
  - `NAV_INFO`
  - `COMMISSION_STATEMENT`
  - `AGENT_DETAILS`
  - `FORM_16A`
  - `HYBRID`
  - `GENERAL`

- Query handlers either:
  - use retrieval-augmented generation (RAG) with FAISS,
  - call internal APIs,
  - or construct a prompt for the LLM.
- Responses may also be cached in MongoDB.

### RAG and retrieval

- `HybridRetriever` performs semantic similarity search plus keyword filtering.
- The advisor assistant uses RAG for product-related queries.
- FAISS index is loaded from `advisor/VectorDB`.

---

## 4. `customer_service` — Customer profile, login, and update bot

### Key files
- `customer_service/main.py`
- `customer_service/services.py`

### Purpose

Handles customer login, profile updates, document attachments, and customer-service-style questions.

### Main flow

1. Load environment with `customer_service.config.load_environment()`.
2. Initialize MongoDB via `customer_service.dependencies.get_db()`.
3. Create Gemini model and a `CustomerUpdateBot` instance.
4. Initialize `OCR` and `Customer_Login` helper objects.
5. Expose `/response/`, `/message-history/`, and `/feedback/` endpoints.

### `/response/` behavior

- If `client_id` is missing, the endpoint treats the request as a login attempt.
- If `attachment` exists, it sends the request to OCR processing.
- Otherwise, it calls `assistant.process_query(...)`.

### Customer service logic

- `CustomerUpdateBot.process_query()` classifies intent and routes to specific handlers.
- Supported intents include:
  - `UPDATE_PHONE`, `PROVIDING_PHONE_NUMBER`, `PROVIDING_PHONE_OTP`
  - `UPDATE_EMAIL`, `PROVIDING_EMAIL_ID`, `PROVIDING_EMAIL_OTP`
  - `UPDATE_ADDRESS`, `PROVIDING_ADDRESS`, `ADDRESS_CONFIRMATION`, `PROVIDING_LANDMARK`
  - `MULTIPLE_UPDATES`
  - `INFORMATION_REQUEST`
  - `GENERAL_QUERY`

- The agent performs real-world update flows:
  - OTP extraction and verification
  - Phone/email extraction
  - API calls to internal OTP services
  - Address update guidance

### Other endpoints

- `/message-history/` returns stored message history.
- `/feedback/` updates feedback for a MongoDB history record.

---

## 5. `Intranet` — Employee/internal knowledge assistant

### Key files
- `Intranet/main.py`
- `Intranet/services.py`
- `Intranet/VectorDB/` (FAISS index files)

### Purpose

Handles internal employee questions, HR policies, and intranet-style knowledge retrieval.

### Main flow

1. Load environment with `Intranet.config.load_environment()`.
2. Initialize MongoDB via `Intranet.dependencies.get_db()`.
3. Create Gemini model `gemini-2.5-flash`.
4. Create `PramericaAssistant` with the Intranet FAISS store.
5. Expose `/response/` endpoint with IP whitelist protection.

### Request handling

- `QA_bot()` validates `emp_id` and `session_id`.
- Calls `assistant.process_query(...)`.
- Returns query response, answer, and metadata.

### Note

- This service is structured similarly to `advisor` but is targeted at internal resources.
- History and client endpoints are currently mock responses.

---

## 6. `mis_agent` — Natural language to SQL reporting agent

### Key files
- `mis_agent/app.py`
- `mis_agent/services.py`
- `mis_agent/knowledge_base.json`

### Purpose

Converts natural language business questions into SQL, executes them against a database, and returns the results.

### Main flow

1. Load environment with `mis_agent.config.load_environment()`.
2. Initialize `MISAgent` lazily on first request.
3. Expose endpoints:
   - `/health`
   - `/{api_prefix}/tables` — list available tables
   - `/{api_prefix}/query` — natural language query execution
   - `/{api_prefix}/execute-sql` — raw SQL execution
   - `/{api_prefix}/knowledge-base` — business terminology and table context

### `MISAgent` behavior

- Connects to a SQL database using SQLAlchemy.
- Loads `knowledge_base.json` for business context, table definitions, and relationships.
- Uses Gemini to generate SQL Server-compatible queries.
- Executes the SQL and returns results.
- If SQL execution fails, it asks the LLM to fix the query and retries.
- Produces both the raw SQL and a natural language answer.

---

## 7. How requests travel in the repo

1. The top-level `app.py` mounts all subapps.
2. Request paths:
   - `POST /agent-api/genai/response/` → `advisor`
   - `POST /agent-api/customer/response/` → `customer_service`
   - `POST /agent-api/Intranet/response/` → `Intranet`
   - `POST /agent-api/master-agent/login` and `/agent-api/master-agent/route/` → `master_agent`
3. `master_agent` can route a query into the advisor, customer service, or intranet flows, and then merge outputs.
4. `mis_agent` lives separately and is not mounted by the root `app.py`; it is a standalone service focused on SQL/reporting.

---

## 8. Key concepts to remember

- `master_agent` is the central dispatcher and general assistant.
- `advisor`, `customer_service`, and `Intranet` are specialist agents with domain-specific LLM and database logic.
- `mis_agent` is a SQL/reporting agent that bridges business questions to data queries.
- Several services use vector search with FAISS and retrieval-augmented generation.
- Authentication in `master_agent` is demo-only JWT.
- The repo separates frontend-facing API routes from business logic in `services.py`.

---

## 9. Suggested next file reads

- `master_agent/app.py` for routing and classification logic.
- `advisor/services.py` for insurance RAG and query handling.
- `customer_service/services.py` for customer update workflows.
- `mis_agent/services.py` for NLP-to-SQL generation.
- `Intranet/services.py` for internal knowledge retrieval.
