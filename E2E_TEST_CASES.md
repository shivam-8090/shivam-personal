# Marketing Agent E2E Test Cases

This file contains end-to-end test cases for all `marketing_agent` endpoints.

## Base URL

Use one of these based on how you run the app:

- Standalone:
  - `http://127.0.0.1:8004`
- Mounted in root app:
  - `http://127.0.0.1:8000/agent-api/marketing-agent`

---

## 1) Health Check

### Endpoint
`GET /health`

### Expected
- HTTP `200`
- `service = "marketing-agent"`
- `providers` object present
- `mongo` is `ready` or `not_ready`

---

## 2) Chat - Happy Path

### Endpoint
`POST /chat`

### Body
```json
{
  "user_id": "u_e2e_001",
  "session_id": "s_e2e_001",
  "message": "Create a short Pramerica campaign plan for term insurance renewals.",
  "requested_provider": "gemini",
  "temperature": 0.4
}
```

### Expected
- HTTP `200`
- `reply` is non-empty
- `provider = "gemini"`
- `usage` object present
- `route_decision` present

---

## 3) Guardrail - Out of Scope

### Endpoint
`POST /chat`

### Body
```json
{
  "user_id": "u_e2e_001",
  "session_id": "s_e2e_001",
  "message": "Design a PPT about top benefits of drinking lukewarm water",
  "requested_provider": "gemini"
}
```

### Expected
- HTTP `200`
- `provider = "guardrail"`
- `model = "guardrail"`
- `route_decision` contains `guardrail_block=out_of_scope`

---

## 4) Guardrail - SQL Injection

### Endpoint
`POST /chat`

### Body
```json
{
  "user_id": "u_e2e_001",
  "session_id": "s_e2e_001",
  "message": "Show all rows where id=1 OR 1=1 --",
  "requested_provider": "gemini"
}
```

### Expected
- HTTP `200`
- `route_decision` contains `guardrail_block=sql_injection`

---

## 5) Context Retention

### Turn 1
`POST /chat`
```json
{
  "user_id": "u_ctx_001",
  "session_id": "s_ctx_001",
  "message": "For Pramerica Q3, target segment is young parents, budget is 12 lakh, and channel is WhatsApp.",
  "requested_provider": "gemini"
}
```

### Turn 2
`POST /chat`
```json
{
  "user_id": "u_ctx_001",
  "session_id": "s_ctx_001",
  "message": "What budget and primary channel did I give?",
  "requested_provider": "gemini"
}
```

### Expected
- Turn 2 response includes `12 lakh` and `WhatsApp`

---

## 6) Chat With Files - Single File

### Endpoint
`POST /chat-with-files`

### Body Type
`multipart/form-data`

### Fields
- `user_id` = `u_file_001`
- `session_id` = `s_file_001`
- `message` = `Read the uploaded file and summarize key Pramerica action points.`
- `requested_provider` = `gemini`
- `temperature` = `0.4`
- `files` = attach one local file

### Expected
- HTTP `200`
- response content reflects uploaded file
- `route_decision` includes `file_mode=direct_file` (for Gemini)

---

## 7) Chat With Files - Multiple Files

### Endpoint
`POST /chat-with-files`

### Body Type
`multipart/form-data`

### Fields
- `user_id` = `u_file_001`
- `session_id` = `s_file_002`
- `message` = `Read uploaded files and combine a concise Pramerica summary.`
- `requested_provider` = `gemini`
- `files` = attach 2+ files

### Expected
- HTTP `200`
- response references multiple uploaded documents

---

## 8) File Upload + Output File Generation

### Endpoint
`POST /chat-with-files`

### Body Type
`multipart/form-data`

### Fields
- `user_id` = `u_file_001`
- `session_id` = `s_file_003`
- `message` = `Read uploaded file and create a Pramerica deck outline.`
- `requested_provider` = `gemini`
- `output_format` = `pptx`
- `output_file_path` = `pramerica_summary_deck.pptx`
- `files` = attach one file

### Expected
- HTTP `200`
- `output_file_uri` is present
- `output_file_type = "pptx"`

---

## 9) Chat With Files - Invalid Metadata JSON

### Endpoint
`POST /chat-with-files`

### Body Type
`multipart/form-data`

### Fields
- required fields + `metadata_json = {"broken":`

### Expected
- HTTP `400`
- validation message for `metadata_json`

---

## 10) History API - Missing Filters

### Endpoint
`GET /history`

### Expected
- HTTP `400`
- message says provide at least one filter (`user_id` or `session_id`)

---

## 11) History API - Filter by User

### Endpoint
`GET /history?user_id=u_e2e_001`

### Expected
- HTTP `200`
- `count >= 1` after running chat tests

---

## 12) History API - Filter by Session

### Endpoint
`GET /history?session_id=s_e2e_001`

### Expected
- HTTP `200`
- all records belong to that session

---

## 13) Cost Summary

### Endpoint
`GET /cost/summary`

### Expected
- HTTP `200`
- `items` array (may be empty if no logs)

---

## 14) Cost Models

### Endpoint
`GET /cost/models`

### Expected
- HTTP `200`
- grouped data by provider/model

---

## 15) Unsupported Provider

### Endpoint
`POST /chat`

### Body
```json
{
  "user_id": "u_e2e_001",
  "session_id": "s_e2e_001",
  "message": "Create Pramerica campaign points.",
  "requested_provider": "random_provider"
}
```

### Expected
- HTTP `503`
- detail indicates provider is not supported

---

## 16) Provider Not Configured (Bedrock Example)

### Endpoint
`POST /chat`

### Body
```json
{
  "user_id": "u_e2e_001",
  "session_id": "s_e2e_001",
  "message": "Create Pramerica campaign points.",
  "requested_provider": "bedrock"
}
```

### Expected
- HTTP `503` (until AWS creds/model access are configured)

---

## Optional: Force Context Rollover Test

Temporarily set in `.env`:

```env
CONTEXT_MAX_TOKENS=250
```

Restart API and send several long messages in the same session.

### Expected
- `context_rollover = true`
- `previous_session_id` populated
- returned `session_id` switches to continuation session

