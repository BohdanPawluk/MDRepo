# External Agent RAG Query API Specification

## Base URL
- Production: `https://rag-api.pawlukwebstudio.com`

## Endpoint
- Method: `POST`
- Path: `/v1/agent/rag/query`
- Full URL: `https://rag-api.pawlukwebstudio.com/v1/agent/rag/query`

## Authentication and Headers

### Authentication modes
1. No `Authorization` header (allowed when edge policy permits)
2. `Authorization: Bearer <token>` (supported for agent-to-agent integrations)

### Required headers
- `Content-Type: application/json`

### Optional headers
- `Authorization: Bearer <token>`
- `Accept: application/json`

Cloudflare remains the source of truth for edge rate limiting and HTTP traffic policy for this endpoint.

## Request Schema

### Content type
- `application/json`

### JSON schema (practical contract)
```json
{
  "messages": [
    {
      "role": "user",
      "content": "string"
    }
  ],
  "session_id": "string or null"
}
```

### Field rules
- `messages`:
  - Type: array of objects
  - Required: yes
  - Minimum: at least 1 message
  - Each item must contain:
    - `role` (string)
    - `content` (string)
- `session_id`:
  - Type: string or null
  - Required: no
  - Recommended: provide a stable agent/session identifier for traceability

### Notes
- Unknown fields may be ignored or rejected depending on request model configuration.
- An empty or missing `messages` array is treated as invalid request input.

## Success Response Schema

### HTTP status
- `200 OK`

### Response body
```json
{
  "answer": "string",
  "sources": ["string"],
  "timing": {
    "embedding_seconds": 0.0,
    "search_seconds": 0.0,
    "lm_studio_seconds": 0.0,
    "llm_seconds": 0.0,
    "total_seconds": 0.0
  },
  "session_id": "string or null"
}
```

### Contract keys (stable)
- `answer`: model-generated answer text
- `sources`: retrieved document snippets used for grounding (array, may be empty)
- `timing`: execution timing object (may be empty)
- `session_id`: echoes inbound `session_id` value

## Error Responses

| HTTP status | Error type | When it occurs | Example shape |
|---|---|---|---|
| 422 | Validation error | Request body fails FastAPI/Pydantic schema validation (observed in smoke test) | `{ "detail": [ { "loc": ["body", "messages"], "msg": "field required", "type": "value_error.missing" } ] }` |
| 400 | Invalid request | Business-level validation failure in endpoint logic (if implemented) | `{ "detail": { "error": "invalid_request", "message": "..." } }` |
| 401/403 | Unauthorized/Forbidden | Authentication or policy rejection when enabled by edge or upstream controls | `{ "detail": { "error": "forbidden", "message": "..." } }` |
| 429 | Rate limit exceeded | Burst traffic exceeds edge/app throttling limits | `{ "detail": "Too Many Requests" }` or policy-defined equivalent |
| 500/502/503/504 | Upstream/runtime failure | LLM provider unavailable, timeout, or internal processing error | `{ "detail": { "error": "...", "message": "..." } }` |

## cURL Examples

### 1) Basic request
```bash
curl -i -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Do you sell exhaust system parts?"}],
    "session_id": "agent-session-001"
  }'
```

### 2) Bearer auth variant
```bash
curl -i -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_AGENT_TOKEN" \
  -d '{
    "messages": [{"role": "user", "content": "healthcheck"}],
    "session_id": "agent-session-auth-001"
  }'
```

### 3) Invalid payload example
```bash
curl -i -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -d '{"bad_field":"bad_value"}'
```

## Compatibility with /v1/rag/query
- `/v1/agent/rag/query` is contract-compatible with `/v1/rag/query` for response keys: `answer`, `sources`, `timing`, `session_id`.
- Both endpoints are expected to use the same RAG execution path and grounding behavior.
- External agent consumers can switch from `/v1/rag/query` to `/v1/agent/rag/query` without response-shape changes.
