---
name: har-analyzer
description: "Diagnoses Celonis platform issues from HAR files and browser logs — aggregates 4xx/5xx errors, traces auth failure cascades (AAM/SAML/OAuth), identifies slow requests, and produces a GDPR-scrubbed CBE-ready summary safe to paste into bug escalations."
metadata:
  source: celoskills
  author: "r.prithivi"
  version: "1"
  function: Services
  tags: "har, l2-support, debugging, gdpr, cbe, authentication, aam, task-mining, saml"
---

# HAR Analyzer

Celonis L2 support diagnostic skill. Analyzes HAR files and browser log dumps to identify errors, trace auth failures, and produce structured reports ready for Customer Bug Escalations (CBEs).

## Gotchas

- **GDPR first, always.** Strip emails, tokens, session cookies, and personal names from every URL, header, and response excerpt before producing any output. When uncertain → redact.
- **401 cascades are almost always a single root cause.** One failed `/auth/token` triggers dozens of downstream 401s — identify the first failure, not the count.
- **AAM vs application errors look similar.** A 403 on a specific endpoint is an AAM role/permission issue; a 403 on every endpoint is likely an expired token or IDP misconfiguration.
- **Task Mining agent timeouts show as 408/504** on heartbeat and extraction endpoints — not as auth errors. Don't confuse with AAM failures.
- **SAML assertion errors in HAR are often indirect.** The HAR shows a redirect loop or 400 from the SP; the real error is in the IDP logs. Flag this explicitly so the customer knows to check their IDP.
- **Slow requests cluster at specific timestamps.** A `timings.wait > 5000ms` cluster at a fixed time usually means resource contention or a heavy query — not network issues.

---

## STEP 0 — GDPR Scrubbing (apply BEFORE producing any output)

Before outputting ANY content derived from the HAR or log file, apply these substitutions to every URL, header, query param, cookie, and response body excerpt:

| PII Type | Examples | Replace with |
|---|---|---|
| Email addresses | user@company.com | `[EMAIL REDACTED]` |
| Bearer / API tokens | `Authorization: Bearer eyJ...` | `[TOKEN REDACTED]` |
| Passwords / secrets | `password=abc`, `client_secret=xyz` | `[CREDENTIAL REDACTED]` |
| Session / auth cookies | `session=abc123`, `JSESSIONID=...` | `[SESSION REDACTED]` |
| IP addresses (user-side) | `192.168.x.x`, `10.x.x.x`, public IPs in user context | `[IP REDACTED]` |
| Personal names in URLs/params | `/users/john.doe`, `name=Jane+Smith` | `[NAME REDACTED]` |
| Customer org / tenant IDs | Keep sanitised form: `[ORG: <id-format-retained>]` |
| Celonis EMS tenant hostname | Retain for debugging context; note "tenant name retained for analysis" |

**When uncertain → redact.** End every report with: *"GDPR scrubbing applied. Human review recommended before external sharing."*

---

## STEP 1 — Identify the input type

- **.har file** → JSON. Parse `log.entries[]`. Each entry contains `request`, `response`, `timings`, `startedDateTime`.
- **Raw log dump / paste** → Plain text. Scan for timestamps, HTTP status codes, error keywords, stack traces.
- **Browser console export** → Look for JS error stacks, XHR/fetch failures, CORS messages.
- **Multiple files** → Process each, then correlate by timestamp.

**If the file is very large:** prioritise (a) all 4xx/5xx entries, (b) the first and last 50 requests, (c) any request with `timings.wait > 5000 ms`. Note what was skipped.

---

## STEP 2 — Parse and categorise entries

For each HAR entry extract (after GDPR scrub):
- `startedDateTime`
- `request.method` + `request.url` (strip PII from query params)
- `response.status` + `response.statusText`
- `timings.wait` (server response time in ms)
- `response.content.text` for error detail (scrub PII)

Categorise:
- ✅ 2xx — Success
- ↩️ 3xx — Redirect
- ⚠️ 4xx — Client error
- 🔴 5xx — Server error
- ⏱️ Slow — `timings.wait > 3000 ms` regardless of status

---

## STEP 3 — Produce the four-section report

### Section 1: Top Errors

Group errors by HTTP status + URL pattern (strip dynamic IDs, e.g. `/api/v2/users/{id}` not `/api/v2/users/12345`).

```
| Status | Endpoint Pattern | Count | First Seen | Last Seen | Sample Message |
|--------|-----------------|-------|------------|-----------|----------------|
| 401    | /api/auth/token  | 12    | 14:32:01   | 14:35:44  | "Token expired" |
```

Flag explicitly:
- **5xx errors** — always P1 candidate; server-side
- **Repeated 401/403** — AAM issue; auth config, role assignment, token expiry
- **SAML / SSO / OAuth / JWT** in failing paths — IDP/auth flow
- **CORS errors** — `blocked by CORS policy` in response or console
- **HTTP 504 / `timings.wait > 10 000 ms`** — timeout / infra issue
- **Task Mining patterns** — agent heartbeat, extraction endpoint, connector timeouts

Celonis-specific patterns to recognise:

| Pattern | Likely cause | Suggested action |
|---|---|---|
| 401 cascade after a single failed `/auth/token` | Token expired or IDP misconfiguration | Check IDP logs, AAM role/token config |
| 403 on specific endpoints only | Missing permission / role assignment | Check AAM role matrix, permission groups |
| SAML assertion errors | IDP metadata mismatch | Verify IDP metadata, NameID format |
| 5xx on `/api/v2/` | Backend service issue | Check Datadog, escalate to PnE |
| CORS block on cross-origin request | CORS allowlist not configured | Update tenant CORS settings |
| Task Mining agent 408/504 | Agent unreachable / timeout | Check agent connectivity, firewall rules |
| Slow response cluster (>5s) at specific time | Resource contention / query issue | Check tenant load, query complexity |

### Section 2: Request Timeline

List the 20 most significant events (errors, slowdowns, auth events) in chronological order:

```
[HH:MM:SS.mmm]  STATUS  METHOD  /endpoint/path  (Xms)  — brief note
```

Example:
```
[14:32:01.443]  200 ✅  GET   /api/v2/health           (45ms)
[14:32:03.110]  401 🔴  POST  /api/auth/token          (312ms)  — first auth failure
[14:32:03.450]  401 🔴  GET   /api/v2/users/me         (8ms)    — cascading from auth
```

If there is a clear inflection point, call it out:
`⚡ Failure onset: ~14:32:03 UTC — all requests failing after this point`

### Section 3: Suspected Root Cause

Write a structured hypothesis — 3-6 sentences maximum.

```
**Pattern detected:** [one-line label, e.g. "Auth token expiry cascade"]

**Evidence:**
- [specific data point + timestamp from the HAR]
- [specific data point + timestamp]

**Confidence:** High / Medium / Low — [one sentence why]

**Suggested next step:** [concrete action, e.g. "Check IDP trust config in AAM at 14:32:00 UTC"]
```

### Section 4: CBE-Ready Summary

Write a professional paragraph (max 200 words) that:
- Describes the issue factually with no PII
- States what was observed (error types, frequency, time window)
- States the suspected cause
- Does not speculate beyond what the evidence supports

```
**Issue Summary (CBE-ready):**
[paragraph — max 200 words, no PII, factual, past tense]

**Scope:** [e.g., "AAM — Authentication flow / IDP integration"]
**Severity indicator:** [P1 / P2 / P3 — based on error patterns and business impact]
**Recommended owner:** [AAM team / Task Mining team / Platform team / Customer action required]
```

Severity guidance:
- P1: All users blocked, 5xx on core auth/data path, complete outage
- P2: Subset of users affected, repeated auth failures, degraded performance
- P3: Intermittent, low-frequency errors, single endpoint, no user-reported impact

---

## STEP 4 — Caveats and completeness

End the report with a brief note covering:
- What could NOT be determined from the HAR alone (server logs, IDP logs, Datadog metrics)
- Whether any entries were skipped due to file size
- Redaction confidence: "Review recommended before sharing" if the file contained many identifiers

---

## Output format

Produce all four sections in a single markdown response using `##` headers. Save the report as `HAR-analysis-[YYYY-MM-DD-HHMM].md`.

Aim for under 800 words in the report body (not counting tables). If the error inventory is large, truncate the timeline to the 20 highest-signal events.

---
*GDPR scrubbing applied. Human review recommended before external sharing.*
