# External Agent API Contract — RAG Query

## Base URL
`https://rag-api.pawlukwebstudio.com`

## Endpoint
`POST /v1/agent/rag/query`

## Authentication
Use one of:
- `Authorization: Bearer <agent_token>`
- Cloudflare Access service token headers (if configured)

Requests without valid auth are rejected (`401` or `403`).

## Headers
- `Content-Type: application/json`
- `Authorization: Bearer <token>` (if using bearer auth)

## Request Body (JSON)
```json
{
  "messages": [
    { "role": "user", "content": "Do you sell exhaust system parts?" }
  ],
  "session_id": "agent-session-001",
  "agent_context": {
    "agent_id": "external-agent-01",
    "task_id": "task-optional-123",
    "origin": "external_web_agent",
    "tool_mode": "rag_only"
  }
}
```

### Field rules
- `messages` (required): array of chat messages.
- `session_id` (required or strongly recommended): caller-provided conversation/session identifier.
- `agent_context` (optional): metadata about the calling agent.
- Unknown fields may be ignored or rejected depending on server policy.

## Success Response (200)
```json
{
  "answer": "Yes, we do sell exhaust system parts...",
  "sources": ["..."],
  "timing": {
    "embedding_seconds": 0.38,
    "search_seconds": 0.01,
    "lm_studio_seconds": 5.53,
    "llm_seconds": 5.53,
    "total_seconds": 5.92
  },
  "session_id": "agent-session-001"
}
```

## Error Responses
- `400 Bad Request` — invalid JSON or missing required fields
- `401 Unauthorized` — missing/invalid auth token
- `403 Forbidden` — token valid but not allowed for this endpoint
- `429 Too Many Requests` — rate limit exceeded
- `500 Internal Server Error` — server-side failure

## cURL Example
```bash
curl -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_AGENT_TOKEN" \
  -d '{
    "messages":[{"role":"user","content":"Do you sell exhaust system parts?"}],
    "session_id":"agent-session-001",
    "agent_context":{"agent_id":"external-agent-01","origin":"external_web_agent","tool_mode":"rag_only"}
  }'
```

## Notes
- This endpoint is for **external agents**.
- Human website chat can continue using `/v1/rag/query`.
- Keep response shape stable for compatibility.
