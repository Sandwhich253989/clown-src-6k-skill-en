# SRC Value-Driven Vulnerability Hunting Rules (Mandatory Lessons)

> Source: postmortem of a list-filter injection exercise.
> Purpose: cover standard vulnerability types and endpoint parameters without omissions, while pursuing only findings with real impact. Whether and how to report is governed only by `vuln-report-format.md`.
> Site-entry workflow (clarify scope, extract keys from JS, test differential surfaces, map objects with a session, chain Medium issues) follows `dig-scope-workflow` §4. This file governs what to investigate, minimum probes per type, and how to allocate effort (§1.1).

---

## 1. What to Hunt: Prioritize Real Impact

A valuable issue can reach High or Critical impact: cross-tenant/cross-user read or write, sensitive business data exposed without login, low privilege reaching platform administrator, injection returning another principal’s data, or a complete chain (cross-user write, account access, command execution). For effort allocation and what to test first on entry, follow only §1.1 below. If a type has an entry point, check it or mark it N/A; do not remove the type from the matrix.

**Meaning of “administrator” (easy to misread; remember this precisely):**
- When the user says to pay attention to “administrator,” check whether **official/platform-level administrator privileges** are reachable and whether there is a privilege-escalation chain.
- It does **not** mean “this is the tenant ADMIN, so nothing they can read counts as a vulnerability.”
- Tenant ADMIN is a **vantage point for comparison**: compare the tenant’s baseline (for example, session total = 2) against whether injection/authorization bypass returns a huge result set or another tenant’s data.

### 1.1 How to Allocate Effort (Test What Can More Easily Become High/Critical First)

Do not remove any types or add a fifth test set; **push XSS as hard as injection/SSRF/RCE**. Site-entry order follows `dig-scope` §4; this table only decides where to spend effort first.

| Test first | Details |
|------------|---------|
| Another person/principal’s data without login | Retrieve lists/details/attachments without logging in. **Once an unauthenticated request works, change IDs (regardless of field name)** (`dig-scope` §4.2.3). If a file channel opens, move into business APIs; do not stop at upload/download. |
| Authentication takeover | Session issuance / reset / account rebinding / ticket replacement; only account access or password change counts (`dig-scope` §4.2.2). |
| Change IDs (any field name) | No account: once unauthorized access works, change IDs. With an account: change IDs while carrying the session (`dig-scope` §4.2.3). Reaching another principal’s data is a successful break-through. |
| Write/logic with an account | Add `role`, change amount/status, skip approval, race coupons/inventory; check whether a tenant identity can reach **platform/official** admin privileges. |
| Full-strength four-part set | Test injection where result counts/reflections are visible; SSRF where a URL will be fetched; XSS where output is reflected or storable; follow command/template sinks toward RCE. Injection must return another principal’s data or show a stable differential. |
| JS keys | Salts, hard-coded keys, demo accounts. |

**Without an account:** once unauthorized access works, change IDs (any field name) → authentication endpoints → JS keys → the four-part set on differential surfaces (including XSS).
**With an account:** change IDs with the session (any field name) → add fields/skip steps/money → test the four-part set on those business parameters that already return data (including XSS).

**Test when there is an entry point, but do not make it the main effort:** CSRF that harms only yourself (follow if a cross-user write succeeds), upload/download with no further chain, weak passwords on a login form.
**Types already in the checklist**—GraphQL / uploads / XML / WebSocket / XXE / deserialization: finish testing them on the current site or mark N/A; they simply do not go ahead of ID changes or account takeover. Do not go hunting for these endpoints on every site, but if they are discovered, do not ignore them.

---

## 2. Do Not Stop Halfway When You Find These (How to Continue Testing)

Whether to report and what severity to assign are governed only by `vuln-report-format.md`. This section explains how to continue investigating:

- A file channel opens → immediately pivot to business APIs and executable/path/SSRF chains.
- A code-sending endpoint works → follow the authentication chain: does the response contain a code, is the code bound to an account, and does reset switch users? (`dig-scope` §4.2.2)
- CAPTCHA passed / password guesses are possible → stop testing the form shell; test empty passwords, skipped steps, reset, and rebinding.
- A DSN/config exposes a host but no credentials → keep looking for complete credentials. If credentials/AKSK exist, test as a key: use a fake-value comparison to prove production accepts it; the validation request must return an identity or list; then make a read-only request that does not affect production (if the first request already returned data, use a different kind of read). After a human account yields a session, likewise make a read-only request; severity follows the report format. A disproven key is treated as no key. Finding a string and stopping to submit a report is prohibited. Phone-number encryption/decryption keys do not count as access keys (whether to report follows the format).
- **Do not test CORS** (see `cors-vuln-report-priority.md`).
- If data is readable as part of normal duties, is public static data, or an endpoint returns HTTP 200 with JSON → pivot to cross-user access/injection/privilege escalation.
- If a response reveals an ID / download URL / token / internal host → add it to the current site’s queue and test it (`dig-scope` §4.1.3).
- **If a Medium issue is reportable, extend the chain before changing sites:** keep following the same object if it has not yet reached write access / cross-user impact / takeover / execution (`dig-scope` §4.3).

## 2.2 Permanent Prohibitions on Hunting Preferences

- **Do not** test CORS.
- **Do not** spend multiple days focused only on unauthorized reads or structurally identical lists without testing **injection / SSRF / XSS / RCE** (details in `dig-scope-workflow` §4.2.1).
- **You must** follow the type matrix in §3: unauthorized access, authorization bypass/IDOR (as mandatory as the four-part set when a session exists; see `dig-scope` §4.2.3), injection, SSRF, RCE chains, credential disclosure, weak passwords (bare default access on middleware; **weak passwords on login forms are not mandatory**, see `dig-scope` §4.1.1; **treat demo accounts in JS/pages as keys**), sensitive paths, XSS, uploads, traversal, CSRF, privilege escalation, logic flaws, **authentication/arbitrary login/takeover** (see `dig-scope` §4.2.2)—**test all types** (the union with `dig-scope-workflow` §4.2; **excluding CORS**).
- After obtaining a batch of unauthorized reads, **immediately** run the full §3 injection parameter table + SSRF URL parameters + XSS reflection/storage + RCE sinks against the same endpoints (see `dig-scope` §4.2.1, using differential surfaces). Do not finish unauthorized-read testing and move sites with zero payloads from the four-part set.
- With a session, do not just spray `'`: map objects / change IDs / add fields / skip steps as in `dig-scope` §4.2.3.

---

## 3. Cover Every Type: Type Matrix (At Least Once Per Target in Each Round)

For the current target’s endpoint list, check each type; **do not test only information disclosure**.
Site-entry order follows `dig-scope-workflow` §4 (clarify scope first, then extract keys, build the list, and test; switch sites when this round is complete).

| Type | Probe requirements (if an entry point exists, finish the probes; if none, mark N/A with a reason) |
|------|---------------------------------------------------------------|
| Unauthorized access | Use `credentials:omit` on key read/write endpoints. If one works, then change IDs (see below). |
| Authorization bypass / IDOR | **If an entry point exists, finish testing it.** No account: once unauthorized access works, change IDs. With an account: baseline → adjacent ID / ID copied from a response / `0` / `-1` / empty → attachment export; see `dig-scope` §4.2.3. Do not stop work over a second account. |
| Injection | For endpoints where result count/content changes or a URL is fetched, test **every** query/filter/sort/keyword parameter (baseline → probe → differential; details in §4). If the only response is “please log in” and there are no business fields → mark the whole section N/A; do not spray `'` at each parameter. Empty lists/errors/timeouts are not “please log in”; compare as described in `dig-scope` §4.1.2; stop when there is no differential. **Choose probes by stack:** operators for JSON/Mongo; ES or list OR+total for search boxes; HQL/SpEL for Java; SSTI if templates are present; for SQL surfaces, continue with `'`/boolean/delay/WAF encodings. If blocked, move the payload to another location (query/json/header/path). Username/password fields are not business parameters; test `tenant` / `corpId` / `moduleId` on a login endpoint if present (`dig-scope` §4.1.1). |
| SSRF | All URL/callback/webhook/file/fetch/import/preview/proxy parameters: internal network / metadata / out-of-band. |
| RCE / code-execution chains | Follow command/expression/template/deserialization/dangerous-upload sinks through to execution or disprove them. |
| Credential/key disclosure | After copying a token/AKSK/private key/connection string: compare with a fake value; validation request must return an identity or list; then make a read-only request that does not affect production. If a human account logs in and produces a session, severity follows the report format. Placeholder/expired values, identical responses for fake/real values, no reachable validation endpoint, phone-number encryption keys → N/A (do not report; see format). |
| Weak/default passwords | Middleware/component default access should be obvious; test through to a real session. Weak passwords on a login form are not mandatory (`dig-scope` §4.1.1). **Use hard-coded demo accounts or test tenants found in JS/pages as keys.** |
| Sensitive paths | Swagger/Actuator/debug/heapdump/env, etc.; if a key is present, continue investigating. |
| XSS | Reflected (put tags/events in reflected parameters) + stored (write, then read back); consider whether it is readable without login. |
| Upload | Type bypass, paths, executable/SSRF chain; do not stop at upload/download. |
| Traversal | `fileKey`/`path`/`filename`. |
| CSRF/authentication parameters | Missing/invalid `ctoken` + Cookie attributes; did the write succeed? |
| Privilege escalation / logic | **If an entry point exists, finish the probes at injection-level depth.** Add `role` / `isAdmin` / `amount` / `status` / `vip` to write endpoints (only change values that can be restored); skip workflow steps; reduce/change money/coupons/inventory, set negative/fractional values; if a claim/approval endpoint appears, make a concurrent request. If no such endpoint exists → N/A. With a session, see `dig-scope` §4.2.3. |
| Authentication / arbitrary login / takeover | If the checklist has session issuance / reset / rebinding / ticket replacement / 2FA, check all of it under `dig-scope` §4.2.2. Username/password fields are not part of the four-part set. |
| CORS | **N/A · Do not test.** |
| Other live surfaces | Test live surfaces omitted from the matrix too; authentication endpoints follow the row above. |

---

## 4. Endpoints and Parameters: Do Not Miss Fields

1. Open the real page → capture the **complete** request (query + body).
2. List every filter/ID field; **test short-enum fields too** (such as `vote`, `answerType`, `status`).
3. On endpoints with differential surfaces, apply this minimum matrix to every filter parameter (endpoints that only say “please log in” are excluded; username/password fields are not business parameters; see §3 and `dig-scope` §4.1.1):
   - Normal value / empty / invalid value
   - Choose probes by stack: SQL surface `'` / boolean; JSON surface operators; search box ES/OR+total; Java HQL/SpEL; SSTI if a template exists
   - Delay (based on database type)
   - WAF encodings: `%0a`, `%0d`, duplicate parameters, comments, mixed case; **also change the location** (query / json / header / path)
4. **A differential is hard evidence:** compare the current tenant’s baseline total/count with the total or other-principal fields after the payload.

**Lesson learned (list filter):**
- `vote`: the user reported an injection; it currently returns many 405s and may have been fixed, but still needs regression testing.
- `keyword=1%0aor%0a1`: bypassed the WAF, total changed from 2 → tens of millions, and cross-tenant metadata appeared → **a real SRC finding**.
- Why it was missed: only checked “is there data?” and did not run the “all-parameter injection matrix.”

---

## 5. Execution Rhythm

1. **Complete the asset set:** domains, logged-in state, menu routes, APIs extracted from JS (including salts/encrypted IDs/hidden routes/demo accounts), and parameter tables from real requests; add IDs/URLs from responses to the list.
2. **Prioritize by value:** under §1.1, first test other principals’ data without login / authentication takeover / ID changes / the four-part set (including XSS); information disclosure comes later and gets downgraded in how it is described.
3. **Cover all types and all parameters** (prioritize differential surfaces); test new endpoints as soon as they enter the queue; with a session, map objects before spraying parameters; CSRF/upload-download/login-form weak passwords are not the main effort.
4. **Keep only reviewable evidence** (requests, differential tables, key fields); do not pad the record with meaningless “findings.”
5. Save confirmed issues under `vuln-report-format`; extend a Medium issue on the same object before switching sites; dig notes do not replace the report body.

---

## 6. Pre-Conclusion Checklist

- [ ] On endpoints with differential surfaces, was every filter parameter tested for injection/authorization differences? Did you mistakenly spray parameters on endpoints that only say “please log in”?
- [ ] Was unauthenticated access tested with `omit` (for business endpoints)?
- [ ] Were cross-tenant/other-user cases tested (not just what the current tenant’s ADMIN can read)? For a single known ID, did you use an ID from a response/list rather than give up?
- [ ] With a session, did you test the object graph / change IDs / add fields / skip steps (`dig-scope` §4.2.3)?
- [ ] Did upload testing follow through to executable/SSRF/business authorization impact instead of stopping at upload/download?
- [ ] After code-sending/CAPTCHA, was the authentication chain tested under `dig-scope` §4.2.2?
- [ ] Did you mistakenly decide that “administrator” means “not a vulnerability if allowed by permissions”?
- [ ] After a WAF 405, did you try another encoding/location? Did you choose injection probes based on the stack?
- [ ] **Did you accidentally test CORS?** (Do not.)
- [ ] **Did this round focus only on unauthorized reads?** Did you mark the four-part set complete just by sending `'` to every path?
- [ ] **Injection:** On differential surfaces, did you baseline + probe + compare every query/filter parameter?
- [ ] **SSRF / XSS / RCE:** Were URL parameters, reflection/storage points, and exec sinks actually tested or marked N/A with a reason? Have any gone untouched for days?
- [ ] After a Medium issue, did you extend the chain on the same object first? Were response clues added to the list?
- [ ] For copied credentials, did you compare a fake value, obtain an identity/list from validation, and make a safe read-only request? Did you wrongly report phone-number encryption keys as vulnerabilities? After logging into a human account, did you make a read-only request?

---

## 7. In One Line

**Test every type and every parameter; under §1.1, first spend effort on issues more likely to become High/Critical (other principals’ data without login / takeover / ID changes / four-part set including XSS); run the four-part set on differential surfaces with stack-appropriate probes; with a session, authorization/logic testing is as mandatory as the four-part set; check authentication endpoints under `dig-scope` §4.2.2; do not test CORS; use baseline differentials to prove cross-principal access/injection. Extend Medium issues before switching sites. Whether to report is governed only by `vuln-report-format`.**