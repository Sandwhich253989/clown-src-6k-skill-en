# CORS (Permanent · Do Not Test)

> **User decision, 2026-08-12:** Do not test CORS going forward. See `vuln-report-format.md` for whether an issue belongs in a report.

## Hard Rules

1. **Do not test it:** CORS is no longer a required test or a required next step in the type matrix or site-entry workflow.
2. Shift effort to **injection / SSRF / XSS / RCE / authorization bypass / unauthorized access to sensitive business data / privilege escalation / logic flaws**, and similar issues.

## In One Line

**CORS = do not test.**