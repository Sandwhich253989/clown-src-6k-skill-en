# Knowledge Base Index

Practical methodology / test checklists / scenario matrix. Use alongside the workflow in `SKILL.md`.

## Conventions

- On a site, read `打穿短表.md` first; open the matching module for details once it fits. Read a short file in full; for a very long one, open the named section first and continue if needed. Do not read this whole directory for every site.
- Open a dedicated file only if `*src经验.md` exists on disk; its absence is not a blocker. Opening `SKILL.md` does not also load a group-specific journal.
- The quick matrix and “injection/SSRF/XSS/RCE” are not the limit. Complete the full vulnerability-class matrix for the site. Test the four core classes on surfaces with meaningful differences (to avoid blind spots); do not test only those four classes, and do not spray `'` at every path. With an authenticated session, authorization / logic tests are just as important as the four core classes (`dig-scope` §4.2.3).
- Convenience and capability come first; saving tokens is secondary and must not prevent opening a module.
- Search by title for techniques named in the quick matrix. For large guides with pointers, retain only the practical Chinese content + pointer sections; guides marked do-not-open / rarely-reportable have been condensed to one line. Keep technique entries; do not remove them or weaken the coverage.
- In-document links now point to real files in this directory (such as `idor-test.md`); do not change them back to `../xxx/SKILL.md`.
- If anything conflicts with `~/.grok/rules`, **follow the rules** (what to test: `src-value`; do not test CORS: `cors-vuln-report-priority`; whether to report: `vuln-report-format`).
- **Do not test `cors-test.md`; do not open `llm-security-test.md` as a jailbreak guide.** Do not use `401-403-bypass.md` to grind login HTML.
- Formal SRC reports: `~/.grok/rules/vuln-report-format.md`.

## File list

| File | Description |
|------|------|
| `打穿短表.md` | Vulnerability technique index (one-line entry / pointer; full details remain in each module) |
| `401-403-bypass.md` | **Do not open to grind login HTML** (condensed to one line); test business API 401 responses directly |
| `api-gateway-test.md` | API gateway |
| `agent-tool-exec-test.md` | Real execution of tools exposed through chat (not jailbreak, not Cloud IDE RPC) |
| `authbypass-test.md` | Authentication bypass (unauthenticated password changes / IDaaS + quick-matrix pointers; English dictionaries removed) |
| `cache-poisoning-test.md` | Cache poisoning / deception (original material + additions) |
| `clickjacking-test.md` | Do not report missing headers (condensed to one line) |
| `cloud-ide-codex-rce-chain.md` | Cloud IDE / Codex: weak credentials → RPC RCE → cluster / API key chain (pointer in quick matrix) |
| `cors-test.md` | **Do not test or open** (condensed to one line) |
| `crlf-injection-test.md` | Rarely reportable (condensed to one line) |
| `csp-bypass-test.md` | Rarely reportable (condensed to one line); for XSS, see `xss-test.md` |
| `csrf-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `csv-formula-injection-test.md` | Rarely reportable (condensed to one line) |
| `dangling-markup-test.md` | Rarely reportable (condensed to one line) |
| `dependency-confusion-test.md` | Rarely reportable (condensed to one line) |
| `deserialization-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `dns-rebinding-test.md` | Rarely reportable (condensed to one line); for SSRF, see `ssrf-test.md` |
| `el-injection-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `email-header-injection-test.md` | Rarely reportable (condensed to one line) |
| `file-upload-test.md` | File upload (pointers to STS / bucket listing / share authorization, etc.) |
| `ghost-bits-cast-test.md` | Ghost Bits principles + common characters + formula; two byte-by-byte tables removed |
| `graphql-test.md` | GraphQL (original material + additions) |
| `hpp-test.md` | Rarely reportable (condensed to one line) |
| `http-host-header-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `http-smuggling-test.md` | Request smuggling (original material + additions) |
| `http2-attacks-test.md` | Rarely reportable (condensed to one line); for smuggling, see `http-smuggling-test.md` |
| `idor-test.md` | Authorization (main Chinese content + quick-matrix pointers) |
| `info-leak-test.md` | Information disclosure |
| `injection-test.md` | Injection (OR+total / email subscription iframe / SSTI probes; English encyclopedia material removed) |
| `insecure-scm-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `jndi-injection-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `js-reverse-guide.md` | JavaScript reverse engineering |
| `llm-security-test.md` | **Do not open as a jailbreak guide** (condensed to one line); for chat tools, see `agent-tool-exec-test.md` |
| `logic-test.md` | Business logic (payments / flows + merchant promotion binding) |
| `oauth-jwt-test.md` | OAuth/JWT/SAML/OIDC (original material + multi-source additions) |
| `open-redirect-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `path-traversal-lfi-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `prototype-pollution-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `race-condition-test.md` | Race conditions (original material + additions) |
| `recon-methodology.md` | Recon methodology |
| `ssrf-test.md` | SSRF (IMDS path differences / GOPROXY / object-storage fetches) |
| `subdomain-takeover-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `type-juggling-test.md` | Specialized knowledge (imported or merged from hack-skills) |
| `waf-bypass.md` | WAF bypass |
| `websocket-test.md` | WebSocket (original material + additions) |
| `xslt-injection-test.md` | Rarely reportable (condensed to one line) |
| `xss-test.md` | XSS (Chinese-language opening / rare events / XSS→RCE / custom protocols) |
| `xxe-test.md` | Specialized knowledge (imported or merged from hack-skills) |

**Total: 48 knowledge files** (excluding this README). SRC report formatting is not covered here; see `~/.grok/rules/vuln-report-format.md`. Severity is governed only by that format; this knowledge base does not assign severity.