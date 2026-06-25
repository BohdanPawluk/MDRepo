# Incident Review — RAG API Access/Rate Limit Debugging
**Date:** June 25, 2026  
**Service:** `https://rag-api.pawlukwebstudio.com/v1/agent/rag/query`  
**Owner:** Bohdan Pawluk  
**Status:** Access redirect removed ✅, rate limiting validated ✅, endpoint publish pending ⚠️

---

## Executive Summary

Today’s incident involved two separate blockers that appeared as one:

1. **Cloudflare Access interception** was still active on the API path, causing `302` redirects to Cloudflare Access login and preventing direct API validation.
2. After Access was removed, backend returned `404 Not Found`, confirming the request now reached origin but the `/v1/agent/rag/query` route was not yet published.

Rate limiting itself was validated successfully once traffic path was clean.

---

## What went wrong (including my mistakes)

### Platform / config issues
1. Access app remained attached to API destination path.
2. Cloudflare UI/policy state appeared misleading versus actual runtime behavior.
3. Testing order initially mixed infra auth interception with app-route existence checks.

### Assistant mistakes (explicit accountability)
1. I delayed the fastest decisive step (remove Access app and immediately retest).
2. I over-trusted dashboard state instead of prioritizing traffic evidence (`302` + `www-authenticate: Cloudflare-Access`).
3. I did not frame dual-root-cause early enough (Access interception + undeployed route).
4. I omitted endpoint-code details in an earlier report revision and had to correct it.

---

## What was done to fix it

1. Reproduced issue with curl and confirmed Access behavior:
   - `HTTP/2 302`
   - redirect to `cloudflareaccess.com`
   - `www-authenticate: Cloudflare-Access`
2. Confirmed Access app targeted endpoint destination.
3. Deleted Access app (`rag-agent-endpoint`) to remove interactive auth gate.
4. Retested endpoint:
   - no redirect,
   - now `HTTP/2 404 {"detail":"Not Found"}`
5. Ran rate-limit burst test (80 requests):
   - `65 x 404`
   - `15 x 429`
6. Cooldown test after 15s returned to baseline (`404`).

---

## Evidence and interpretation

### Observed outputs
- **Before Access deletion:** redirect/login behavior (`302`)  
- **After Access deletion:** origin reachable, route missing (`404`)  
- **Burst:** rate-limit enforcement active (`429` present)  
- **Cooldown:** normal baseline resumed (`404`)

### Interpretation
- Access interception problem is solved.
- Cloudflare rate limiting rule works.
- Remaining gap is app deployment of `/v1/agent/rag/query`.

---

## Root Cause

### Primary
- Access app enforcement on API endpoint caused browser-login redirect flow for machine API requests.

### Secondary
- Endpoint route `/v1/agent/rag/query` not yet published in running backend version.

---

## Current State

### Fixed ✅
- No Access redirect for tested endpoint.
- Rate limiter triggers under burst traffic.

### Pending ⚠️
- Live backend route implementation/deployment for `/v1/agent/rag/query`.

---

## NEXT ACTIONS (updated endpoint contract)

1. **Implement and publish `POST /v1/agent/rag/query` in `main.py` with the SAME behavior as `/v1/rag/query` (no stub contract):**

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

2. **Keep response contract aligned with existing public endpoint `/v1/rag/query`:**
   - `answer`
   - `sources`
   - `timing`
   - `session_id`

3. **Deploy backend revision** so production serves this route.
4. **Smoke test** immediately after deploy:

```bash
curl -i -X POST "https://rag-api.pawlukwebstudio.com/v1/agent/rag/query" \
  -H "Content-Type: application/json" \
  -d '{"query":"healthcheck"}'
```

5. **Re-run burst + cooldown validation**
   - Burst should include some `429`.
   - Baseline/cooldown should return normal app response (not `302`, not `404`).

6. **Close incident only when all success criteria pass.**

---

## Success Criteria (Definition of Done)

- `POST /v1/agent/rag/query` returns the real app response contract (`answer`, `sources`, `timing`, `session_id`).
- No Cloudflare Access redirect (`302`) on endpoint.
- Burst test triggers rate-limit (`429`) predictably.
- Cooldown returns to baseline behavior.
- Contract for request/response is documented and stable.

---

## Lessons Learned

1. Prioritize runtime traffic evidence over dashboard appearance.
2. Treat infra/auth path validation and app route existence as separate gates.
3. For API testing, remove interactive Access layers unless explicitly using service-token auth.
4. Use a fixed test sequence:
   1) no redirect  
   2) route exists  
   3) burst triggers limiter  
   4) cooldown resets

---

## Accountability Statement

I (assistant) contributed to delay by not forcing the quickest decisive isolation step early and by initially omitting endpoint-code details in the report. That is on me.
