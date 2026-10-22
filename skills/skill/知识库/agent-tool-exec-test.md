# Real Tool Execution Through the Chat Interface
> Quick-matrix pointer. Look for a pattern where the identity endpoint blocks access but the chat endpoint still accepts requests, and the tool list contains a tool that can run commands.
> **This is not** jailbreak / prompt testing (do not open `llm-security-test.md` as the starting point).
> **This is not** weak credentials on a Cloud IDE + `command/exec` RPC (see `cloud-ide-codex-rce-chain.md`).
> For unauthorized history reads, see “Unauthorized Reads of Other Users’ Assistant Tasks” in `idor-test.md` (details are only in that file).

> Whether to report is governed only by `vuln-report-format.md`. Commands should produce only a harmless marker / `id`.

## What to look for

Any of these patterns:
1. An identity / whoami / profile endpoint returns unauthenticated, 401, or `unauthenticated`, while the **chat endpoint** (`/chat` / `createTask`, etc.) in the same front end still accepts requests without a Cookie.

2. **Test even without a whoami comparison:** a public page can POST to create a session, with a body that needs only `message` (or equivalent).

Also, JavaScript or the tool list shows a command-execution tool (common names include `bash` / `shell` / `code_interpreter` / `python` / `execute`; the list is **not exhaustive**—identify what executes).

Common front ends include helix-assistant, cloud-h5, and cloud assistants with tool streams. Recognize clones by behavior; do not rely on product names.

## What to test (unauthenticated)
1. If an identity endpoint exists, verify that it blocks access. If there is no identity endpoint, do not assume there is no vulnerability.
2. POST to the chat endpoint and ask the model to run `id` using **that tool** (or `echo marker && id`), or use Python to compute an MD5 value that you can also compute locally. Do not merely say “please execute” without naming the tool. `id` proves only that a command ran; **continue with the checks below**.

3. Inspect the event stream / tool response. **Do not stop when SSE has only deltas and no `toolName`:** compare a returned 32-character hex value with local hashlib; or ask it to list the working directory and check for `AGENT.md` / `IDENTITY.md` / `SOUL.md` that never appeared in the prompt.

4. If there is a `fileUrls` / attachment URL parameter, provide an external-site URL and check whether the server fetches its title / ICP into the conversation.


5. After command execution is established, check these three items (test only when there is an entry point; otherwise record why): the SRC-provided internal validation flag; cloud metadata / temporary keys; and the sandbox’s other-party business content (chat history, tickets, key files). A public homepage does not prove access to an internal network.

Repeat the same test with an authenticated session: even after identity checks pass, does the chat tool still accept a command you supplied?

## What counts as a finding

Any of the following in stdout / SSE:
- The SRC validation flag (such as `ssrf-`)
- Cloud credentials (temporary token / permanent AKSK) that reveal the account
- Another party’s business content (not a marker you just submitted yourself)

A sandbox `uid=`, a locally reproducible MD5, or a sandbox filename never shown in the prompt proves only that a command ran; **it is not a confirmed finding**.

## False positives
- The model only claims it executed something, the number does not match the local result, or the sandbox filename was already in the prompt
- The sandbox rejects commands or the tool list is empty
- The chat endpoint also requires login and uses the same gate as the identity endpoint
- Prompt jailbreak only, without tool execution (that is not this test)
- Only curling a public site (Baidu homepage / ICP number) as proof of internal network access

## Stop conditions
If there is no such endpoint, mark this test N/A and return to ID substitution / password-change checks / the other core tests.
If the endpoint only returns a sandbox `uid=` and you cannot obtain a flag / cloud key / another party’s content, the issue is disproved; stop rather than grinding until root access.
Test **only the current site** when you identify this pattern; do not run FOFA across the internet for assistants with the same appearance.
