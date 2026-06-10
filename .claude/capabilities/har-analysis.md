# Capability: HAR File Analysis

## Purpose

Parse HTTP Archive (HAR) files to identify failed requests, extract error payloads, and trace API call patterns.

---

## PII Redaction (MANDATORY — applied BEFORE any analysis)

HAR files contain highly sensitive data. Apply these redactions immediately upon parsing:

### Headers to NEVER report:
- `Authorization` (Bearer tokens, API keys)
- `Cookie` / `Set-Cookie` (session IDs)
- `X-API-Key`, `X-Auth-Token`
- `X-SFDC-Session` (Salesforce session ID)
- Any header containing "token", "secret", "key", "password", "credential"

### Request/Response body redaction:
- Email addresses → `[EMAIL_REDACTED]`
- Phone numbers → `[PHONE_REDACTED]`
- SSN patterns → `[SSN_REDACTED]`
- Credit card numbers → `[CC_REDACTED]`
- Physical addresses (multi-line street/city/state/zip) → `[ADDRESS_REDACTED]`
- Base64-encoded auth → `[AUTH_REDACTED]`
- Customer names in JSON values → `[NAME_REDACTED]`

### What IS safe to report from HAR:
- URL paths and query parameter names (NOT values containing PII)
- HTTP methods (GET/POST/PUT/DELETE)
- Status codes (200, 400, 401, 403, 404, 500)
- Error response structures (field names, error codes, error messages)
- Timing data (wait, receive, dns, connect durations)
- Content-Type headers
- Salesforce error payloads (errorCode, message, fields arrays)
- Org IDs and Record IDs (internal identifiers, not PII)

---

## Analysis Approach

1. **Parse HAR JSON** → filter entries by status code (4xx/5xx failures)
2. **For each failed request:**
   - Report: URL path, method, status code, timing
   - Extract: error response body (with PII redacted)
   - Note: request sequence (was it after auth? part of a batch?)
3. **Look for patterns:**
   - Repeated failures (same endpoint, same error)
   - Auth failures followed by functional failures (token expired?)
   - Timeout patterns (slow responses preceding failures)
   - CORS or CSP errors (misconfigured community/experience site)
4. **Correlate with Salesforce context:**
   - Map URL paths to API type (REST, Apex REST, Aura, Lightning)
   - Identify the operation (CRUD, composite, batch, custom endpoint)

---

## Common Salesforce HAR Patterns

| Status | URL Pattern | Likely Issue |
|---|---|---|
| 401 | `/services/data/vXX.X/` | Session expired, token invalid |
| 403 | `/services/data/vXX.X/sobjects/` | FLS/CRUD insufficient permissions |
| 400 | `/services/data/vXX.X/composite/` | Malformed composite request |
| 500 | `/services/apexrest/vlocity_*/` | Apex service failure (managed package) |
| 500 | `/services/apexrest/omnistudio_*/` | OmniStudio standard runtime failure |
| 500 | `/aura?r=1&` | Aura/LWC component error |
| 504 | Any | Gateway timeout — long-running operation |
| 400 | `/services/data/vXX.X/actions/custom/` | Invocable action input validation |

---

## URL → Component Mapping

| URL Pattern | Component |
|---|---|
| `/services/apexrest/vlocity_cmt/` | Managed package OmniStudio (CMT) |
| `/services/apexrest/vlocity_ins/` | Insurance managed package |
| `/services/apexrest/omnistudio/` | Standard Runtime OmniStudio |
| `/aura?r=1&aura.ApexAction` | Apex via Aura framework |
| `/webruntime/api/` | Experience Cloud / LWR |
| `/lightning/` | Lightning UI navigation |

---

## Timing Analysis

| Metric | Threshold | Meaning |
|---|---|---|
| `wait` > 30s | Critical | Server processing too slow |
| `receive` > 10s | Warning | Large response payload |
| `dns` > 1s | Warning | DNS resolution issue |
| `connect` > 2s | Warning | Network/TLS handshake slow |
| `blocked` > 5s | Warning | Browser connection limit reached |
