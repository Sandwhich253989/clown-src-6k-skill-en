> Whether to write a report is determined only by `~/.grok/rules/vuln-report-format.md`. This article is a testing method: don't stop at being able to upload and download; follow through to execution, paths, SSRF, and cross-user business objects.
> For short-table pointers, search by heading. PHP shells / GIFAR / ImageTragick / English attachments have been cut; don't blindly test cross-bucket issues when there is no object storage.

# File Upload Vulnerability Testing Guide

## Testing flow (opening)

Find upload points (avatar/attachment/import/rich text) → first upload a normal file and check the path, whether it is renamed, whether it lands on a CDN/OSS or local disk, and whether it can be opened directly. Don't stop at being able to upload and download: follow through to overwriting another user's object, signed access, execution/path/SSRF. Adapt extension/Content-Type/parser bypasses to the stack you encounter.

### STS / wildcard overwrite of object keys (short table has a pointer)

Recognize: Object storage (BOS/OSS/S3/TOS, etc.) where upload first requests STS/pre-signed credentials; object keys are often file MD5s; bucket names increment by date or batch. Or a credential-issuing endpoint such as `assumerole`, where `filename` / `Action` in the request is used to build the Policy. `xluser` / AES / `1.jpg` below are common skins; **test wildcards even if these strings are absent**.

Test:

1. If `identifier` / `key` / `object` / `prefix` / `filename` in the credential request has **no length validation**, try `*`, `**`, then try **empty** and `/` (some implementations treat an empty prefix as the whole bucket).
2. One `*` may land in an empty bucket; **two or more** may match the current business bucket, making the credential `key` a wildcard for that bucket.
3. For paths / `actionName` / path parameters inserted into the Policy, try `../../../` and `../../../&/../../`; check whether the signed scope expands to the root.
4. Another user's object key is often its MD5: open the document page and search the JS/API for `md5sum` / `md5`; no need to download or pay.
5. With the STS credentials just issued, set the upload path to that other user's MD5 and overwrite the object.
6. Fire another request at `assumerole`: try `filename=*-*` (or `*`) and `Action=*`. Some implementations have the **later Policy override the earlier one**, turning the credentials into whole-bucket access. If it works, start with `list_objects`, then delete/overwrite listed keys. **Don't stop at List/Delete 403**: GET/PUT an arbitrary key. Also test the credential-issuing XHR without login headers. **`illegal user id` on a sibling drive's anonymous ticket does not mean this gateway rejects it too**: test the STS endpoint that accepts anonymous xluser tickets. If query `upload_dir`/`dir` is passed unchanged into the OSS Policy, set it to `*`. If the page uses a CDN, force-refresh or wait for the cache to expire before verifying.
7. On PUT, try headers `x-cos-acl: public-read` and `x-cos-grant-full-control: id="your UIN"` (equivalent headers exist for OSS/S3). Count it only if **the object actually becomes publicly readable or control is granted to your UIN**; a 200 alone does not prove takeover.
8. If there is no “file exists” oracle, still guess short original filenames (`1.jpg`/`2.png`/`5.jpg`) with anonymous CDN GETs. If the token endpoint has AES/salt hardcoded in frontend code, calculate the signature yourself while logged out; `filename=*` often directly returns appId+bucket, and combining the CDN prefix can expose **other people's ID photos**.
9. **If a landing page hardcodes a supabase / nocode `role=anon` JWT, CRUD on REST tables is not the end.** With `apikey` + `Authorization: Bearer`, send `POST /storage/v1/object/{bucket}/{official-prefix/probe-key}`. Official covers/case images are often under prefixes such as `use-cases/` and `covers/`. First POST your own probe file; POST again to the same object name with `x-upsert: true` and change the body to a different marker string. Compare the two bodies using public `GET /storage/v1/object/public/{bucket}/{key}`. **Do not overwrite real official images**; if you can overwrite your own file under the same prefix, official objects under that prefix are exposed to the same overwrite. DELETE the probe file after testing. If REST `/rest/v1/` returns 403, don't stop; Storage REST may be enabled separately.

Count as success: Reopen/download **the other person's document/image** and see its contents changed to yours; or list other people's keys and delete/overwrite them; or use `x-upsert` under an official prefix to replace a probe file with the second marker string (official objects under the same prefix use the same key). Proving only that you can upload to your own key is a false positive.

False positives: `*` only signs for a dead bucket; wildcard works but overwrite gets 403; you changed your own object; server-side policy prevents overwriting; only your own prefix works; CDN never refreshes so it looks as if the overwrite failed. A miss on one site does not remove this row from the short table.

This is separate from S3 pre-signing “change your own Content-Type,” “use a signed URL to change the tenant and read their files,” and the following “bucket policy open to anonymous users,” “signature not bound to Host,” and “signature covers Content-Type”: this finding is **credential scope wildcarded to the whole bucket, enabling overwrite or deletion of other users' objects**.

### Signature not bound to Host (short table has a pointer)

Recognize: COS / OSS / S3 signed URL; query has `q-header-list`, `SignedHeaders`, or `X-Amz-SignedHeaders` **without `host`**. Test only when object storage has signed URLs; don't blindly swap domains otherwise.

Test:

1. Check which headers are signed in the signature parameters. Continue only if `host` is absent.
2. Change the URL host to the domain of **another bucket in the same account** (copy it from certificate SANs, errors, console, or JS); leave the path and signed query unchanged.
3. Then add `?uploads` (or `&uploads` after an existing query) and request ListMultipartUploads; also try GETting an object if possible.

Count as success: Listing or reading objects from **another bucket**. If only the domain changed but it is still your bucket, it doesn't count.

False positives: Signature covers Host; changing the host returns 403; only your own bucket. A miss on one site does not remove this row from the short table.

This is separate from STS `*` (credential scope is wildcarded) and signed URLs that only change a tenant field: this finding is **Host omitted from the signature, so the same signature reaches another bucket**.

### Signature covers Content-Type (short table has a pointer)

Recognize: You control the object contents (your own upload or one STS can overwrite); Content-Type on the object is fixed (such as `image/jpeg`); you have a temporary key or can request another pre-signed URL.

Test:

1. Still try changing CT during upload (`text/HtMl`, `text/html,image/png`). That tests whether “CT was not included in the signature.”
2. **Sign again afterward**: use the temporary key to generate a GET for an **existing object** with `response-content-type=text/html` (COS/OSS support signing an overridden response header).
3. Open the new signed URL in a browser; don't only inspect curl's Content-Type.

Count as success: The browser executes the content as HTML (stored XSS). If it remains an attachment or the original CT, it doesn't count.

False positives: Signing endpoint rejects this parameter; you can only change an inaccessible object of your own; `X-Content-Type-Options: nosniff` is set and CT remains an image. A miss on one site does not remove this row from the short table. If there is no object storage or temporary key, don't blindly try signing.

This is separate from “change Content-Type during upload PUT”: that changes the type **when writing**; this one **uses a signature to override the response header to HTML when reading**.

### Permanent object-storage key in webpack plaintext (short table has a pointer)

Recognize: The webpack bundle for an admin/operations console includes production `accessKeyId` + `secretAccessKey` (permanent MSS / S3-style credentials). Or a Weblogic `/console/login/LoginForm.jsp` has an inline `_reportCfg`-style `SRV_` key. Upload signing via `getUploadSign` computes the policy HMAC-SHA1 in frontend code rather than using an authenticated STS endpoint. The policy's `starts-with $key` is often an empty string.

Test (while logged out):

1. Copy the AK/SK from online/production (put actual values only in the report).
2. Compute a POST policy yourself, extend expiration, and reproduce `starts-with $key` exactly as in JS (if empty, keep it empty).
3. POST to the bucket. Compare against a fake signature (`SignatureDoesNotMatch`) and no signature (`conditions has no signature`).
4. PUT any key, then CDN GET it; DELETE the proof key you just uploaded to confirm write access. Try overwriting an object already referenced by the official page.
5. Don't stop at List/PUT 403: first try `GetBucketLocation`. Compare with a fake AK (`InvalidAccessKeyId`). If the page's bucket name is misspelled (staic/static), try nearby names.

Count as success: Complete permanent cloud credentials can sign (a real signed PUT returns 200 or `GetBucketLocation` returns a region). Overwriting an existing official object is stronger proof.

False positives: `InvalidAccessKeyId`; only uploads to a fixed prefix; `getUploadSign` is actually an SSO endpoint with no local key; key is expired. Do not store secret values in the database. A miss on one site does not remove this row from the short table.

Separate from STS wildcard (request a temporary ticket first), bucket policy open to anonymous users (no key needed), and viewer XOR-obfuscated COS permanent key (must decrypt it and then query AccountId): this finding is **a plaintext permanent key in webpack plus frontend-side signing**.

### Bucket policy open to anonymous users (short table has a pointer)

Recognize: Official site images, user guides, or agreements are served directly from an OSS / cloudrun-style bucket domain; the bucket root or `?policy` / GetBucketPolicy / equivalent policy endpoint can be fetched and contains an all-Allow statement.

Test:

1. Inspect the policy if available; then LIST the bucket (without AK).
2. Try PUT / overwrite on an **object already referenced by the official site** (do not modify the bucket policy itself).
3. Prioritize guides, agreements, and site images; after overwriting, open the original official URL to verify.

Count as success: The official guide/agreement/image content changes to yours. Proving only that anonymous users can LIST, or that you can upload only to a new key nobody references, is a false positive.

False positives: Policy is read-only; PUT only works in your own prefix; the downloaded content was already a public static page. A miss on one site does not remove this row from the short table.

Separate from STS `*` (credentials must first be requested and key wildcarded), filename `../` traversal across tenant directories, and Azure container-level SAS: this finding is **anonymous access allowed by bucket policy, with no credentials**.

### Storage proxy `sign key=/` (short table has a pointer)

Recognize: Business gateway proxies object storage as `/api/storage/sign` (or similar sign endpoint), and the query accepts only `key`. `key=/` or `key=.` returns S3 `ListBucketResult` XML; this is not an STS request.

Test (while logged out):

1. GET `?key=/` (then try `.`). Compare: a random key should return `NoSuchKey` / 400.
2. Copy a business prefix from the listing (such as `images/` or `editor-`) and GET through the same endpoint.
3. Check whether objects are unpublished material; don't stop at the official site's banner.

Count as success: Listing and reading the original contents of other users' unpublished objects.

False positives: Only public static assets from app-static; `filename=` returns 400, meaning no endpoint. A miss on one site does not remove this row from the short table. Separate from STS wildcard (request credentials first) and bucket policy open to anonymous users (call the bucket domain directly): this finding is **the business gateway acting as a List/Get proxy**.

### Artifact download details treat presence of uin as the login gate (short table has a pointer)

Recognize: Software / private cloud / material download center; details JSON has `DownloadURL`; frontend copies `uin`/`skey` from Cookie into the query. With an empty identity, the URL is an empty string and the page redirects to login.

Test: While logged out, GET details with a non-empty fake identity value (`uin=1`), without skey. Compare against empty uin. Range-GET the first few bytes of the returned signed URL to check for ELF/PK.

Count as success: The actual private-cloud installation package/internal deployment document. If an empty uin also produces a URL, that's completely unauthenticated access (a separate finding). Signed URL returns 403 / only a public manual → no success.

### Onboarding JS hardcodes fileKey (short table has a pointer)

Recognize: The onboarding/qualification SPA bundle has a mock or demo formData with a long encrypted `fileKey` hardcoded. The site root 301s to a new domain, but **the old host's download endpoint may still work**. Don't dismiss the mock as a placeholder, and don't treat the 301 as proof the whole site is dead.

Test (while logged out):

1. Copy `fileKey` from the new domain's chunk.
2. Call the old host's `/json/view/file/downloadFile?fileKey=` (or equivalent).
3. Compare: a random fileKey should return empty/error; the real key should return an image.

Count as success: The original license/ID image from a private bucket downloads.

False positives: Treating the 301 to the new domain as proof the site is dead; mock fileKey is actually a placeholder and random values return the same image. A miss on one site does not remove this row from the short table. Actual secret values and complete JS do not belong in this article.

### Share authorization false but media still downloads (short table has a pointer)

Recognize: Cloud recording share; authorization endpoint has `download_enable`/`view_minutes_enable`. Or the meeting JSON gateway has no `permission/auth` at all (don't stop at 404).

Test (while logged out):

1. Try both `download-multi-record-file` **and** `sign-multi-record-file` (requires `auth_share_id` + `resource_type`). International JSON gateway missing `sharing_id` returns only `2710500 invalid parameter`; include both `auth_share_id` and `sharing_id` to get an MP4 signed. Missing this can make you incorrectly conclude there's no issue. **If POST JSON says “record id cannot be empty,” don't stop; switch to GET query and put `record_id` / `auth_share_id` in the URL.**
2. If there is no authorization endpoint, don't assume there is no entry point: call `public/record-detail/download-multi-record-file` directly.
3. For minutes/timeline, try `get-full-summary` / `query-timeline` with only `record_id`; don't assume a share parameter is required.
4. A 403 from COS without Referer **is not the end**: GET again with the share page as Referer.
5. Check whether `permission/get-cfg` reveals other records in the same meeting.
6. Getting a URL without GETting the MP4 is only partial; keep following it.

Count as success: Despite authorization being false, retrieve the real MP4 (`ftypisom` + nontrivial size); or retrieve the full meeting notes with only `record_id` (not just the title).

False positives: Authorization actually blocks access; cannot download the file; only a public manual; notes contain only the title and no body. A miss on one site does not remove this row from the short table.

### Password-share response contains the plaintext extraction code (short table has a pointer)

Recognize: Share-details JSON contains `need_pwd` / `auth_level` (or similar) indicating an extraction code is required, while the auth object includes the plaintext code in `pass_word` (or similar). Or the landing HTML / `syncData` embeds the same plaintext code.

Test (while logged out):

1. Call the view endpoint with the extraction code blank. Compare: file list should be empty and the gate field should indicate a code is required.
2. Copy the plaintext code from that same response and submit it to the view endpoint. Compare: list is empty without the code; only after entering it do you get `file_id`.
3. **If plaintext `pass_word` is already in the landing HTML / `syncData`, copy it directly**; you do not need to call View CGI first.
4. Use `file_id` and the code to call the download endpoint; follow through to the actual file header (PDF/ZIP body, etc.), don't stop at JSON containing a code.

Count as success: Obtain the extraction code and download the password-protected share contents, rather than an empty-password share that directly exposes the file list.

False positives: `pass_word` is hashed and cannot be used to sign in; the empty-code View already exposes the files (that's a no-code share, not this case). A miss on one site does not remove this row from the short table.
Actual secret values and complete JS do not belong in this article.


