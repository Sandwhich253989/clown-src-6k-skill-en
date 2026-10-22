> Whether to report is governed only by `~/.grok/rules/vuln-report-format.md`. This guide covers versions / frameworks / health / unauthenticated shells / fingerprints → continue looking for complete credentials, keys, and cross-tenant business data. If you find credentials / cloud keys / tokens, compare with a fake value, verify the key can reveal an identity or list, then use a read-only test that does not affect production before reporting; do not report phone-number encryption keys.
> Structure: this guide is short enough to read in full. Test middleware ports only when observed (§5); do not scan ports at every site first.

# Information Disclosure Testing Guide

## 1. Common disclosure points

### 1.1 Excessive fields in API responses

```bash
# Check whether login / user-information APIs return sensitive fields
curl "https://target.com/api/user/info" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Fields to check:
# password, passwordHash, salt
# idCard, bankCard, realName
# phone (unmasked), email
# secretKey, apiKey, token
# internal fields: is_admin, role, balance_internal
```

### 1.2 Error-message disclosure

```bash
# Send a malformed request to trigger an error
curl "https://target.com/api/user?id='"
curl "https://target.com/api/user?id[]=1"

# Check the response for:
# Database errors (exposed SQL statements)
# Stack traces (exposed code paths)
# Internal IP addresses
# Framework / version information
```

---

## 2. File / directory disclosure

### 2.1 Common sensitive paths

```bash
# Development leftovers
/.git/config
/.git/HEAD
/.svn/entries
/.DS_Store
/.env
/.env.local
/.env.production
/config.php
/config.yml
/application.properties
/application.yml
/web.config

# Backup files
/index.php.bak
/index.php~
/backup.zip
/backup.tar.gz
/www.zip
/site.tar.gz
/db.sql
/database.sql

# API documentation (may expose endpoint lists)
/swagger-ui.html
/api-docs
/v2/api-docs
/swagger.json
/openapi.json
/doc.html
/redoc

# Monitoring / operations endpoints
/actuator
/actuator/env
/actuator/mappings
/actuator/beans
/metrics
/health
/info
```

### 2.2 Automated scanning

```bash
# Check paths in bulk with ffuf
ffuf -u "https://target.com/FUZZ" \
  -w sensitive_paths.txt \
  -mc 200,301,302,403 \
  -t 30 -o leaks.json -of json

# Filter 403 responses from ffuf results (content may exist behind the block, so investigate further)
cat leaks.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for r in data['results']:
    print(f\"{r['status']} {r['url']} ({r['length']} bytes)\")
"
```

---

## 3. Information disclosure in JavaScript files

```bash
# Extract sensitive information from JavaScript files
# Download all JavaScript files and search:
grep -rE "(apiKey|api_key|secret|password|token|ak|sk)\s*[=:]\s*['\"][^'\"]{8,}" *.js

# AK/SK disclosure (cloud-service credentials)
grep -E "AKID[A-Za-z0-9]{16,}" *.js       # Cloud-provider AK prefix
grep -E "LTAI[A-Za-z0-9]{16,}" *.js       # Cloud-provider AK prefix
grep -E "AKIA[A-Za-z0-9]{16,}" *.js       # AWS
grep -E "[a-zA-Z0-9+/]{40}=" *.js         # Possible Base64 key

# GitHub PAT (documentation / community sites often bundle the key used by contributor widgets)
grep -oE "ghp_[A-Za-z0-9]{20,}" *.js
grep -oE "github_pat_[A-Za-z0-9_]{20,}" *.js

```

### 3.1 GitHub PAT in documentation sites

Open-source documentation / community front ends often bundle `ghp_` / `github_pat_` tokens to fetch GitHub organizations, contributors, or stars.

Extract it from JavaScript without logging in:

```bash
curl -s "https://api.github.com/user" -H "Authorization: token $PAT"
# Control: without the token, the request should return 401
curl -s "https://api.github.com/user/repos?affiliation=owner&per_page=5" -H "Authorization: token $PAT"
```

Confirmed when `/user` returns a real login / name and `permissions.admin=true` for that account repository (the token can act as that GitHub user). Even if an official organization repository grants only pull access, include admin access to personal repositories in the impact.

False positives: the key has been revoked; `/user` returns 401; it is only a GitHub App installation token that reads a public organization. Put actual secret values only in the formal report, not here.

### Permanent object-storage key hidden with XOR in viewer JavaScript (quick-matrix pointer)

Recognize: landing-page / viewer JavaScript includes `usePrivateCode` (or similar), and `COS_TOKEN` SECRET_ID/SECRET_KEY are not plaintext AKIDs. The first 16 characters form the repeating XOR key; decode the following hex string to obtain the permanent AK/SK. Grepping only for `AKID` will miss it.

Test without logging in: after decoding, sign a request to the cloud STS API **GetCallerIdentity**, then the cloud CAM API **GetUserAppId**. Do not stop if ListBuckets returns 403.

Confirmed when you obtain AccountId/Uin/AppId. This is a long-lived key, not a temporary token that expires within minutes.

False positives: the decoded value produces AuthFailure from the cloud API; `exampleValue` decodes to the `hello_world` placeholder. Put actual secret values only in the formal report, not here. Do not remove this quick-matrix entry because one site did not match.

### CI repository key hard-coded in admin-console JavaScript (quick-matrix pointer)

Recognize: admin-console JavaScript hard-codes a CI `pipelineId` + base64 `auth`, and an unauthenticated trigger endpoint exists.

Test without logging in: copy the auth value and call the Git-hosting open API `DescribeMyDepots`; the current-site trigger is only a control.

Confirmed when it lists another team private repository name / HttpsUrl / ProjectId, or the key scope includes `depot_read` and the open API accepts it.

False positives: the key expired; the repository is public; only the trigger responds that the product is offline, with no proof the key can still list private repositories. Put actual secret values only in the formal report, not here. Do not remove this quick-matrix entry because one site did not match.

### Real identity-document sample in a documentation chunk (quick-matrix pointer)

Recognize: an open-platform documentation center; the page document / category endpoint returns 401; the entry script dynamically imports a `technical-document` DocPage chunk (or similar). This is not the “AppSecret hard-coded in debug documentation” item.

Test without logging in: do not stop at a 401 from the documentation API. Follow the homepage script → DocPage `import()` → documentation chunk. Extract identity numbers, phone numbers, and names from sample requests / responses. Continue only if the identity-number checksum is valid.

Confirmed when a checksum-valid identity number matches the person name / phone number.

False positives: placeholder names / numbers (e.g. Zhang San / 110101); fabricated numbers with an invalid checksum; an empty `appliIdNo` template; public product manuals with no sample messages. Do not remove this quick-matrix entry because one site did not match. Put actual identity values only in the formal report.

### Connection configuration in an anonymously downloadable signed bundle (quick-matrix pointer)

Recognize: the enterprise software center opens to a login page, but a signed object-storage download endpoint needs no Cookie; or the same site serves `/download/` + a software filename without a Cookie. The archive contains a Navicat `ncx` file (connection name, Host, account, `SavePassword` ciphertext). If the homepage redirects with 302 to a login wall, **an inline signing function in the 302 response body also counts as an entry point**; do not inspect only the login landing page.

Test without logging in: obtain the signed URL from the signing endpoint, then use Range GET to fetch the actual zip. **Do not stop if the signing endpoint is missing or requires login:** use a filename from the 302 body or homepage software list and directly request `GET /download/{filename}` on the same site (a random filename often redirects to the homepage; the correct one returns 200). Extract `*.ncx`. Decrypt Password with the Navicat 12 fixed key `libcckeylibcckey` and IV `libcciv libcciv` using AES-128-CBC, then remove PKCS7 padding.

Confirmed when the internal database Host, account, and plaintext password are recovered. Put actual secret values only in the formal report.

False positives: the archive contains only the public client and no connection configuration; the ciphertext cannot be decrypted. Do not remove this quick-matrix entry because one site did not match.

### Unauthorized user key on a distributed-file master (quick-matrix pointer)

Recognize: public HTTP `/version` returns `"Model":"master"` (a distributed-file cluster). Do not log in to the admin endpoint.

Test without logging in:

1. Call `GET /user/list` to obtain `access_key` + `secret_key`
2. Control: a fake 16-character ak sent to `GET /user/akInfo?ak=` should return `access key not exists`; a real ak reveals `user_id` / user_type
3. Call `GET /admin/getVol?name=` to inspect another user business volume (Owner / InodeCount)

Confirmed when the complete AK/SK pair reveals an identity / another user business volume.

False positives: only a version / cluster name appears with no key; the list is empty; fake and real keys return the same message; only your own empty test volume appears. Put actual secret values only in the formal report, not here. Do not remove this quick-matrix entry because one site did not match.

```bash
# Internal address disclosure
grep -oE "192\.168\.[0-9]{1,3}\.[0-9]{1,3}" *.js
grep -oE "10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" *.js
grep -oE "172\.(1[6-9]|2[0-9]|3[01])\.[0-9]{1,3}\.[0-9]{1,3}" *.js
```

---

## 4. Special API disclosures

### 4.1 Pagination limits can be bypassed

```bash
# Use a large page size to retrieve all data
?page=99999&pageSize=1000
?offset=0&limit=99999

# Export endpoint has no quantity limit
/api/export/users?format=csv
/api/export/orders?startTime=2020-01-01&endTime=2026-01-01
```

### 4.2 Search endpoint wildcards

```bash
# Use wildcard search to retrieve all data
?keyword=%         # URL-encoded %, SQL LIKE wildcard
?keyword=*
?keyword=.         # Some systems
?q=               # Empty search returns all results
```

### 4.3 GraphQL introspection

```bash
# GraphQL endpoint exposes the schema
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { types { name fields { name } } } }"}'

# A complete schema response may reveal unpublished endpoints and fields
```

---

## 5. How to test middleware ports when observed

> **Test only when observed.** Do not start by running nmap against every port on every site. If FOFA / the site / response headers expose these ports or consoles, follow the table. Record confirmed findings using `vuln-report-format`; if you find only a health / version shell, continue checking for credentials and do not stop at PONG.

| What you see | What to test | What confirms a finding | False positive / next step ||
|--------|------|------------|-------------|
| `:6379` Redis, unauthenticated `PING`→`PONG` | Write a webshell / crontab / `authorized_keys` (the directory must be a real web root or cron directory). For SSRF to this endpoint, use `gopher://`; see `ssrf-test.md` | A command runs or business data is read | `requirepass` is enabled; `CONFIG` is renamed; only `PING` works ||
| `:873` anonymous rsync `rsync host::` lists modules | Download business files; if a module is writable, check whether cron can be written | Recover keys / business data | Empty module; read-only and contains only public static files ||
| `:9000` PHP-FPM/FastCGI directly exposed | Send a FastCGI request to an existing `.php` (`PHP_VALUE=auto_prepend_file=php://input`). For SSRF, use gopherus fastcgi | The specified PHP file executes | Unix socket only; public port 9000 is not FPM ||
| `:8009` AJP | Use Ghostcat to read `/WEB-INF/web.xml`. See `path-traversal-lfi-test.md` §21 | Read web.xml / class files | `secretRequired`; port is not AJP ||
| `:8088` Hadoop YARN UI | `POST /ws/v1/cluster/apps/new-application`, then submit an app with `commands` | Your command runs in the cluster | UI requires login; submission returns 401 ||
| `:2375` Docker API | `GET /v1.24/containers/json`; if container creation is allowed, mount `/:/host` | List containers or read a host file | TLS 2376 requires a client certificate; only the version is exposed ||
| `/h2-console` | JNDI in the JDBC URL or `CREATE ALIAS`. See “H2 Console” in `jndi-injection-test.md` | A command runs | Only accessible locally ||

Directly testing a public service and reaching an internal service through SSRF use the same sinks; only the entry point differs. If these ports are not observed, do not scan for them just for this table.
