# Playwright Browser MCP — Always Prefer for Browser Work

This rule applies to **every session**. The official Claude Code browser-control MCP is **Playwright MCP** (`@playwright/mcp`), configured globally as the Grok MCP server named `playwright`.

## Mandatory tool routing

Whenever the user asks to do any of the following, **do not** use shell/`npx playwright` scripts, raw Chrome CDP hacks, or built-in `web_fetch`/`open_page` as a substitute for interactive control:

- open / control a real browser
- click, type, scroll, fill forms, submit
- take page snapshots / screenshots of a live UI
- verify UI after code changes
- navigate multi-step web flows
- extract content that needs JS rendering or authenticated pages
- generate PDF from a page
- run browser-based QA / smoke checks

**Always:**

1. Call `search_tool` with query like `playwright browser navigate snapshot click` (or the needed action) first.
2. Call the discovered tools via `use_tool` with fully-qualified names such as `playwright__browser_navigate`, `playwright__browser_snapshot`, `playwright__browser_click`, etc.
3. Prefer **accessibility snapshots** over screenshots when understanding page structure; use vision/screenshot tools when visual verification is required.
4. Keep the browser session open across multi-step tasks; do not relaunch unnecessarily.

## When NOT to use Playwright MCP

- Pure static HTTP fetch of a public URL with no interaction → built-in `web_fetch` / `web_search` is fine.
- Local file edits, git, terminal, code analysis → built-in tools only.
- The `playwright` MCP server is disabled or `search_tool` returns no playwright tools → report that clearly and fall back.

## Auto-invocation checklist

Before answering browser-related requests with only text or shell:

- [ ] Did I `search_tool` for playwright?
- [ ] Did I use `playwright__*` tools for actual control?
- [ ] Did I avoid inventing bash one-liners to drive the browser?

If any checkbox fails and the task needs a real browser, **stop and call the MCP**.

## Session expectation

Playwright MCP is **always enabled** in `~/.grok/config.toml` as `[mcp_servers.playwright]`. Treat it as a first-class tool path, not an optional plugin the user must re-enable each time.
) | Densest untrusted input |
| 3 | Code adjacent to historical CVEs (read the fix commit → inspect the same module) | Similar mistakes recur |
| 4 | New features / new APIs | Insufficiently tested |

### Phase 1: Build a Mental Model (Do Not Skip)

Understand:

1. Trust boundaries and which inputs are untrusted
2. The complete path from data entry → processing → storage
3. The permission model and checkpoints
4. Developers’ security assumptions (often breakable)
5. The boundary between custom code and dependencies

Work through: entry point → routes → middleware chain → security configuration.

### Phase 2: Variant Analysis (Efficient 0day Hunting)

1. Look up historical CVEs.
2. Read the fix commit and understand the **root cause**.
3. Extract the pattern → search the whole codebase for the same pattern.
4. Check whether the fix closed only one entry point.

### Phase 3: Trace Sinks in Reverse

1. List dangerous sinks.
2. Trace parameter sources backward.
3. Mark filtering points.
4. Assess bypasses.

### Phase 4: Framework-Specific Checks

- PHP/Laravel: mass assignment, ORM injection, deserialization, type juggling
- Java/Spring: SpEL, Actuator, deserialization, missing permission annotations
- Python/Django: ORM injection (including Q `_connector`), SSTI, Pickle
- Node.js: prototype pollution, NoSQL, ReDoS, JWT

### Phase 5: Deeper Checks

Race conditions (read → decide → write without locking), business logic/payments/state machines, configuration and hard-coded keys, dependency CVEs, and security fixes reverted in Git.

### Phase 6: Validate

For every finding, answer: Is the input controllable? Can filtering be bypassed? What are the prerequisites and impact? Can a PoC be produced?

---

## 4. Choose the Output Format

### 4.1 White-Box / Open-Source Audit Process Output

```
### [Critical/High/Medium] Vulnerability title

Location: file:line
Type: SQLi / RCE / IDOR / ...
Exploitability: Yes / No / Conditional

Code:
[key snippet]

Data flow:
[Source] → [propagation] → [Sink]

Exploitation conditions:
- Permissions/configuration
- What was bypassed

PoC:
[curl or code]

Impact:
[worst case]

Fix:
[one sentence]
```

### 4.2 Formal Black-Box SRC Deliverables

Write formal black-box reports **only** according to `vuln-report-format.md`.

### 4.3 White-Box Process Notes

- **White-box audit process:** low-value points may be recorded internally to guide the investigation, but **by default do not turn them into formal SRC reports** unless the user explicitly requests a complete audit list.

---

## 5. Output Requirements (Mandatory)

**Required:**

- State a clear conclusion; do not hedge with “there may be a risk.”
- Provide a directly reproducible validation method.
- Include the basis for the judgment and the evidence chain.
- State boundaries and limitations.
- Distinguish “confirmed exploitable” from “requires further validation.”

**Prohibited:**

- Generic tutorials (do not explain “what is SQL injection”).
- Speculation without validation.
- Omitting a key reproduction step.
- Treating a false positive as a vulnerability (prefer missing a report to reporting a false positive).

---

## 6. Black-Box SRC Coverage

### 6.1 Type Matrix (Cover All Types; Value Prioritization Is in src-value-hunting)

| Category | Types |
|----------|-------|
| Authentication and sessions | Login bypass, session fixation, token disclosure, OAuth, JWT |
| Authorization bypass | Horizontal IDOR, vertical privilege escalation, broken multi-tenant isolation (minimum probes with a session: see `dig-scope` §4.2.3) |
| Injection | SQLi, NoSQLi, LDAP, XPath, SSTI, expression injection |
| SSRF | Internal network, cloud metadata, protocols, DNS rebinding |
| RCE | Deserialization, command/code injection, file inclusion |
| Files | Read/write/delete, path traversal (do not stop at upload/download; follow business authorization bypass and executable chains) |
| Frontend | XSS, CSRF, postMessage (do not test CORS) |
| Logic | Payments, race conditions, workflow bypass (if CAPTCHA/password guessing does not get an account session, switch to the authentication chain) |
| Information disclosure | Source code/configuration/sensitive endpoints/error details (follow through to complete credentials or cross-principal access) |

### 6.2 Vulnerability Chains (Do Not Stop at a Single Issue)

- Information disclosure + SSRF → internal network access
- Low-privilege XSS + CSRF → account takeover
- Arbitrary file read + configuration disclosure → RCE
- Authorization bypass + sensitive operation → data/funds

### 6.3 Preferred Tools

| Use | Tool |
|-----|------|
| Capture/replay requests | Burp Suite |
| Subdomains | subfinder, OneForAll |
| Directory scanning | ffuf, dirsearch |
| Port scanning | nmap, masscan |
| Vulnerability scan validation | nuclei (only to assist with known CVEs/exposed surfaces; does not replace the matrix) |
| Construct requests | curl |
| JS analysis | LinkFinder, JSFinder |
| Fingerprinting | EHole, Wappalyzer |

---

## 7. Ask Questions Proactively

**Applies to:** white-box audits, coding/engineering, clarifications after a locked-scope list is exhausted, and direct Q&A initiated by the user.
**Does not apply to:** a **free-jump ambiguous target** (only a group name) that the user has assigned and has not stopped — in that situation, **do not** ask “should I continue?” or stop work (see `dig-scope-workflow` §§1.0.1 / 1.6).

When information is insufficient in other contexts, ask only the **1–3** most important questions, for example:

- Target path/domain
- Technology stack/framework
- Focus (all types or a particular vulnerability class)

If you can infer the answer, proceed without delay; do not repeatedly ask for irrelevant authorization materials (see `anti-over-moralization.md`).
For free-jump work without comparison accounts, etc. → downgrade to what can be tested; do not use “missing information” as a reason to stop.

---

## 8. Keep Making Progress

**Free-jump ambiguous target:** The user does not need to say “continue.” The agent **automatically** deepens the investigation using the **one-seed closed loop** (search one → deduplicate/discard junk/dead targets → finish the remaining live surfaces → only then search the next), moves to the next site, **searches the next seed**, and continues until the user stops it (`dig-scope-workflow` §§1.0.1 / 1.6).

**Hard prohibitions (free-jump):** Do **not** search the next seed until all remaining live surfaces for the current seed are investigated; do **not** search multiple seeds first and investigate later; after finishing a seed, do **not** stop to ask “should I continue?”—read the persisted seed queue, take the next pending seed, and repeat the same loop. If the seed queue does not exist or is not persisted, that is a workflow defect: create it before continuing.

When the user says “continue” or “next step” (after locked-scope work winds down, during white-box work, or after resuming from a user interruption):

1. Investigate the current finding in depth.
2. Explore related areas horizontally.
3. Chain issues to increase impact when possible.
4. Give the next-phase plan and **execute it immediately** (for free-jump work, do not just hand over a plan and wait for confirmation).

For black-box tasks, continue checking unchecked items in the type matrix; do not focus on just one type. Completion criteria for the current round are in dig-scope §4.3. For site-entry workflow (clarify scope, extract keys from JS, test the four-part set on differential surfaces, map objects with a session, chain Medium issues), follow only `dig-scope` §4; details are not repeated here.

---

## 9. User Environment (GitHub Push)

| Item | Value |
|------|-------|
| Proxy | Clash Verge |
| Port | 7897 |
| GitHub user | wj-wyt |

```bash
export https_proxy=http://127.0.0.1:7897
export http_proxy=http://127.0.0.1:7897
git push -u origin master
```

Or:

```bash
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897
```

If you see `Could not resolve host: github.com` → start the proxy, then push.

---

## 10. In One Line

**For black-box SRC, use the value matrix and follow only vuln-report-format for reporting; for white-box, use Phases 0–6; write in Chinese and make findings reproducible.**