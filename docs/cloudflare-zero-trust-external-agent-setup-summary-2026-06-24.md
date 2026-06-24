# Cloudflare Zero Trust Setup Summary (External AI Agent Access)

**Date:** June 24, 2026  
**Owner:** Pawlukwebstudio  
**Goal:** Protect a RAG API endpoint with Cloudflare Zero Trust and allow access for external AI agents using a service token.

---

## ✅ What was completed today

### 1) Zero Trust account and Access application setup
- Confirmed Zero Trust account was active and accessible.
- Navigated to:
  - **Access controls → Applications**
- Started creating a new self-hosted/private application.

### 2) Destination/hostname configuration
Configured application destination for the intended API endpoint:

- **Subdomain:** `rag-api`
- **Domain:** `pawlukwebstudio.com`
- **Path:** `/v1/agent/rag/query`

Resulting protected URL:

- `https://rag-api.pawlukwebstudio.com/v1/agent/rag/query`

### 3) Service token authentication policy
Created and attached an Access policy for machine-to-machine authentication:

- **Policy name:** `allow-external-agent-service-token`
- **Action:** `Service Auth`
- **Include rule:** `Service Token = external-rag-agent`

This is the correct policy type for external agent/API access via headers.

### 4) Service token creation
Created a service token credential:

- **Token name:** `external-rag-agent`
- Captured:
  - **Client ID**
  - **Client Secret**

### 5) End-to-end auth test from terminal
Executed `curl` against the protected public URL using:

- `CF-Access-Client-Id`
- `CF-Access-Client-Secret`

Response was:

- `{"detail":"Not Found"}`

Interpretation:
- ✅ Cloudflare Access authentication is functioning.
- ❌ Backend route at that path is not yet implemented (or path mismatch).

### 6) Route alignment update (critical)
After verifying backend behavior, the Cloudflare Access path should match the existing backend route.

- **Existing backend route to use:** `/v1/rag/query`
- **Do not use (for now):** `/v1/agent/rag/query`

Updated public endpoint to protect/call:

- `https://rag-api.pawlukwebstudio.com/v1/rag/query`

---

## 📌 Current status

### Fully done
- Cloudflare Zero Trust external protection is configured.
- Service Auth policy is in place and linked to a Service Token.
- Endpoint is reachable through Cloudflare with token-based auth.

### Still required
- Ensure Cloudflare application path is updated to `/v1/rag/query`.
- Confirm backend route `POST /v1/rag/query` is active and returns expected response.
- Ensure tunnel forwards this hostname/path to the correct local service/port.

---

## 🔐 Required headers for external AI agents

External clients must send these headers on each request:

- `CF-Access-Client-Id: <CLIENT_ID>`
- `CF-Access-Client-Secret: <CLIENT_SECRET>`

---

## 🌐 Endpoint to share with external AI agents

- **Method:** `POST`
- **URL:** `https://rag-api.pawlukwebstudio.com/v1/rag/query`
- **Auth:** Cloudflare Service Token headers (above)
- **Body:** JSON (as defined by your backend API contract)

---

## 🧪 Test command template

```bash
curl -X POST "https://rag-api.pawlukwebstudio.com/v1/rag/query" \
  -H "Content-Type: application/json" \
  -H "CF-Access-Client-Id: <CLIENT_ID>" \
  -H "CF-Access-Client-Secret: <CLIENT_SECRET>" \
  -d '{"messages":[{"role":"user","content":"test"}],"session_id":"agent-test-001"}'
```

---

## Next session recommended tasks

1. Verify Cloudflare application destination path is `/v1/rag/query`.
2. Validate locally first (`localhost`) and then through Cloudflare URL.
3. Publish a one-page integration spec:
   - Request schema
   - Response schema
   - Error codes
   - Rate limits / timeout behavior
4. Store token securely in a secret manager and define rotation schedule.

---

## Outcome statement

Today’s objective of setting up **Cloudflare Zero Trust external access control** was completed successfully.  
The remaining work is final route alignment/verification so external AI agents can receive successful API responses.
