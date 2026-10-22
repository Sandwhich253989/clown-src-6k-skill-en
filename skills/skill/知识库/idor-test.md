> Whether to write a report is determined only by `~/.grok/rules/vuln-report-format.md`. When logged in, the minimum probes are in `dig-scope` §4.2.3 (object graph, changing IDs, sentinel values); use another user's ID from a list/response for a single account, rather than grinding through registration for a second account. By default, prove cross-user/cross-tenant access with read/list differences. Still test write authorization, but **do not modify/delete someone else's existing data**. See “How to test write authorization” below for the order. No bulk edits/deletes and no real financial loss.
> Search headings for short-table pointers. The English BOLA encyclopedia has been cut; “How to test write authorization” remains above.

## I. Existing knowledge base

# IDOR / Authorization Bypass Testing Guide

> **Minimize harm:** Default to read/list differences. Test write authorization, following “add your own first → then delete what you added.” Do not interpret “read-only” as “don't test/report write IDOR.”

### How to test write authorization (add first, then clean up)

If read differences close the loop, don't write. If that isn't enough to prove “can change someone else's thing”:

1. **Test adding first.** Use your own account (or unauthenticated) to call the creation endpoint and see if you can attach an item under another person/store/tenant. Success proves write authorization bypass.
2. **Then delete the same item you just added.** Use the same endpoint or corresponding delete endpoint; leave no dirty data.
3. **Do not** edit another person's existing order, change their password/role, or delete their original address/coupon/product. Those cannot be cleanly reverted and are not the least harmful proof.
4. If there is no creation endpoint and only existing objects can be changed/deleted: change one test field **you can restore**, then stop after one request. For password/role/rebinding, follow `dig-scope` §4.2.2 and `authbypass-test.md` (test whether removing the old value works); revert immediately if successful. If you cannot revert it, stop at the response; do not leave someone else's password, role, or email changed.
5. To prove financial impact: touch only a test amount/state **you can restore**, and revert it immediately. Do not transfer production funds, clear another person's inventory, or change their existing order so it cannot be recovered. **This is not** a reason to skip testing payment/write IDOR. No bulk actions and no delete tests against production master data. This is harm minimization, **not** “don't test write IDOR / takeover / added fields.”

## Distinguish the concepts

- **Horizontal authorization bypass**: Access another user's resource at the same privilege level (A accesses B's order)
- **Vertical authorization bypass**: A low-privilege account calls a high-privilege endpoint (ordinary user calls admin API)
- **Unauthenticated access**: Directly access an endpoint requiring authentication without logging in

---

## Testing flow

### Step 1: Find user-identifier parameters

Look for user/resource identifiers in:

```
URL path:      /api/user/12345/info
URL parameter: ?userId=12345&orderId=ABC
Request body:  {"uid": 12345, "target_id": "user_abc"}
Response body: {"id": 12345, "created_by": 67890}  ← collect these IDs
```

**High-value endpoint traits**:
- Paths containing `user`, `account`, `profile`, `order`, `bill`, `message`
- Responses containing other people's phone, email, real name, national ID
- Modification endpoints: `update`, `edit`, `delete`, `change`

### Step 2: Object IDs (two accounts help; don't grind registration if you have one)

```
With a control account: log in as account A, get its token, and use account B's resource ID for comparison.
Single account / no second account: use IDs from lists/responses, neighboring IDs, `0`/`-1`/empty as someone else's ID (`dig-scope` §4.2.3). Do not grind registration just to get a second account.
```

### Step 3: Test by replacing identifiers

```python
# Horizontal authorization test example
# Use A's token to access B's resource

import requests

token_a = "Bearer eyJ..."
victim_user_id = "B's user ID"

# Original request (access your own resource)
r1 = requests.get(
    "https://target.com/api/user/info",
    params={"userId": "A's ID"},
    headers={"Authorization": token_a}
)

# Authorization-bypass request (access B)
r2 = requests.get(
    "https://target.com/api/user/info",
    params={"userId": victim_user_id},
    headers={"Authorization": token_a}
)

# Compare responses
print("Self:", r1.json())
print("Other:", r2.json())
# If r2 successfully returns B's data → horizontal authorization bypass
```

### Step 4: Test vertical authorization bypass

```bash
# Call an admin endpoint using an ordinary user's token
# First capture the admin endpoint while logged in as an administrator
ADMIN_ENDPOINT="https://target.com/api/admin/users/list"
USER_TOKEN="ordinary user's token"

curl -X GET "$ADMIN_ENDPOINT" \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json"

# 200 success → vertical authorization bypass
# 403 Forbidden → access control is in place
```

### Step 5: Test unauthenticated access

```bash
# Remove Authorization and request directly
curl -X GET "https://target.com/api/user/info?userId=12345"

# Or replace it with an invalid token
curl -X GET "https://target.com/api/user/info?userId=12345" \
  -H "Authorization: Bearer invalid_token_123"
```

---

## Common bypass techniques

### Guessing IDs
```
Numeric IDs: try ±1, ±10, 0, -1, 999999
Tenant/application fields in lists: if another real ID is blocked, also try 0 / -1 / empty / omit (see “Sentinel tenant” below)
UUID: may obtain another user's UUID from response or JS
Phone number: some endpoints use phone numbers directly as identifiers
```

### Parameter pollution
```
# Submit the same parameter more than once
POST /api/user/info?userId=A_ID&userId=B_ID
POST body: userId=A_ID   + URL: ?userId=B_ID
```

### Path traversal
```
/api/user/A_ID/orders  →  /api/user/B_ID/orders
/api/order/123         →  /api/order/124, 125...
```

### Encoding bypasses
```
userId=12345         → userId=0x3039 (hexadecimal)
userId=12345         → userId=%31%32%33%34%35 (URL-encoded)
```

---

## Collecting PoC evidence

```
Must save:
1. Complete request (including token and headers)
2. Complete response (including victim data)
3. Comparison screenshots (A's normal response vs. the bypass response)
4. Proof linking the victim account ID to the returned data
```

---

### Encrypted IDs (short table has a pointer)

Recognize: Request IDs for editing/viewing addresses, orders, or profiles are long ciphertext strings; plaintext small integers appear in responses or lists; JS has `RSAUtils` / `JSEncrypt` / `security.js`, or hardcoded `modulus` + `exponent`.

Test: Ciphertext is not a wall; anyone can use a public key.

1. First call your own item and note the plaintext ID in the response.
2. Use the same frontend encryption to encrypt neighboring numbers (plaintext ±1, then expand outward).
3. Change only this ciphertext ID; keep the session unchanged.

Count as success: After changing it, response reveals **someone else's** name/phone/address. Proving only that your ciphertext decrypts to your own plaintext does not count.

False positives: Server filters by session and returns only your own data even when encryption is correct; key is for signature validation, and arbitrary ciphertext is simply rejected. A miss on one site does not remove this row from the short table. Do not store real key values or a site's `security.js` URL in the repository; copy it from the current site's JS during testing.

### Sentinel tenant (short table has a pointer)

Recognize: Ticket / qualification / session list carries `appId`, `tenantId`, `orgId`; response has attachment URL (`sign` + `file_name` + another tenant field).

Test (changing to a real other ID is already horizontal access; this is an additional probe):

1. Try tenant field `0`, `-1`, empty string, and omit it — backend often treats sentinel values as “no filter.”
2. Count as a bypass only if `total` / row count jumps well above your tenant's baseline; increasing `pageSize` or turning `page` only retrieves more rows, **it is not a vulnerability**.
3. Try type fields (`serviceType`, etc.) with `0` / `-1` too; they may switch to another business-order set.
4. For signed downloads in the list: opening unchanged often returns 403; **leave `sign` / `file_name` unchanged and change only the tenant in the URL to yours** (signature may not cover this field). See the pre-signed section in `file-upload-test.md`.

Count as success: List exposes another tenant's ticket body / customer-service conversation, or attachment downloads the other party's certificate.

False positives: `0` still returns only your own data; changing tenant for download returns 403; only a file channel exists, with no business list. A miss on sentinel values does not remove this row; try list filtering on the next site too.

### Registry catalog does not filter tenants (short table has a pointer)

Recognize: Container / cloud function / mini-program cloud is present; you can `docker login` or see Registry / Harbor / `/v2/`.

Test: This is list authorization bypass, not unauthenticated file download.

1. Log in to the registry with **your own account**.
2. `GET /v2/_catalog` (Harbor may also have a project-list endpoint; same probe).
3. If returned repository names match **another tenant**, then call `/v2/<repository>/tags/list` and `docker pull` one image.

Count as success: Catalog contains another tenant's private repository and the image can be pulled (layers contain their app/config). Proving only that you can log in or pull an official public image does not count.

False positives: Catalog returns only your repositories; names are listed but pull returns 403; registry was public already. A miss on one site does not remove this row from the short table.

Separate from anonymous fileId download and an OSS bucket policy open to everyone: this finding is **logged-in catalog exposes every private repository on the site**.

### Twin login prefixes (short table has a pointer)

Recognize: Business H5 puts login RPCs under `/fapi/d/` (or similar authenticated prefix); same gateway also has `/fapi/n/` (or n / unlogin / guest) unauthenticated prefix. Testing only the d endpoint in JS makes the whole area look login-protected.

Test (while logged out):

1. Control: d endpoint should require login / ticket.
2. Change `d` in the path to `n`, and call the same RPC with the same object ID.
3. Try a neighboring ID; follow certificate-image URLs in the response.

Count as success: While logged out, retrieve someone else's ID card / certificate / phone number.

False positives: n still requires login; n only exposes public config; only your own newly submitted supplementary file. A miss on one site does not remove this row from the short table.

### Neighboring numeric RPC commands (short table has a pointer)

Recognize: Business page calls a numeric RPC gateway (such as `/data/{number}/forward`). The page often hardcodes only one cmd (invoice/ads, for example). Neighboring numbers may expose another internal list/write endpoint; do not stop at the hardcoded one. `numeric RPC gateway` is a common skin name; test even if it is absent.

Test (while logged out):

1. Copy the forward URL from JS and change the number in the path to a neighboring value.
2. First send empty `{"req":{}}` to see if it lists data; control against the hardcoded cmd, which should be another business function.
3. If the list returns internal records, call the neighboring write endpoint with an ID (first use a fake/nonexistent ID). Revert immediately if possible.

Count as success: Internal publication/operator details, or a field can be changed.

False positives: Neighboring number is the same public endpoint; only a public software catalog. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Identity-domain account CRUD (short table has a pointer)

Recognize: Account CRUD in the same product's business frontend hits a login gate (`AuthFailure.NoLogin`); a separate `*-ids` identity subdomain exposes the same `/api/ms-account/` without Cookie. GET list may return 404; POST may return data.

Test (while logged out):

1. Call the same list endpoint on the business frontend as a control; it should require login.
2. Instead call identity subdomain `POST /api/ms-account/user/list`; try empty body / `Offset+Limit`.
3. If it works, try `generate` → `reset-pwd` with only Id (without Id should be a parameter error) → `delete` to clean up the probe account.

Count as success: List contains phone/company email; can create an account, change password without old password, or delete by Id.

False positives: Testing only the business frontend and concluding there is no issue; list returns data but Mail/Phone are all empty and write endpoints are gated. A miss on one site does not remove this row from the short table.

### Detail is not gated by list filters (short table has a pointer)

Recognize: Public list shows only published/public items; detail, hidden, tab, preview/export use the same business ID. Or list has a visibility query parameter that filters hidden entries by default. Or list requires login/empty body while detail needs only numeric ID + business key (such as workid). Or external detail blanks contact/phone while review/approval detail uses the same business ID. Or documentation CMS frontend hardcodes a public `area` / `client` / `channel`, but API has no auth. Or public search has material-type parameter (`materialType` / `tab`) defaulting to public. Or onboarding/review query with only business ID returns an empty shell. Or same-product full list without visibility filters directly includes unpublished/offline detail bodies while public search and showcase details remain gated. Or browse/catalog says visitors must log in, but same-site search API still returns internal/proprietary document bodies while logged out.

Test (while logged out):

1. Note public-list IDs; try IDs absent from the list against detail / hidden / tab; if detail returns 403, continue to preview / export.
2. Add `status=hidden` / `all` / `private` (or equivalent visibility parameter) to the list endpoint itself. Control: without parameter, list should be empty or public only; even if detail for the same ID still says “no permission,” don't stop — the gate may apply only to detail.
3. **If this list endpoint returns 401 / requires login / empty body, don't stop.** Try another guest query in the same product and inspect `hidden` / `is_public` / `isHidden` in its response. If public `contents` returns 404, try hidden/contents with the same ID. Keep trying detail that needs only numeric ID + business key. If `is_secret=1` still returns full text, don't stop at the title.
4. **If detail returns 200 and JSON says unpublished / login required / `canAnswer=false`, don't stop.** Check whether `savedConfigDraft` / similar draft fields in that response contain the unpublished body. The denial message and draft can appear in the same JSON.
5. **If external detail blanks contact / mobile / email, don't stop.** Try review / approval / audit detail with the same ID; the gate may apply only to public display endpoint.
6. **If work/project has `period=edit|publish` (or similar state parameter), don't stop at publish.** While logged out call content endpoint with `period=edit`. Control: publish for same ID says nonexistent/deleted and info blocks “no permission to read others.” Count it only if edit still returns the body.
7. **If JSON list filter is null / missing field reports Unknown / invalid parameter, don't stop.** Set that field to empty array `[]` and try again. Backend often treats empty array as “no filter,” exposing unpublished/all rows anonymously; omitted/null may instead be blocked. Control: `null` or removing the key should error/return empty, while `[]` returns unpublished content.
8. **If documentation CMS frontend hardcodes public `area` / client / `channel`, don't stop at the default.** While logged out, change the parameter to the internal area (often 1 vs 2, oa vs public). Count only if integration/hardening/intranet algorithm content absent from the default public list appears in list or detail.
9. **If public search has `materialType` (or similar material-type field), don't stop at the frontend's default public value.** Compare public standards/showcase type, then change it to internal-market materials/marketing standards. If path missing `/v2/` returns `auth failed`, don't stop; add it following the frontend's replacement rule and retry. Success: internal material list/file URLs behind the SSO wall, with `isLimitAuth=1`.
10. **If self-service onboarding/review query with only store number/business ID returns an empty shell, don't stop.** Add review status=approved (`baseInfoStatus`, etc.). Follow `uid` from response to email/contact endpoint immediately. Control: without status it should be a shell or lack phone.
11. **If certificate-image JSON has no idNo, don't stop.** Open trademark registration/license scans in detail: registrant row often appends national ID to the person's name. Count it if image visibly matches; don't scan only JSON fields.
12. If export-task list with `pageNum=0` is empty, don't assume no data. Change to `pageNum=1`; it often returns other people's exports sitewide. Copy `fileName` / key to download.
13. **If detail telephone/mobile is masked, don't stop.** Unpublished detail `requirement` / notes / free text often contains the full 11-digit phone, while only structured fields were stripped.
14. **Also test full-list endpoints without visibility parameters.** Check rows directly for unpublished / offline / rejected detail bodies. Control: public search for that ID has `total=0`, and the person's showcase says nonexistent. Don't test only public search/detail and conclude there is no issue.
15. **If browse/catalog says visitors must log in, don't stop.** Same-site search API (Keyword+Page+PageSize) may still return internal/proprietary document body and detail URL while logged out. If detail JSON says `isLogined=false` but includes full text, don't stop. Control: the human-facing catalog says visitors must log in to view.
16. **If detail endpoint has been fixed / says project nonexistent, don't stop.** Same-site visit-log / telemetry / doc-add-visit-log may still expose the full OA/internal document by numeric article number. Control: original detail endpoint should now block the same business ID.
17. **If catalog says visitors must log in to view/download, don't stop.** Same-site file-list endpoint (`getHomeCosFileList`, etc.; `solution_id`+`version_id`+`folder_id=0`) may expose training videos/whitepaper/manual list while logged out. Send list `file_id` to permission endpoint (`getFilePrivilege`) and inspect `PreviewUrl` / signed download. Control: human-facing catalog says visitors must log in to view and download.
18. **If content SPA uses a `location.host` regex to select Color/gateway domain, don't use the wrong TLD.** JS commonly uses `.*\\.jd\\.hk` → `content gateway domain`; using the `wrong content gateway domain` makes the endpoint look absent. Still test unauthenticated floor queryContents + previewContentDetail. Control: public detail should be nonexistent/1414.
19. **If public documentation-site itemList/catalog only shows external products, don't stop.** While logged out change `item` (or similar item number) to an auto-incrementing integer and call detail/page. Body may be compressed markdown; decode it as the frontend does. Count only libraries marked “internal/not externally released/not for external use” that are absent from public itemList.
20. **If a collection-form/questionnaire detail JSON has `relative` / linked table pointing to an answer-sheet, don't stop.** Human-facing page is the submission form; answer table may have a separate ID. While logged out call answer-sheet detail/opendoc. Control: form page has only questions; answer table contains ID/phone cells.
21. **If content preview only needs numeric contentId and response says `isDelete=1` / deleted / unpublished, don't stop.** Control: item should be absent from public showcase. Count only if full H5/editor JSON is returned, not just title.

Count as success: Unpublished business body appears in a list (not a public showcase title); another entity's phone/ID; an ID printed in certificate registrant line; or unpublished/offline work source/body.

False positives: Adding visibility parameter still returns only public content; only title, no body; list 401 treated as no issue; edit and publish return the same already-published body; empty array and omitted parameter return the same published content; `materialType` still returns public showcase only; only business ID returns empty shell and no review status tried; only public search/showcase detail tested; browse page says login required and testing stops; numeric item still returns same public documents. Using wrong Color domain is a mistake, not a false positive. A miss on one site does not remove this row from the short table.

### Gateway passes without an auth header (short table has a pointer)

Recognize: Logistics/open gateway (such as `lop-proxy`) frontend JS hardcodes `Auth-Control: no-auth` + `LOP-DN` (business domain). Other business endpoints require login, but list/waybill endpoints return data with this header set.

Test (while logged out):

1. Copy `LOP-DN` and `Auth-Control: no-auth` (or equivalent gateway-auth header) from JS.
2. Try empty-page requests to site/list endpoints; then change enumerable `siteId` for waybill/detail.
3. Control: without header it should require login or report unregistered Host.

Count as success: Other people's phone/address/site contact, not public tracking data.

False positives: Header passes but business endpoint still requires login; only public tracking and no PII. A miss on one site does not remove this row from the short table.

### Anonymous session reads registration list (short table has a pointer)

Recognize: BaaS anonymous session; list endpoint does not check for administrators. Page hardcodes a few admin IDs and backend redirects to login. Appwrite / Weavefox / TablesDB are common skins; test even if names are absent. Frontend often has `Client().setEndpoint`, `X-Appwrite-Project`, `POST /v1/account/sessions/anonymous`; registration/event table uses `tablesDB.listRows`.

Test (logged out; do not use admin account):

1. Without Cookie call `GET /v1/tablesdb/{db}/tables/{table}/rows` — baseline should be `total=0` / empty rows (system actually blocks guests).
2. `POST /v1/account/sessions/anonymous` with body `{}`, same `X-Appwrite-Project`; get `Set-Cookie: a_session_…`.
3. With this anonymous session call the same listRows again.

Count as success: After anonymous session, `total` jumps above empty-session baseline and rows contain **other people's** phone/email/registration content. Admin allowlist exists only in browser and is not applied to list endpoint.

False positives: No session already reveals whole table (that is complete lack of auth, report separately); anonymous session remains empty or contains only your own new row; DELETE on someone else's row returns 401 only proves writes are disabled, read still counts; public operations display list. A miss on one site does not remove this row from the short table.

Separate from “no Project key, CRUD works without even an anonymous session”: the key here is **anonymous session treated as authenticated, with no admin check**.

### Cloud development anonymous user table (short table has a pointer)

Recognize: Cloud development has anonymous login enabled; low-code data-source functions exist. Environment ID can be copied from console/frontend. This is not the existing proxy `targetUrl` case or BaaS `sessions/anonymous`.

Test (while logged out):

1. `POST https://{envId}.api.cloud-development-gateway/auth/v1/signin/anonymously`, header `x-device-id`, body `{}`; obtain JWT.
2. `POST /v1/functions/lowcode-datasource`, with that ticket in `Authorization: Bearer`, body `dataSourceName=sys_user`, `methodName=wedaGetRecords` (`getRecords` is the same source).
3. Control: same gateway `GET /auth/v1/user/query` should block anonymous users; data source name `users` often fails row permissions, **don't stop** — `sys_user` is the system-user table.
4. If HTTP gateway (often `/web?env=`) returns `LOGIN_TYPE_DISABLED`, **don't stop**: change to `POST /web?env=`, call `auth.signInAnonymously` → `auth.getUserInfo` to obtain `jwt;expire` (**do not strip the suffix after the semicolon**) → `functions.invokeFunction`, and call `sys_user` / `wedaGetRecords` the same way.
5. If old HS256 ticket sent to `/v1/functions` returns `KID_INVALID`, this probe failed; use the gateway ticket from step 4 instead. Do not conclude the data source is absent.
6. If `sys_user` fails row permissions, **don't stop**: use same anonymous `/web` ticket with `database.countDocument` / `database.queryDocument`, `collectionName=users` (user table populated by federated login, not data-source name `users`).

Count as success: records contain **other people's** phone/email/uin/super-admin flag, not an empty account you just created anonymously.

False positives: Row permissions block anonymous access and `/web` users collection is also empty; data source does not exist; only demo todo/sales data. Do not treat old HS256 `KID_INVALID` as no issue. HTTP gateway `LOGIN_TYPE_DISABLED` is not a false positive. Data source `users` failing is not a false positive. A miss on one site does not remove this row from the short table. Do not store actual secret values or a particular JWT.

### Copy openid from detail and query the mailbox (short table has a pointer)

Recognize: Recruitment/specified-user detail contains `specifyUsers` or openid; H5 hardcodes SHA256 request signature; or invite-page URL already has `?openid=`.

Test (while logged out):

1. Call detail and copy openid.
2. Compute Signature using the key and call `/msg/message/list/`.
3. If URL already has openid, also try unsigned `get_invite_code` / `count` / `detail`; a listed `uid` can also be supplied.

Count as success: Changing openid changes mailbox/invite code/list and reveals the other person's points-expiration notice or masked phone/nickname.

False positives: List is identical for everyone as a sitewide broadcast; endpoint requires real login/signature; fake openid has no invite code; public operations list. A miss on one site does not remove this row from the short table.

### Unauthorized full-text read by knowledge-base nodeId (short table has a pointer)

Recognize: Published-page conversation-data `artifactMap` contains knowledge-base `nodeId`; main site `/space/d/` requires login.

Test (while logged out): Call publishing domain `POST /space/api/page/share/query/pagechunk` (body=`pageId=nodeId`). Control: the share page's own pageId should say “cannot be shared.”

Count as success: Retrieve someone else's full document (title/character card/outline, `role=editor`).

False positives: Only reads published HTML snapshot; pagechunk returns 12607 for knowledge-base too; body already exists in conversation snapshot. A miss on one site does not remove this row from the short table.

### Anonymous CSRF header reads detail (short table has a pointer)

Recognize: Anonymous CSRF/temp-token endpoint directly issues a token; business detail checks only this header and not login; numeric query ID and response `data.id` can differ. This is not the assistant's optional `GetHistoryList` login header (see “Assistant history reads another person's task without authorization”).

Test (while logged out):

1. Call csrf/token endpoint to obtain header.
2. Change detail numeric ID (1, 2, 13); **if query ID differs from response ID, don't stop**.
3. Control: without header it should fail; nonexistent ID should return “not found.”

Count as success: Unpublished/test training script or internal session body (cardMap/full dialogue, not just title).

False positives: Only public plaza; token passes but result remains empty. A miss on one site does not remove this row from the short table.

### Custom identity header treated as session (short table has a pointer)

Recognize: Admin SPA request interceptor treats a custom header (`User-Id` / `employeeId` / `X-User-Id`, etc.) as login identity, without Cookie. Unauthenticated list often returns only numeric employee/BD IDs, not phone numbers.

Test (while logged out):

1. Copy header name from JS interceptor.
2. Call unauthenticated list (empty args/headquarters filter). If it returns only numbers, **don't stop**.
3. Set header to one of those numbers and call current-user info endpoint. Control: without header or with `1` should return no result.

Count as success: Return another person's name + 11-digit phone.

False positives: Header passes but still empty; list already returned phone (that's a different finding). A miss on one site does not remove this row from the short table.

Separate from “anonymous CSRF header reads detail”: that uses a temporary token header; this one puts a **numeric identity in the header**. Separate from “onboarding H5 empty args expose BD contacts”: that returns a whole contact list with empty category, while this list has no phone and requires a second request treating the header as identity.

### Customer-detail endpoint also accepts phone (short table has a pointer)

Recognize: CRM/enterprise-IM customer detail endpoint frontend sends only external-contact ID (`externalUserId` / `contactId`). Sending only this ID returns an empty shell. Backend also accepts `phone`, and may require `tagShow` / `udfShow` to reveal internal fields.

Test (while logged out):

1. Control: only external-contact ID should return empty shell.
2. Add 11-digit `phone` to body and enable `tagShow` / `udfShow` (or equivalent expand fields).
3. Fake/nonexistent number should return empty shell; changing number should change person.

Count as success: Return another person's name/company/internal UDF, corresponding to the complete phone number queried.

False positives: phone is always blank or only the same test account appears. Empty national-ID slot alone is not a false positive. A miss on one site does not remove this row from the short table.

### Assistant history reads another user's task without authorization (short table has a pointer)

Recognize: Assistant/Agent frontend has `GetHistoryList`; login state is only in an optional header. This is separate from executing a tool in chat (see `agent-tool-exec-test.md`).

Test (while logged out): Call list and change guid to see if the same batch appears; then pass `session_id` to `GetHistory`. Paginate using `last_ts`+`direct=back`.

Count as success: List/detail contains another person's original task (downloads, subscriptions, dialogue cards), not public plaza content.

False positives: Only public plaza `GetSquareTasks`/`share_id`; changing guid makes list empty or leaves only your own entries. A miss on one site does not remove this row from the short table.

### Hardcoded appKey accesses business tables (short table has a pointer)

Recognize: Frontend hardcodes `appId`+`appKey` in `AV.init` / LeanCloud (or `X-LC-Id`/`X-LC-Key`); or nocode/supabase landing page hardcodes `role=anon` JWT.

Test (while logged out): Send both headers to `/1.1/classes/*`: try `_User` first as a 403 control, then scan business-table count/limit; if writes work, change a probe field and revert. For supabase, use `apikey`+`Authorization: Bearer` on tables listed by `/rest/v1/` swagger. **If REST tables return 403 / only public operation config is exposed, don't stop**: call `POST /storage/v1/object/{bucket}/{official-prefix}` with `x-upsert:true` (see STS step 9 in `file-upload-test.md`).

Count as success: Business-table `count` is huge or reveals **other people's** drafts/email/phone; PUT successfully modifies another object's `objectId`.

False positives: Both `_User` and business tables return 403; can only read your own newly created data; can only LIST public operations config. A miss on one site does not remove this row from the short table. Do not store actual secret values.

### Form model exposes answer key (short table has a pointer)

Recognize: Questionnaire/quiz form-model endpoint; frontend hardcodes business ID or allowlist (`FORM_WHITE_LISTS` / demo `encryptFormId`); response schema includes standard answers in `answer` or internal review questions/sample images. This is not a public registration-form title or list of submitted answers.

Test (while logged out):

1. Use JS allowlist/demo ID to call getModel / schema / getFormModel; don't stop at form title.
2. Follow sample image preview/imgUrl in schema.
3. Also test admin list/export/fillList; don't fabricate answers if there are no submissions.

Count as success: Internal quiz body + answer key, or actual ID/license sample image downloads.

False positives: Only public registration title; schema has no answer; only your own new submission. A miss on one site does not remove this row from the short table.

### Unauthorized internal knowledge-base text (short table has a pointer)

Recognize: Customer-service/developer-support umi has `getKnowledgeList.json` + `getKnowledgeInfo.json`; or lobby/help HTML detail accepts numeric article number; or public announcement JSON (`bulletin` / getbulletin) uses `callname` + `callcontent` as RPC while page calls only public menu (`getKnowledgeByMenuId`); or unauthenticated CMS `siteList` / `contentList` / `content` can list non-official sites with channel names containing “internal knowledge base”; or customer-service/IT chatbot unauthenticated search endpoint (menu ID + fuzzy searchText) stores body under `buttonList`/`searchList` `behavior.value`, not title `content`. Do not recognize only the umi JSON pattern.

Test (while logged out): Control `queryUserInfo`/`queryFeedbackList` should deny. Start list `categoryId` at 1, then send `id` to Info. **If no JSON list, try HTML detail too** (`showKnowledgeInfo.htm?knowledgeId=` / `help_detail.htm?help_id=`), with a modern UA (IE may hit netd). Announcement endpoint: if public menu has very few entries, change `callname` to `getKnowledgeList` (`callcontent` includes pagination), then call `getKnowledge` with detail ID. CMS: copy non-official siteId from `siteList`, inspect channel names via `contentList`, change siteId and call `content` detail; default official Banner is not this finding. If gateway says `loginMode is null`, don't stop; add header `loginMode: 0`; if siteList still fails due to missing siteId, directly call contentList with internal siteId. content also needs siteId; without it you may get no body. Chatbot: if `chat_dir_id` directory endpoint returns 401, don't stop; call search endpoint (`search_recommend`, etc.), set `searchText` to common words and `id` to menu number; inspect body in `behavior.value`.

Count as success: `pager.items` list has thousands or count is huge, and Info/HTML/detail/`behavior.value` reveals **internal** scripts/investigations/SMS/operations knowledge-base text, not public FAQ/agreements.

False positives: Only public help/error codes/external agreement/official Banner; Info returns title only; ticket endpoint is also open (separate issue); only default official site tested; directory node 401 ends test; search returns only title content. A miss on one site does not remove this row from the short table.

### Open-payment onboarding gateway enumerates appId with fake signature (short table has a pointer)

Recognize: Open-payment/onboarding gateway body has `appId`+`sign` (may also include `random`/`merchantId`). With fake signature, live app returns `MERCHANT_NOT_EXIST` or `SUCCESS`; dead app returns `SIGN_ERROR` / `APP_NOT_FOUND`. This is not “compute your own signature from a hardcoded JS salt” or a real AppSecret copied from docs.

Test (while logged out):

1. Set `sign` to a fake value (such as 32 `a`s), enumerate `appId`, compare response codes.
2. For a live one, call merchant query (`query` / `query/v2`) and change `merchantId` (docs sample ID, neighboring ID).
3. With the same fake signature, also probe signing/agreement endpoint; only check whether response accepts it. Do not bulk sign or change settlement card.
4. **If ST/demo checkout precreate with fake signature or node-side signature also reaches production gateway**: while logged out POST `/api/precreate` or `/demoapi/precreate`, using fake sign + live appId + another merchantId. If this site's order query is 404, don't stop; call production `/api/pay/query`.

Count as success: Return another merchant's ID/bank card/phone; or production order query returns `ORDER_NEW` with their transaction number.

False positives: Fake signature always returns `SIGN_ERROR`; `SUCCESS` but all identity/card fields are blank; documentation sample appId is dead; only `channelPayerNo` returned and production query has no order; works only for a demo store. A miss on one site does not remove this row from the short table. Do not store actual secret values.

### Short-link 302 query contains phone number (short table has a pointer)

Recognize: SMS/operations short-link resolver; guessed short code returns 302 to landing page with plaintext `phone` / `name` / amount in query. Homepage may be OpenResty welcome page, 302 to brand site, or `index.html` with only Hello; **do not use `/` alone to declare the whole site safe**.

Test (while logged out):

1. Try a short dictionary on `/open` `/app` `/s/` `/www` (`aaaaa`, `1`, `1234`).
2. Inspect only 302 target query; do not follow into landing business domain.
3. Control: unmatched code should show expired page, no phone number.

Count as success: Redirect URL contains **someone else's** phone/name.

False positives: Expired page; public marketing contains no PII; shortcode requires a real SMS to resolve. A miss on one site does not remove this row from the short table. Homepage Welcome / 302 official site / Hello shell is not a false positive.

### Privacy-number failure returns real number (short table has a pointer)

Recognize: Order/store/logistics H5 calls privacy-number or virtual-number (AXB) endpoint; unauthenticated request needs only business appid + enumerable object number. On failure, response discloses real 11-digit phone used as fallback and says privacy-number retrieval failed, will dial real number. Don't discard it as merely a virtual-number endpoint.

Test (while logged out):

1. Copy functionId / privacy-number path from frontend; try object numbers 0, 2, neighboring values.
2. Control: nonnumeric object number should be parameter error and no phone.
3. Change the number; it counts as batch exposure only if returned phone changes.
4. If empty Origin or this site's Origin gets `cross-origin 403`, **don't stop**; change Origin to business domain (order/cart/H5 domain, not tied to one host) and retry.

Count as success: Response contains **another person's** real phone.

False positives: Only virtual/intermediate numbers; login required; changing object number doesn't change the phone; empty Origin 403 treated as no endpoint. A miss on one site does not remove this row from the short table.

### Onboarding H5 empty args expose BD contact list (short table has a pointer)

Recognize: Onboarding H5 has a merchant-recruitment phone page; other settle endpoints in same suite 302 when logged out. JS has a BD lookup RPC by category (queryBd / bdInfo, etc.); when industry/category is empty, body is empty. Page may display only the public 400 hotline.

Test (while logged out):

1. Copy gateway `/api` and functionId from recruitment-phone page JS.
2. Call with category field empty (`body={}`). Control: hardcoded numeric category often returns an empty array.
3. Other settle endpoints in same suite may 302; don't treat that as no site entry point.

Count as success: Internal BD full list with names + 11-digit phones + business emails.

False positives: Hardcoded numeric category returns empty array and test stops; treating the page's public 400 hotline as this finding. A miss on one site does not remove this row from the short table.

### Onboarding H5 hardcoded token exchanges pin (short table has a pointer)

Recognize: Onboarding H5 / mini-program bundled JS hardcodes token for OCR/business-info endpoint. JSON endpoint needs only pin (or similar account) + nonempty token, no Cookie. Control: empty/omitted token returns only empty data.

Test (while logged out):

1. Copy hardcoded token and business-info path from onboarding chunk.
2. Empty-token control should return empty data.
3. Use hardcoded token with pin `admin` / `test` / `0` / short account and retry.
4. After hardcoded token passes, try any other nonempty string as control (if that passes, token content is not validated).

Count as success: Changing pin returns corresponding real phone.

False positives: Empty token also returns data (complete lack of auth, separate issue); token passes but only returns the account you just onboarded. A miss on one site does not remove this row from the short table. Do not store actual secret values.

### Company-name / invoice-title autocomplete (short table has a pointer)

Recognize: Registration/onboarding/invoice-title autocomplete; dropdown shows only company names; endpoint calls business registry/Dun & Bradstreet Match or title suggest. Frontend may mark endpoint `auth:true` while server still skips login validation.

Test (while logged out):

1. POST name+country code, or call title endpoint. suggest field may be `prefix` (using `title` returns “keyword cannot be empty,” making it look absent).
2. If suggest reports empty keyword, don't stop: same product may also have a special-invoice/business-registry completion endpoint; its field is `title`, frontend marks `auth:true`, still test it.
3. Compare with page dropdown: page shows only company name; count only if response includes phone/card/address.

Count as success: Response includes responsible person or invoicing phone/address/bank account that matches a person.

False positives: Both dropdown and API contain only company names; login required; public company directory has no phone; frontend marks auth so test stops. A miss on one site does not remove this row from the short table.

### Mass assignment / hidden writable fields (short table has a pointer)

Recognize: JSON for registration/profile edit/user creation has more fields than page controls; Swagger, admin “create user,” or frontend comments reveal extra `role` / `isAdmin` / `verified` / `tenantId` / `balance` / `permissions`.

Test:

1. Compare admin user creation vs. your own registration/profile edit; add only fields that differ. If there is no comparison, copy them from Swagger / JS.
2. Add fields only to your own account once; revert immediately if successful. Do not change roles on another person's existing account.
3. Also try `__proto__` / nested `user.role`; this is separate from PP template RCE (see `prototype-pollution-test.md`).

Count as success: Your own account becomes privileged or balance/verification state actually changes. Field is accepted but privileges do not change → no success.

False positives: Can only change display name; public operations toggle; server-side allowlist discards extra keys. A miss on one site does not remove this row from the short table. Do not invent fields if no registration/profile-edit endpoint exists.

## 12. QUICK IDOR CHECKLIST

```
□ A control account helps; for one account use lists/responses/neighbors, don't grind registration for a second
□ Map all API calls that contain object IDs (Burp History export filter)
□ Test all HTTP verbs on each endpoint (for writes: POST to add first, then delete your own addition; never delete another person's existing object)
□ Test ID in all locations: path, body, header, query, cookie
□ Try sequential IDs (−1, +1 from your own)
□ Encrypted ID: if JS has public key, encrypt neighboring numbers yourself (see “Encrypted IDs”)
□ For tenant fields in lists, try 0 / -1 / empty (also try when a real other ID fails; see “Sentinel tenant”)
□ If Registry/`/v2/`: with your own account call `_catalog`; can you list and pull another tenant's image? (see “Registry catalog”)
□ After BaaS anonymous session, call listRows again and compare to empty table without Cookie (see “Anonymous session reads registration list”)
□ After cloud development anonymous login, call `lowcode-datasource` `sys_user`/`wedaGetRecords`; don't stop at HTTP gateway `LOGIN_TYPE_DISABLED` (see “Cloud development anonymous user table”)
□ If detail includes specifyUsers/openid: copy openid and query mailbox (see “Copy openid from detail and query the mailbox”)
□ Send publishing-page artifactMap nodeId to pagechunk (see “Unauthorized full-text read by knowledge-base nodeId”)
□ Assistant GetHistoryList optional header: call list/detail while logged out (see “Assistant history reads another user's task without authorization”)
□ If frontend hardcodes LeanCloud/supabase anon, test business tables (see “Hardcoded appKey accesses business tables”)
□ Unauthenticated knowledge list + detail returns internal scripts (see “Unauthorized internal knowledge-base text”)
□ Open-payment onboarding gateway: enumerate appId with fake signature, then change merchantId on live app (see “Open-payment onboarding gateway enumerates appId with fake signature”)
□ For SMS short-link, don't stop at homepage Welcome / 302 official site / Hello shell; guess /open /app /s/ /www and inspect 302 query for phone (see “Short-link 302 query contains phone number”)
□ For unauthenticated privacy/virtual-number endpoint, check whether failure returns real phone; object number may be enumerable (see “Privacy-number failure returns real number”)
□ Registration/onboarding/invoice-title autocomplete: test suggest (prefix) and special-invoice/business completion endpoint (title) while logged out; test even if frontend marks auth (see “Company-name / invoice-title autocomplete”)
□ Onboarding H5 recruitment phone page: JS BD endpoint, leave category empty; do not hardcode number (see “Onboarding H5 empty args expose BD contact list”)
□ If backend SPA treats User-Id header as identity, don't stop at numeric-only list; change header to listed ID and call current-user-info endpoint (see “Custom identity header treated as session”)
□ If browse/catalog says visitors must log in, don't stop; call same-site search (Keyword+Page) while logged out and follow detail URL (see “Detail is not gated by list filters” step 15)
□ If catalog says login required to download, don't stop; test same-site file list + permission endpoint PreviewUrl while logged out (see “Detail is not gated by list filters” step 17)
□ Change external search `materialType`/`tab` to internal type; if missing v2 path returns auth failed, don't stop (see “Detail is not gated by list filters” step 9)
□ If onboarding/review query with business ID returns empty shell, add review status=approved and follow response uid to email endpoint (see “Detail is not gated by list filters” step 10)
□ If public documentation itemList shows only external products, don't stop; set item to numeric ID and call detail/page (see “Detail is not gated by list filters” step 19)
□ If form/questionnaire detail relative points to answer sheet, don't stop; call answer-sheet table while logged out (see “Detail is not gated by list filters” step 20)
□ Supabase anon JWT: beyond REST tables, call Storage REST + x-upsert against official prefix (see file-upload STS step 9)
□ Try UUIDs/GUIDs collected from your own account data
□ Test sub-resources (attachments, comments, transactions)
□ Test admin endpoints directly (BFLA)
□ Test POST/PUT body for extra fields (mass assignment)
□ Compare JSON response field count vs documented fields (hidden fields)
□ Test state/status: change only your own order / test fields you can revert
□ Merchant promotion/coupon binding: change productId to another person's product; count only if customer-side price drops (see logic-test.md §1.4)
```

---


Merchant promotion/coupon bound to SKU: validates coupon ownership but not product ownership → your coupon + someone else's productId; count only if customer-side price drops. Full section is in logic-test.md §1.4.



