# Implementation Plan — External Agent RAG Endpoint (Phased + Manual Runtime Control)

**Project path:** `/Applications/MAMP/htdocs/rag_app-beta-v1.4.0`  
**Primary file:** `main.py`  
**Date:** June 26, 2026  
**Owner/Operator:** User (manual runtime control)  
**Implementer:** Copilot in VS Code (code changes only)

---

## 1) Objective

Implement and publish the external agent endpoint:

- **`POST /v1/agent/rag/query`**

with behavior aligned to existing `/v1/rag/query` response semantics, while enforcing a strict phased process with stop/confirm checkpoints and manual listener control by the user.

---

## 2) Critical Constraints (Must Follow)

1. **Backup first**  
   Before any code change to `main.py`, create a timestamped backup copy in a backups directory.

2. **Copilot must NOT run FastAPI listener**  
   Copilot cannot start/stop/restart runtime jobs.  
   The user manually controls listener lifecycle in UI/terminal.

3. **Phased implementation required**  
   Copilot must stop at the end of each phase and request approval before continuing.

4. **Runtime checkpoints are manual**  
   At test checkpoints, Copilot must explicitly prompt user to:
   - “Please START FastAPI listener now and reply `READY`.”
   - “Please STOP FastAPI listener now and reply `STOPPED`.”

5. **Minimal-risk changes**  
   Avoid broad refactors. Preserve existing endpoint behavior.

6. **Contract alignment**  
   New endpoint response must include:
   - `answer`
   - `sources`
   - `timing`
   - `session_id`

---

## 3) Technical Target

## Endpoint
- `POST /v1/agent/rag/query`

## Expected implementation approach
- Reuse existing request model (`ChatRequest`) if already present.
- Reuse existing RAG execution path used by `/v1/rag/query` (e.g., `rag_endpoint(chat_request)`), not a duplicated second logic path.
- Map output into stable response shape:
  - `answer`: string
  - `sources`: array (default empty array if unavailable)
  - `timing`: object (default empty object if unavailable)
  - `session_id`: from incoming request session identifier

## Reference implementation shape
```python
@app.post("/v1/agent/rag/query")
def agent_rag_query(chat_request: ChatRequest):
    result = rag_endpoint(chat_request)
    return {
        "answer": result["response"],
        "sources": result.get("retrieved_documents", []),
        "timing": result.get("timing", {}),
        "session_id": chat_request.session_id,
    }
```

> Note: Exact keys from `result` may vary by current codebase; Copilot must adapt to existing internal return structure while preserving public response contract.

---

## 4) Phase Plan (Mandatory Stop/Confirm Gates)

## Phase 0 — Discovery & Safety Prep
**Tasks**
- Inspect `main.py` and identify:
  - `ChatRequest` model definition
  - Existing `/v1/rag/query` handler
  - Core function used to execute RAG pipeline
- Propose exact backup command(s)
- Present findings and intended minimal edit scope

**Gate**
- STOP and ask: **“Proceed to Phase 1 backup? (yes/no)”**

---

## Phase 1 — Backup (No Code Changes Yet)
**Tasks**
- Ensure backup directory exists
- Create timestamped backup of `main.py`
- Verify backup file exists and is readable

**Suggested commands**
```bash
mkdir -p /Applications/MAMP/htdocs/rag_app-beta-v1.4.0/backups
cp /Applications/MAMP/htdocs/rag_app-beta-v1.4.0/main.py \
   /Applications/MAMP/htdocs/rag_app-beta-v1.4.0/backups/main.py.bak.$(date +%Y%m%d-%H%M%S)
ls -lah /Applications/MAMP/htdocs/rag_app-beta-v1.4.0/backups/main.py.bak.*
```

**Gate**
- STOP and ask: **“Backup complete. Proceed to Phase 2 implementation? (yes/no)”**

---

## Phase 2 — Endpoint Implementation
**Tasks**
- Add `POST /v1/agent/rag/query` to `main.py`
- Reuse existing RAG flow
- Ensure response contract keys:
  - `answer`
  - `sources`
  - `timing`
  - `session_id`
- Keep existing routes unchanged

**Output required from Copilot**
- Exact patch/diff of changes
- Short rationale on why change is low risk

**Gate**
- STOP and prompt user:
  - **“Please START FastAPI listener now and reply `READY`.”**
- After user says READY: ask to continue to smoke tests.

---

## Phase 3 — Smoke Test (User-run only)
**Tasks**
- Provide user with curl tests (Copilot does not run them)
- User executes and pastes outputs
- Copilot interprets results

**Minimum tests**
1. Basic POST to `/v1/agent/rag/query`
2. Auth header variant (if bearer/token path enabled)
3. Invalid payload test (expect `400`)

**Gate**
- STOP and ask: **“Smoke tests reviewed. Proceed to Phase 4 rate-limit validation? (yes/no)”**

---

## Phase 4 — Rate-limit + Cooldown Validation (User-run only)
**Tasks**
- Provide burst test command sequence
- Confirm some `429` responses during burst
- Provide cooldown retest
- Confirm baseline behavior after cooldown

**Expected interpretation**
- No Cloudflare Access redirect (`302`) for machine API path
- Endpoint route exists (not `404`)
- Rate limiter enforces under burst (`429` present)
- Cooldown returns normal baseline app response

**Gate**
- STOP and ask: **“Validation complete. Proceed to Phase 5 documentation publish? (yes/no)”**

---

## Phase 5 — Publish API Spec Document
**Tasks**
- Create/update:
  - `docs/External Agent API Contract — RAG Query.md`
- Ensure publish-ready quality:
  - Base URL
  - Endpoint
  - Authentication modes
  - Required headers
  - Request JSON schema + field rules
  - Success response structure
  - Error response table
  - Curl examples
  - Compatibility notes with `/v1/rag/query`

**Gate**
- STOP and ask: **“Documentation updated. Final review/approve? (yes/no)”**

---

## 5) Success Criteria (Definition of Done)

1. `POST /v1/agent/rag/query` implemented and reachable.
2. Response contract stable and includes:
   - `answer`
   - `sources`
   - `timing`
   - `session_id`
3. No `302` Access redirect on API request path.
4. No `404` for deployed route.
5. Burst test yields predictable `429`.
6. Cooldown returns baseline behavior.
7. Publish-ready API contract document committed/available.
8. `main.py` backup exists with timestamp created before modifications.

---

## 6) Copilot Operating Instructions (Strict)

Copilot must follow these rules in this task:

1. Never start/stop/restart FastAPI listener.
2. Never skip backup phase.
3. Never auto-continue phases without explicit user confirmation.
4. Always provide explicit “next action” prompt at phase end.
5. Prefer minimal edits over refactors.
6. If blocked by runtime state, request manual user action and wait.

---

## 7) Manual Test Commands (Reference for User)

## Basic request
```bash
curl -i -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -d '{
    "messages":[{"role":"user","content":"Do you sell exhaust system parts?"}],
    "session_id":"agent-session-001",
    "agent_context":{"agent_id":"external-agent-01","origin":"external_web_agent","tool_mode":"rag_only"}
  }'
```

## Bearer auth variant
```bash
curl -i -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_AGENT_TOKEN" \
  -d '{
    "messages":[{"role":"user","content":"healthcheck"}],
    "session_id":"agent-session-auth-001"
  }'
```

## Invalid payload (expect 400)
```bash
curl -i -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -d '{"bad_field":"bad_value"}'
```

---

## 8) Handoff Prompt (Paste into Copilot)

```text
Read and follow docs/IMPLEMENTATION_PLAN_AGENT_RAG_QUERY.md exactly.
Execute phase-by-phase, stopping at each gate for my approval.
Do not run FastAPI listener; I will run it manually.
Start with Phase 0 discovery and show findings before any edits.
```

---

## 9) Change Log

- v1.0 (2026-06-26): Initial phased implementation plan with mandatory backup, manual runtime control, stop/confirm gates, and publish-spec deliverable.
