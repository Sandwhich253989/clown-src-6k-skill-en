# Desktop Task Folder (Permanent Requirement · Task Root Format)

> User requirement (reconfirmed 2026-08-04; **strengthened 2026-08-04: separate `js/` + assets contain only public URLs**):
> When asked to do a task, create a task root on the Desktop in the `{target}_SRC挖洞` format, with clear subfolders and rules.
> **Do not** dump reports/scripts/assets/js directly into the Desktop root; **do not** create only a flat `xxx_dig` folder and stuff reports into it as a workaround.

---

## 1. Standard Directory Format (Permanent Template)

Name the Desktop task root `{target/topic}_SRC挖洞` or another equally clear Chinese name.

**Create all of the following together when starting a task:**

```
Desktop\{任务名}_SRC挖洞\
  README.md          ← task description, directory structure, progress, resume command
  资产\              ← “publicly testable URL assets” found during research (domain/URL list)
  js\                ← JS source files extracted/downloaded from the site (for analysis)
  报告\              ← formal reports
  {短名}_dig\        ← scripts, queue, state, findings, logs, temporary probes
```

**Reference example (must match):** `Desktop\{目标}_SRC挖洞\`

| Subfolder | Put here | Do not put here |
|-----------|----------|-----------------|
| `资产/` | List of publicly testable URLs/domains (target assets discovered and retained during research: URL lists, subdomains, API entry URLs, attack surfaces from FOFA exports, etc.); **free-jump work must also include `种子队列.md`** (business names/brands/SRC domains/wholly owned domains + pending/doing/done; see dig-scope §1.4; no quantity limit) | **Do not** put extracted `.js` files here; no reports or scanning scripts |
| `js/` | JS files **extracted/downloaded** from the target site (LinkFinder/manual capture/bulk-downloaded frontend bundles), for API reverse engineering | No formal reports; do not put the “URL asset list” here |
| `报告/` | Formal report `.md` files (filename = vulnerability title) | Scripts, state, partial logs, raw JS |
| `{名}_dig/` | Dig scripts, queue/state/findings, small validation scripts, logs, temporary probes; **new tasks must include `线程必读.md`** (iteration gate; see `hunt-iter`) | Final deliverable reports; **finalized extracted JS belongs in `js/`, not piled into dig as an asset** |
| `README.md` | Structure description, progress table, resume command, retention rules; **must include one iteration-gate line** (see `hunt-iter.md`) | — |

### 1.1 Keep Assets and JS Separate (Mandatory)

| Content | Correct path |
|---------|--------------|
| Publicly testable URL/domain/entry-point list (surfaces discovered during research that can be tested further) | `资产/` |
| `.js` source files extracted from a page/CDN | `js/` |
| Temporary downloads, partial files, JS/HTML cache used for a one-time probe | May be kept temporarily in `{名}_dig/`; **move finalized extracts with analysis value to `js/`** |

**Explicitly prohibited by the user:** putting extracted JS into `资产/` again.

---

## 2. Hard Rules

1. **One task, one root directory.** Create it at the outset, before producing outputs.
2. **Four subfolders plus dig:** `资产/` + `js/` + `报告/` + `{名}_dig/` — **keep their purposes clear and do not mix contents.**
3. Fixed report path: `{task root}/报告/`; scripts must point `REPORT_DIR` there.
4. Fixed extracted-JS path: `{task root}/js/`; download/extraction scripts should output there by default.
5. Fixed URL-asset path: `{task root}/资产/`; list only publicly testable URLs/domains, not JS files.
6. Resumes, incremental work, Pass2/3/4 → **stay in the same task root**; do not create loose files elsewhere on the Desktop.
7. **Do not** put report `.md` files, queues, state, or stray py/js files in the Desktop root.
8. If the user specifies a path, use it; otherwise use this format.
9. **New tasks must include the iteration gate:** create `{名}_dig/线程必读.md` at the outset and write “For iteration, see `hunt-iter.md`.” Pin the same sentence at the top of `README.md` and, for free-jump work, `资产/种子队列.md`. Subthread deliverables must include iteration information. The detailed rule belongs only in `hunt-iter.md`.

---

## 3. When the User Says “Clean Up” (Operations)

For the cleanup policy, follow only the “Cleanup” row in `vuln-report-format.md`. This file describes what and how to delete, and what browser data must remain untouched.

Definition of “has a report”: `{task root}/报告/` exists and contains at least one formal report `.md` file. A missing or empty folder means there is no report.

### 3.1 No Reports → Remove the Entire Root

| Action | Description |
|--------|-------------|
| **Delete the entire task root** | Remove the whole `Desktop\{任务}_SRC挖洞\` tree |
| Everything inside it | Delete `资产/`, `js/`, `{名}_dig/`, `README.md`, process junk, etc. **together**; do not retain assets separately |

Applies to tasks with no findings, tasks that produced only dig/assets/js and no report, and tasks with an empty `报告/` folder.

### 3.2 Has Reports → Keep Only “Assets + Reports”

| Keep (must not delete) | Delete (may clean up) |
|------------------------|-----------------------|
| `{task root}/报告/` and every file in it | Entire `{名}_dig/` folder |
| `{task root}/资产/` and every file in it | Entire `js/` folder |
| The task root itself (keep the outer folder) | `README.md`, other loose scripts/logs/temporary files |
| | Task-related junk accidentally left in the Desktop root |

When reports exist: **do not** delete `报告/` or `资产/`; **do not** move reports/assets away and then delete the whole task root.

After cleanup:

```
Desktop\{任务}_SRC挖洞\
  资产\     ← keep
  报告\     ← keep
```

(`js/`, `*_dig/`, and README may be absent.)

### 3.3 Scope

1. If the user says “clean up” and **names a task**, handle only that task root.
2. If the user says “clean up” and means **all task folders / apply the rules across the Desktop**, process every `*_SRC挖洞` folder (and equivalent task root) on the Desktop under §3.1 / §3.2.
3. Do **not** mistake folders that are not task folders (tools, SRC documentation folders, ordinary software installers, etc.) for task roots and delete them.
4. This session’s **Desktop** `.playwright-mcp` snapshot/console **files** may be cleaned (only temporary screenshots/logs inside that directory).

### 3.3.1 Never Clean Browser Accounts, Passwords, or Configuration — Permanent (User-Specified 2026-08-04)

> The user explicitly stated: **do not delete browser-saved accounts or passwords during future cleanup.**

During cleanup, **never touch, delete, reset, or clear** any of the following, even if the user only says “clean up”:

| Protected item | Description |
|----------------|-------------|
| Accounts/passwords saved in the browser / autofill | Chrome/Edge password vaults, Login Data, Autofill |
| Browser profile directories | `User Data`, `Default`, Profile, `Login Data`, `Web Data`, entire Cookies folders |
| Profile of the real browser connected through CDP/9222 | Do not clear the whole store with `ClearBrowserCookies`, disable “save passwords,” or wipe credentials |
| Credential storage in a persistent Playwright context | Do not clear saved logins the user relies on in storage state (temporary cookie files inside task dig folders may be deleted along with dig) |

**Browser-related junk that may be cleaned (only):**

- Temporary **snapshot/console files** under `Desktop\.playwright-mcp\` (page snapshots, console exports)
- Process cookie text files and probe caches inside a task’s `{名}_dig/` (**not** browser password vaults)

**Do not** perform these actions as “cleanup”: close browser profiles, delete a Profile, apply site-wide `Clear-Site-Data`, reset a browser, clear “passwords and forms,” or globally clear storage in the 9222 browser.

### 3.4 Execution Order

1. List task roots to process (named by the user or `*_SRC挖洞` folders on the Desktop, etc.).
2. For each task root, check whether `报告/` **contains files**.
3. **No report** → use `Remove-Item` on the entire task root.
4. **Has a report** → delete only dig/js/README/other items; **lock and preserve** `报告/` + `资产/`.
5. If reports exist but there is no `资产/` folder, an empty `资产/` folder may be kept or omitted; **never touch `报告/`**.
6. Browser: only temporary snapshot/console files under Desktop `.playwright-mcp` may be cleaned; **never touch password vaults/profiles**.

### 3.5 Pre-Cleanup Checklist

- [ ] Did you first check whether `报告/` contains any files?
- [ ] If there is no report, will you delete the entire root (including assets/js/dig)?
- [ ] If a report exists, will you keep only `资产/` + `报告/` without deleting a report?
- [ ] Could you mistake a non-task folder (such as an `SRC文档` folder) for a task root?
- [ ] **Could this accidentally clear browser-saved accounts/passwords, Profile, or Login Data? If so, stop.**

---

## 4. Preflight Checklist (Before Writing Files, Delivering a Report, or Starting a Task)

- [ ] Does `Desktop\{任务}_SRC挖洞\` already exist (or an equivalent path specified by the user)?
- [ ] Does it contain `资产/`, `js/`, `报告/`, `{名}_dig/`, and `README.md`?
- [ ] Are extracted JS files only in `js/` (**not** `资产/`)?
- [ ] Does `资产/` contain only publicly testable URL/domain lists?
- [ ] Are reports written only in `报告/`?
- [ ] Are dig state and scripts only in `{名}_dig/`?
- [ ] Did anything accidentally land in the Desktop root? If so, **move it into the appropriate subfolder** (do not pretend cleanup means deleting it).

---

**In one line:** A new task = `README + 资产(public URLs) + js(extracted JS) + 报告 + *_dig`; **JS never goes in assets**. For cleanup policy, follow only `vuln-report-format`.