# Marketing Agent

FastAPI service for a LangChain-based marketing assistant with provider routing,
LangSmith tracing, MongoDB query/cost logging, and a provisioned vector document
collection for future RAG.

For a full component-by-component explanation and end-to-end request flow, see
`ARCHITECTURE.md`.

## Run Locally

From the `Agentic-Ai` folder:

```powershell
python -m uvicorn marketing_agent.main:app --reload --port 8004
```

Main endpoints:

- `GET /v1/health`
- `POST /v1/chat`
- `POST /v1/response/` compatibility alias
- `GET /v1/cost/summary`
- `GET /v1/cost/models`

Create Mongo indexes once from a Python shell or admin script after the database
is reachable:

```powershell
python -c "from marketing_agent.dependencies import get_store; get_store().ensure_indexes()"
```

## Minimum Environment

```env
GEMINI_API_KEY=...
MONGO_CNXN_STRING=...
DB_PREFIX=BusinessEnablerAdmin
LANGCHAIN_API_KEY=...
LANGCHAIN_PROJECT=marketing-agent
ENABLE_LANGSMITH_TRACING=true
MARKETING_DEFAULT_PROVIDER=gemini
```

Optional pricing config:

```env
MARKETING_MODEL_PRICING_JSON={"gemini:gemini-2.0-flash":{"input_per_1k":0.0,"output_per_1k":0.0}}
```

Bedrock, Vertex, and OpenAI-like providers are implemented behind the same
interface but disabled by default until credentials and optional packages are
available.
