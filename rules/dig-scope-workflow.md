# Vulnerability-Hunting Scope · Locked Scope / Free-Jump (Permanent)

> **Always enhance; never impose a hard ceiling** (same spirit as `skill-as-boost`).
> **The user should not have to wait a long time without a new report to notice idling** — the agent must self-check and correct course; do not use queues/logs to look busy.
> The **single-site type matrix** is consistent with `src-value-hunting.md`; this file does not narrow the types.
> **Site-entry workflow** (clarify scope §4.0 / extract keys §4.1 / response clues §4.1.3 / differential surfaces §4.2.1 / object graph with a session §4.2.3 / extend Medium issues / fat or thin): follow Section 4 of this file; probe density by type follows `src-value` §3.
> **Free-jump: keep investigating until the user explicitly stops you; do not ask “should I continue?” midway.**
> **One-seed closed loop (2026-08-16): every seed follows “search one → deduplicate/discard junk/dead targets → finish the remaining live surfaces → only then search the next”; never search multiple seeds first.**

---

## 0. Determine the Mode First

| User instruction | Mode | Summary of behavior |
|------------------|------|--------------------|
| Ambiguous target (only a group name, no URL list), no fixed URL list | **Free-jump** | Prepare as many seeds as reasonably possible → persist the queue → **one-seed closed loop (search one → deduplicate/discard junk/dead targets → finish the remaining live surfaces → only then search the next)**; **never search all seeds first**; when resuming an old folder, skip seeds/hosts already searched or tested; when all are done, search random additions or rescan for new hosts (§1). |
| Fixed site / fixed URL / documentation URL / URL list | **Locked scope** | **Do not proactively expand scope with FOFA**; test multiple hosts in the asset cluster, subdomains reached through business flows, and multiple paths on the same host. |

Principle: **no specified surface = free-jump; a specified site/document/list = locked scope.** The instruction for this task takes precedence (if the user says “lock scope” / “don’t jump yet,” that overrides the default).

### 0.1 Locked-Scope Definition (Avoid Locking Too Tightly · User 2026-08-06)

The lock means **“do not quietly switch to free-jumping across the whole group with FOFA,”** not “pin testing to the literal entry URL.”

| Lock (prohibited) | Do not lock too tightly (must investigate) |
|-------------------|--------------------------------------------|
| Search/probe the whole group or unrelated business divisions with FOFA before visiting the site | Test all hosts included in/loaded by the entry point |
| Jump to a tangential domain unrelated to the current assets on your own | Follow subdomain/gateway/API domain changes caused by login or clicking through a feature |
| Use documentation as a free-jump seed to expand across the group | Expand to all paths/endpoints on the same host; include **business hosts for the same product** exposed by JS/configuration |

**Inside locked scope:** the user’s entry host; business hosts in pages/JS/configuration/responses; subdomains/gateways reached naturally through a business flow; multiple hosts in the same asset cluster.
**Out of scope:** FOFA for every brand before visiting the site; attacking an unrelated product line merely because it “seems to be from the same company.”
**Do not misread this:** locked scope does not mean testing only a text-identical hostname; it does not mean testing only the one path in the documentation; it does not mean stopping whenever a subdomain changes. Documentation defaults to locked scope; only when the user separately asks to hunt a group is the task free-jump.

**End of locked scope:** once the current asset cluster’s matrix is complete, you may stop or give a brief report; this is not an unlimited free-jump.

**Order for a locked-scope list (hard rule · do not grind on the first login wall while “important” assets wait):**

| Do | Do not |
|----|--------|
| If the user marks assets “important/priority,” **test that group first** | Treat “one, two, three” as a requirement to finish the first section from top to bottom |
| “Test all” means each group will be covered in turn; keep only 2–3 representatives of the same shell (§3.3) | Spend ages on CAPTCHA/default credentials at the first login wall before moving to CDP / BI / LIMS |
| For identical login shells, do not retest the login form. If the representative site was tested under §4.2.2, compare siblings at a glance (is there a new reset/rebind/ticket-replacement path?). If there is no new entry point, do not start a full matrix again. Still compare `jump` / `service=` / `moduleId` to see whether they represent another business surface (if so, test it). | Run a full matrix again on every host with the same shell; skip the entire task just because the login shell is the only difference |

Do not interpret §1.0.1’s “do not pick the best surfaces and discard other live URLs” (the one-seed free-jump rule) as requiring locked-scope work to follow the source list’s exact order for every full matrix.

### 0.2 Collaboration Ruling (Overrides researcher’s “wait for continue”)

| Context | Behavior |
|---------|----------|
| Free-jump ambiguous target | **Do not stop or ask whether to continue**; if comparison accounts are missing, downgrade to what can be tested and record it as you go. |
| Locked scope / white-box / coding | Once the list is exhausted or before running, report progress as specified by the relevant files. |
| A unique technical prerequisite is truly missing and cannot be worked around | Give a very brief blocker update, then keep investigating surfaces that remain testable; **do not use a question as a reason to stop.** |

---

## 1. Free-Jump

### 1.0 Overview (Must Follow at Startup)

```
Ambiguous target
  ├─ Does Desktop have an uncleared {目标}_SRC挖洞 folder?
  │     Yes → Resume: read seed queue + covered_hosts + reports/ + dig STATE
  │            Skip done/cold seeds and covered hosts; start with pending
  │     No  → Start new: create task root; prepare and persist as many seeds as possible; order may be shuffled
  │
  ├─ Is there a matching `*src经验.md` in the knowledge base? Open it only if present; its absence is not a gap
  │     Yes → Open it alongside the short table, including in a new window. A specialist article gives rules/dead-end clusters; it is not this site’s asset list and does not replace the matrix
  │
  ├─ Take seed S (prefer pending; live-filter P2 registered seeds first)
  │     → Search only this S (FOFA/supplemental pool only for S; page through this seed’s results as needed)
  │     → URLs found: deduplicate → discard junk → discard dead targets → 100%-ownership gate
  │     → Visit every remaining live surface and run the full matrix (§4) → save successful findings to `报告/` under the format
  │     → Add promising search domain R back to the queue only; do not search R before investigating all remaining live surfaces for this seed (§1.1.1)
  │     → Finish every remaining live surface for this seed (or mark all junk/dead) → set done → only then search the next pending seed (do not ask to continue)
  │
  └─ Are all queue entries done / is coverage nearly complete, and has the user not stopped you?
        → Search with randomly added keywords OR rescan old seeds only for hosts newly discovered relative to covered hosts
        → When company registry/SRC information becomes available, incrementally add fourth-tier entries and new business names
```


### 1.0.1 One-Seed Closed Loop (Hard Rule · User 2026-08-16)

> **Every seed follows this process without exception.**
> **Do not** search every seed in the queue first and investigate afterward.

```
Take one seed S
  → Search only this S (FOFA/supplemental pool only for S; page through results for this seed as needed)
  → URLs found: deduplicate → discard junk (§3.1) → discard dead targets → ownership gate (§3.2.0)
  → Finish investigating all remaining live surfaces (each site meets §4.3, or is recorded as junk/dead/skipped as an identical shell/already covered)
  → Only then mark this seed done → only then may you search the next seed
```

**Prohibited:**

| Prohibition | Explanation |
|-------------|-------------|
| Search multiple seeds in a row / search them all first | Do not run FOFA on the entire queue, stockpile a large pool, and only then return to investigate. |
| Change seeds while live surfaces remain | If live URLs remain untested after cleaning this seed, do not open the next seed or run FOFA on an R domain that was added back to the queue. |
| Misread “batch FOFA” | “Batch” means paging through results for **this one seed**, not searching many seeds in a row. |

**What “finished investigating” means / does not mean:**

| It means | It does not mean |
|----------|-----------------|
| Every `host:port` that remains after cleaning and should be tested has met §4.3 (endpoint list + matrix + four-part set on differential surfaces + §4.2.2; add §4.2.3 when a session exists; immediately disprove thin surfaces), or is recorded as junk/dead/represented by a tested identical shell/obvious same-gate differential (§4.1.2)/globally covered | Exhaustively scrape every historical path in existence; send `'` once to 80 empty shells |
| A dead shell, 403 idle loop, or identical shell found after visiting a site is recorded and skipped | Run a matrix on two or three “best” surfaces, abandon remaining live URLs, and move to the next seed |
| If the pool is empty after cleaning (all junk/dead) → immediately mark the seed done and search the next one | Treat getting the FOFA result count for a seed as “searched, so it’s okay to switch seeds” |

**Discard dead targets (required after searching this seed; remove them, then investigate):**
No response / timeout / parked page / no business response = dead; do not test it.
401 / 403 / login wall / admin-console challenge page = **live**; do not discard it as a dead site.
**Live does not mean grind on the login form.** See §4.1.1: first find business APIs / business hosts reached after redirects; without a session, the main work is unauthenticated testing. If session issuance/reset/rebinding/ticket replacement has an entry point, follow §4.2.2; otherwise mark it N/A and do not grind on the form. Identical shells: see §3.3.

**Still allowed inside this seed’s closed loop:**

- Page through FOFA results for this seed until its result set has been collected.
- Persist promising search domain R as pending (§1.1.1), but **do not search R** until all remaining live surfaces for this seed are investigated.
- If the quota is exhausted, stop searching and investigate the remaining assets already in this seed’s pool.
### 1.1 Prepare as Many Seeds as Possible (Hard Rule · User 2026-08-06)

**Prohibited:** Search only the user’s exact original term once; set a “maximum N seeds”; treat one batch as a complete queue; never replenish the queue if Qichacha is unavailable; **search multiple seeds first and investigate later.**
**Required:** After tiering, strictly follow the **one-seed closed loop (§1.0.1):** search S → clean (deduplicate, discard junk/dead targets, ownership gate) → investigate every remaining live surface → only then search the next seed. A single site only needs §4.3 to be complete before moving to the next host within this seed; “finished” does not mean exhaustively checking every path in existence.

| Dimension | Requirement |
|-----------|-------------|
| **Business names / products / sub-brands** | List as many as possible (payments/finance/cloud/blockchain/mobility/digital technology/international/internal divisions, etc.); add everything discoverable from SRC, official site, news, or public company names to the queue. |
| **Wholly owned entities, levels 1–4** | Follow subsidiary → grandchild → great-grandchild → fourth generation, **as far as possible to level 4**; seed legal entity names, former names, public brands/root domains; **do not include minority-owned affiliates** (§3.2.0). |
| **Wholly owned root domains / registered primary domains** | Large companies often have far more than 15; list as many as possible; live-filter P2 long-tail domains before FOFA. |
| **SRC site scope** | Cover all business lines, domains, and product lists; add them to the queue as P0. |

**No quantity limit.** List as many business names, wholly owned entities through levels 1–4, root/registered domains, SRC scope entries, and recycled R domains as possible. Some SRC programs have dozens or hundreds of entries; that is normal and required. **Do not** cut the list to a fixed number.
“Persist at least one batch before starting” only prevents laziness such as writing a single copy of the user’s original term; **no count is a goal or a ceiling.** If tools are unavailable → write down all known entries and start; when tools work again, **incrementally add to the queue.**

| Source | Details |
|--------|---------|
| User’s original term | An entry point, not the only seed. |
| Business names / sub-brands | Add as many as possible; no cap. |
| Official SRC site | Cover all and add as P0. |
| Wholly owned entities, levels 1–4 | Follow 100% ownership chains; exclude minority-owned affiliates. |
| Registration/root domains | List as many as possible; live-filter P2 first. |
| **Root domains recycled from valuable surfaces** | A valuable asset confirmed by FOFA/on-site work → extract its **registered root domain (second-level domain)** as a new seed and search it again (§1.1.1). |

**Tiering (what to search with FOFA first):**

| Priority | Seed type | Strategy |
|----------|------------|----------|
| **P0** | Business named by the user, domain/business line published by the SRC, root domain of a known active product | FOFA directly; still discard junk afterward. |
| **P1** | Commonly seen in Qichacha/certificates, or domains whose title/org repeatedly matches the brand | FOFA only after passing the quick live-domain check. |
| **P2** | Large lists of ICP-registered primary domains, historical shell domains | **Must pass the §1.2 live check first**; if not, do not send to FOFA. |
| **Junk/cold** | Dead registrations, parked-only domains, no DNS | Do not touch again this round. |

Finish P0/P1 using one-seed loops before scanning P2; **do not** start by running FOFA over hundreds of unfiltered registered domains one by one. The queue may list many P2 domains (no quantity limit), but each must pass a live check before being searched, and only one may be searched at a time.

### 1.1.1 Recycle Root Domains from Valuable Assets as Seeds (Hard Rule · User 2026-08-07)

> During free-jump work, when FOFA/on-site work finds a **valuable live surface**, **immediately** add the corresponding **searchable domain keyword usable with `domain=`** to the seed queue.
> Only **after finishing all remaining live surfaces in the current seed** and when it is time to switch, **prioritize** searching that term across its whole domain. If there are hosts new relative to covered hosts, investigate them; otherwise move to another seed.
> Purpose: do not pin searches to only the user’s original term; root domains for valuable business clusters must roll into the keyword pool for further expansion.
> **Do not change course immediately:** adding a domain back does not mean searching R right away; do not start a new search for R before completing the current seed’s loop.

**Terms (understand these; do not hard-code example domains into the rule):**

| Term | Meaning |
|------|---------|
| **Business host** | A specific site (multi-level subdomain / entry point with a path); mark covered and run the matrix. |
| **Search domain R** | The level obtained by moving **upward from a valuable host**, suitable for `domain="R"` (the business-cluster root/independent brand root, not every leaf host). |
| **When to persist** | **As soon as a valuable testable surface is confirmed**, persist it as pending; **do not wait** for exhaustive testing of that site. |

**Reduction rules**

- Leaf subdomain → move up to its **business-cluster root** for searching (multiple third-level hosts in one cluster correspond to **one** R).
- If the host is already a business-cluster root/independent brand root, R is the host itself.
- Do not only recycle a group-wide root that is already in the queue; **new product/business-cluster roots** should each be queued and searched with `domain=`.
- **Do not** treat every complete host string as a new seed for FOFA. |

**Recycle only when all conditions hold:**

1. Source: FOFA for this task **or** a host from on-site/JS/response work (already passed §3.2.0).
2. **Valuable:** large business surface / many JS files or endpoints / login backend / business API / Swagger or admin console, etc., with room to run the matrix further.
3. Reduced to an R not **already** in the seed queue.
4. Ownership passed the gate; do not recycle minority-owned affiliates / CDNs / third parties / junk shells.

**Do not recycle:**

| Do not queue | Reason |
|--------------|--------|
| Junk shell / Hello page / same-shell SEO site | No business-matrix value. |
| CDN / image host / third-party library | Noise. |
| R already in queue | Prevent an infinite loop. |
| Multiple hosts in the same cluster when R is pending/done | Add R once only. |
| Minority-owned affiliate / outsourced / out-of-scope asset | Ownership gate. |

**Persistence and search cadence**

1. **As soon as value is confirmed**, add a line to `种子队列.md`: seed=`R`, P0/P1, **pending**, note “recycled · source host=…”.
2. **Continue on the current host** (§4.3); keep investigating the rest of the current seed. This can happen in parallel with persisting R.
3. **Only after all remaining live surfaces in the current seed are finished and it is time to switch:** take a recycled pending entry first → search only R with `domain="R"` (page through results as needed) → deduplicate/discard junk/dead targets + ownership gate → **finish investigating the remainder** before searching the next seed.
4. **If** new hosts/surfaces relative to covered are found → **prioritize finishing them**; **if none**, immediately move to the next pending seed (do not ask to continue).
5. No quota available: persist R first and investigate the remaining assets already in the current seed; if quota is available, search R only after the current seed is complete.

**Prohibited:** Run FOFA on every leaf host; record a valuable host but fail to recycle R; recycle and then fail to deduplicate/discard junk/dead targets or investigate all survivors; keep searching R after it is exhausted; **run FOFA on R while live surfaces remain in the current seed.**

**In one line:** Investigate valuable hosts; immediately add the upward-reduced **search domain R** to the seed pool; **only after the current seed is complete**, prioritize FOFA over all of R for new assets, then move on if there are none.

### 1.2 Long-Tail Registered Domains · Live-Domain Filtering (Hard Rule)

**Registered does not mean testable.** Cheaply filter long-tail domains first; send live ones to FOFA and skip dead ones immediately.

| Dead/cold (skip) | Live/worth sending to FOFA |
|------------------|-----------------------------|
| No root-domain DNS **and** no useful host from crt/passive sources | Root or **subdomain** resolves (business sites often expose only api/admin). |
| Probes of root + www + known subdomains all show parked/default pages | Any host has a real business surface (login/API/admin console). |
| FOFA `domain=` count probe returns `size=0` or only parked results (when quota is available) | Multiple hosts/ports or distinct titles already observed. |
| No title/certificate/redirect link to the brand | Certificate SAN contains a business name; or root domain was revealed by a confirmed live site. |

**Do not declare dead based on one check:** if only root/www fail, also check crt/passive subdomains; 401/403/login wall alone = **live**; without FOFA quota, live filtering uses DNS+HTTP+crt only, **do not use FOFA again just to filter.**

```
Long-tail registered domain d
  1) Root DNS? If no → do not immediately declare it dead; do step 3)
  2) Short-timeout probes of root/www on 80/443 → business-like response means live; parked/unreachable → step 3)
  3) crt/passive subdomains → useful subdomain means live; none and root/www dead → cold
  4) Optional domain result-count probe if quota is available; skip if not
  5) If it passes → run full FOFA → discard junk → investigate
```

**Quickly discard even after FOFA:** empty/very few results and all CDN default pages; uniform parked titles; only email/unrelated ports; no connection to the brand → **immediately switch to the next seed**, do not force it.

| Stage | Filters |
|-------|---------|
| §1.2 live check | Dead registrations at the domain-field level |
| §3.1 discard junk | Noise hosts returned from live domains |
| §3.2 Hummingbird check | Only uncertain ownership |
| §3.2.0 ownership gate | Minority-owned/non-100%-owned entities |

### 1.3 Task Folder: Resume / Start New / Everything Already Covered (Hard Rule · User 2026-08-06)

**In one line:** If an old folder exists, skip hosts already tested/searched; if it was cleaned, start a new task with many seeds in shuffled order; if everything has been covered, search randomly again or rescan for additions.

**A. Old folder remains (resume)** — When the same ambiguous target is requested again:

| Do | Do not |
|----|--------|
| **Enter the existing** `Desktop\{目标}_SRC挖洞\` | Create another “{target}2_SRC挖洞” or treat it as a new task |
| At startup read: `资产/covered_hosts.txt`, `资产/种子队列.md`, `报告/`, and dig STATE | Ignore covered data and rerun a full matrix on a done site |
| Prefer pending entries, uncovered hosts, new FOFA surfaces not covered yet | Treat done/cold seeds as P0 and run them pointlessly again |
| Still test a **new path/port/application** on the same host (§3.1.1) | Misread “skip tested” to mean do not touch new endpoints on the same host |

**B. User cleaned the task folder (start new)**

There is no covered list/old queue to rely on → create a new directory and queue; still follow §1.1 for seeds; **the order may be shuffled**; do not grind forever in the fixed order of the user’s original term.

**C. Nearly everything has been tested and the user has not stopped the task**

Nearly all seeds are done/cold and nearly all hosts covered → **do not ask whether to continue or idle.** Prioritize step 1; rescanning is spot-checking, not a production line.

1. **Random new hunting (default first):** add business/product/domain keywords, run FOFA in shuffled order, and investigate only new uncovered hosts.
2. **Rescan old seeds (spot-check):** rerun FOFA / crt / homepage **at a glance** for previously done seeds; **care only about additions relative to covered** (new host, new path/port/application on same host, or a changed build number that exposed new APIs).

**Rescan acceptance gate (hard rule · prevent a rescan factory):**

| Evidence required before entering the site | Switch immediately; do not run the full matrix |
|-------------------------------------------|-----------------------------------------------|
| New host relative to covered | Build/UMI version unchanged and no new paths |
| **New** path/port/another application on the same host | Only confirming “same shell as Txx” |
| JS hash changed **and** new APIs were extracted | Reconfirming an already-reported finding in `报告/` only (chain extension, new path, or new type does not count) |

- Rescan output contains only a list of new surfaces or one line `0 additions`. `0 additions` **does not count as progress**.
- **Two consecutive** `0 additions` rescans in this round → the next slot **must** use a random new business name; do not start another “XX rescan.” Reset the counter between tasks or when the user starts a new window.
- Do not rotate through done seeds such as Wealth/Antshield/Morse/Chain using threads just to pad covered entries.

Still follow: 100% ownership, §1.5 quotas, §3.1.1 deduplication. “Do not report false leads” in an experience article does **not** mean “permanently ban this root domain” (see the corresponding `*src经验.md`).

### 1.4 Persist the Seed Queue + Round-Completion Gate (Hard Rule)

**Failure patterns:** finish a search round and stop + ask “should I continue?” + fail to switch to the next brand/domain; **or search all seeds first and investigate later.**
Use the **persisted queue + round gate + one-seed closed loop (§1.0.1)** to prevent this.

**Path (prefer the short path):** `Desktop\{任务}_SRC挖洞\资产\种子队列.md` (or `{名}_dig/seed_queue.md`)

| Seed | Type | Status | Notes |
|------|------|--------|-------|
| Business name / SRC domain / wholly owned subdomain… | P0/P1/P2 | pending / doing / done / cold | When to FOFA, pool size, reason for cold; when resuming, done = already searched, skip first |

**At startup:**
- New task → write the queue (as many business names as possible + wholly owned levels 1–4 + SRC/registration domains; as many as can be listed, no limit; order may be shuffled).
- Resume → **read** queue + covered + reports; start at pending or §1.3-C; **do not** overwrite history with an empty queue; incrementally add level-4 entities and new business names where possible.

**Prohibited:** queue contains only the user’s original term; write a fixed number and call it complete; list items verbally without persisting; pretend not to see the old queue when resuming.

```
pending → doing → search only this S → deduplicate/discard junk/dead targets + ownership gate → finish investigating all remaining live surfaces → done / cold
Live surfaces remain for this seed → do not take the next pending entry and run FOFA
In the same tool loop when this seed becomes done/cold → immediately take the next pending entry → doing → repeat the same loop
Prohibited: after done, end the round and wait for the user to say “continue”
Prohibited: search multiple seeds first and investigate later
All done and not stopped → §1.3-C (random search/rescan), not stopping work
```

**Terminology guardrails:**

| Phrase | Correct meaning | Incorrect (prohibited) |
|--------|-----------------|------------------------|
| Finish this seed / round complete | End only the current seed or site; seed finished = remaining live surfaces are done, then immediately take the next pending seed and repeat the loop | Free-jump task is over; may ask user |
| Finish one round of FOFA searches | Search results for this seed are collected; start cleaning and testing; **do not search the next seed yet** | The whole group is done; or switch to next seed immediately after search |
| All searches complete | All queue entries done → still follow §1.3-C, still do not ask whether to continue | Stop after searching everything |
| Batch FOFA | Page through results for this one seed | Search multiple seeds in a row / search them all first |

**Self-check before the final reply (if any item applies, do not wrap up and stop):**

1. Any pending entries? → Do not finish; mark the next one doing and continue.
2. Did a seed just become done? → Do not ask whether to continue; start the next one at the same cadence.
3. All done and covered? → Follow §1.3-C + §1.5; only actually stop if the user tells you to.
4. Does the last sentence say “should I continue/switch brands/what are your next instructions?” → **violation**; remove the question and take the next seed.

**Blacklisted endings (a free-jump bug):** “The round of searches is done, should I continue?”, “Should I test other brands too?”, “I’ll stop here until you give instructions”, “Should I switch seeds?”
**Legitimate reasons to stop:** user says stop/clean up/change target/explicitly lock scope; locked-scope cluster is complete; the entire environment is unavailable and alternatives have been tried, requiring a very brief blocker report.

**After context compaction/resuming:** first read `种子队列` + `covered` + `报告/`; resume from the first pending or incomplete doing item. Do not remember only the user’s original term after compaction.

### 1.5 FOFA Quota and Pool Replenishment

- **Quota available:** search only **this one** seed (page through its results if needed) → deduplicate/discard junk/dead targets + lightweight ownership check (§3.2) → **finish remaining live surfaces** → only then search next seed. Do not use an available quota to search multiple seeds in a row.
- **No quota (820041/429, etc.):** **stop idle FOFA attempts** (including count probes for live filtering); investigate remaining assets already in this seed’s pool; for P2 use DNS+HTTP+crt for live filtering; when quota is available and this seed is complete, take the next untested seed.
- **No results / API down / quota exhausted and pool empty:** pool replenishment must still deduplicate/discard junk/dead targets, check lightweight ownership, and apply the ownership gate:
  - crt.sh, subfinder, user’s existing `资产/`, and **in-scope** business hosts extracted from on-site JS
  - Add new business names from SRC/Qichacha; newly registered domains enter P2 and must be live-filtered.
- If a tool is missing, switch methods; **do not stop working**. Save confirmed findings under `vuln-report-format`.

### 1.6 Expand Scope · Start · Continue Hunting

| Action | Details |
|--------|---------|
| Main expansion | **One-seed closed loop (§1.0.1):** search one → deduplicate/discard junk/dead targets → finish the rest → only then search the next (+§1.5); use Hummingbird only if ownership is uncertain. |
| **Recycle valuable root domains** | A valuable live surface confirmed by FOFA/on-site work → add the new **registered root domain** to the seed queue; search it again **only after the current seed is complete** (§1.1.1). |
| Host from JS/config/response | Add a same-product business host to the queue only if the entry site has passed **§3.2.0**; do not expand from a minority-owned site; new root domains also follow §1.1.1. |
| Fake domains/business keys/third-party library domains | Do not queue or search with FOFA. |
| JS after entry | Primary use is endpoint discovery on the current site (§4.1: paths + keys); queue real business hosts as above. |
| Entry priority | Business JSON / backend / admin console / Swagger / Actuator / debug endpoints. **A login page means there may be a business surface behind it** (§4.1.1); do not grind on the login form. Login-form weak passwords are not an entry priority. Not grinding on the form does not mean skipping authentication endpoints (§4.2.2). |
| Seed rotation | After cleaning a seed and finishing §4.3 on every remaining live surface or marking it junk, switch to the next seed and search it; P2 must be live-filtered first. |
| Keep hunting | Site complete → next live site in this seed; all remaining surfaces in seed complete → search next seed; **do not stop after writing a report**; switch methods if a tool fails. |
| Prohibited | Search only the user’s original term; **search multiple seeds first and investigate later**; start FOFA on the next seed while live surfaces remain in this one; stall because Qichacha is incomplete; ask whether to continue; treat one FOFA round as task completion; record a valuable root only as a host without recycling/searching it; **keep searching FOFA with new terms while live surfaces remain untested** (§2.1). |
| Deep-testing priority | See **§2.1**; expansion exists to feed on-site work, result counts are not progress; do **not reduce** seed breadth or matrix depth. |

If SRC/Qichacha is down → **do not block testing**; use available seeds and replenish the queue when service returns.
Progress may be recorded briefly (“finished remaining live surfaces for seed X / saved report for site Y”), but **do not end by asking a question.**

---

## 2. Detect Idling (Correct Course as Soon as a Signal Appears · Self-Triggered)

**User’s hard rule (2026-08-07): skip when idling.**
When idling is detected → **immediately** switch to the next live URL in this seed (or mark the current one junk); do not keep probing a 403 shell, credential-free health endpoint, already-covered login-form shell, or already-reported issue. Clicking the login form does not mean §4.2.2 is complete; do not repeat authentication probes after they are checked.
**Still tracing redirects / using JS to find business surfaces is not idling.** Testing session issuance/reset/rebinding/ticket replacement under §4.2.2 is not idling. Grinding CAPTCHA/default credentials/another attempt at the same login shell is idling.
A **rich surface** (lots of JS / backend / business JSON) where you are still extracting chunks, restoring signatures, writing the §4.0 notes, or mapping the object graph → active work, **not** idling. A **thin shell** (403, no business scripts, same gate) may be skipped after 15–20 minutes without a lead.
**Do not use idling as a reason to search FOFA for a new seed while live surfaces remain in this seed.**

| Signal | Meaning |
|--------|---------|
| `报告/` unchanged for a long time + only 403/health/repeated paths accumulating | **Idling → switch to the next live host in this seed** (do not search a new seed). |
| Subthread/current thread has >15–20 minutes of noise with no lead that could go in a formal report | For a **thin shell**, kill this path and switch to the next live URL in this seed; a **rich surface** still being analyzed for JS/object graph does not meet this condition. Do not search a new seed before finishing the rest. |
| Finish a seed/FOFA round, stop, and ask whether to continue | Improper stopping (§1.4); after FOFA, clean and investigate; do not stop or immediately search the next seed. |
| Search multiple seeds first / run FOFA on the next seed while live surfaces remain in this one | Violates one-seed closed loop (§1.0.1). |
| Old folder exists but start from scratch without skipping covered | Violates §1.3-A. |
| All done, then idle without random search or rescan | Violates §1.3-C. |
| Repeatedly rescan done seeds this round, unchanged UMI but full matrix / pad only covered entries | **Rescan factory** (§1.3-C gate); use a random new business name for the next slot. |
| Queue is not persisted or contains only the user’s original term | Brands/domains will be missed. |
| End the reply with “should I continue/switch brands?” | Blacklisted phrasing. |
| Search only the user’s original term and fail to create other available seeds | Missed scope (§1.1). |
| Mix multiple seeds together, then rerun full matrices on covered hosts after a seed switch | Violates §3.1.1. |
| Hummingbird every entry / blindly run FOFA on every registered domain | Violates §3.2 / §1.2. |
| Deep-test minority-owned company as a group asset | Violates §3.2.0. |
| Keep trying FOFA after 820041/429 | Quota-exhaustion collision (§2.1 gate D). |
| Start new-seed FOFA while live surfaces remain in this seed | Fake expansion progress (§1.0.1 / §2.1 gate A). |
| Map new keywords aggressively before the site meets §4.3 | Violates deep-testing priority (§2.1 gate B). |
| Drop the current matrix and run FOFA as soon as a valuable term appears | Violates “recycle only; do not change course” (§2.1 gate C). |
| All threads run FOFA/live checks; nobody builds an endpoint list | Dual-FOFA factory (§2.1 gate E). |
| Blind scan without an endpoint list on a frontend site; test only two or three types | Fake progress / skewed testing. |
| Get stuck waiting for JS extraction to finish | Misuses §4.1. |
| Repeated CAPTCHA/default-password attempts on identical login walls; test first section of shells while user-marked priority surfaces wait | Misreads locked-scope order/identical shells (§0.1 / §3.3 / §4.1.1). |
| See a login page and move to the next site without checking business APIs | Violates §4.1.1. |
| List contains session issuance/reset/rebinding/ticket replacement, but switch sites with zero payloads | **Missed testing** (§4.2.2) → add probes first; this is not a reason to skip to another site. |
| Business APIs extracted, all return the same login code, but open another full matrix on the next host under the same product | **Grinding the same gate** (§4.1.2) → close this host, quickly compare the sibling, switch to a truly untested business surface. |
| Reports unchanged for a long time, only scope-expansion probes | Hard signal of idling. |
| Full matrix on each identical-shell mirror; count a recheck of an already reported issue as new | Repetitive idle work. |

**Does not count as progress:** FOFA result count, blacklist entries, live-check counts, fake middleware/pure file channels, reconfirming an already-saved report, CORS (do not test); checking off the four-part set on a pure login code / “please log in again” / NotLogin / “product being upgraded” response; sending `'` to every path without differential surfaces.
**Counts as progress:** the short §4.0 scope description, endpoint list (or valid fallback), response clues added to the current site’s queue, full type matrix on business JSON, four-part probes on differential surfaces, §4.2.2 completed where an entry exists or N/A if none, §4.2.3 object graph/ID changes with a session, formal report saved, **unauthenticated exception tested or disproved**.
**Locked scope:** correct idling only by switching paths within the authorized scope; do not use idling as an excuse to expand beyond it.

### 2.1 Deep-Testing Priority Gate (Correct “Expand Scope Without Visiting Sites” · User 2026-08-07)

> **Purpose (same spirit as `skill-as-boost`):** this section controls **cadence/idling**, not the capability ceiling.
> **Corrects only:** treating FOFA counts as progress, stockpiling keywords without investigating, both threads searching at once, abandoning a half-finished site to map more scope as soon as a valuable term appears.
> **Never cuts:** preparing as many seeds as possible, full type matrix, depth on injection/SSRF/XSS/RCE, following JS hosts, vulnerability chains, new attack surfaces, switching tools, or stronger instructions in the user’s current request.
> Mantra: **search one seed → clean → finish remaining live surfaces → only then run FOFA for another seed**; within this seed, endpoint list before matrix, matrix before switching sites; recycle valuable terms without changing course; when rate-limited, work the pool.
> Relationship to §2 “skip when idling”: skipping a dead shell/403 noise surface (switch to the next live URL **within this seed**) = correct; running FOFA for a new seed while live surfaces remain in this seed = wrong (see §1.0.1 / gate B).

#### 2.1.0 Capability Boundary (Prevent Misreading This as Cutting Capability)

| This section may constrain (cadence only) | **Absolutely does not mean** |
|------------------------------------------|------------------------------|
| If live sites remain untested in this seed, hold the next seed search | Prepare fewer seeds or impose a “maximum N”; seed count has no limit, list as many as possible |
| If the current site matrix is incomplete, do not use “run another FOFA batch / search all seeds first” to pad progress | Do not follow new hosts from JS, skip multiple paths, or stop high-value leads |
| Do not make both threads into FOFA factories | Do not let two threads deep-test two hosts at once |
| Persist a valuable term in the queue first | Do not search recycled R or supplement fourth-level entities/new business names |
| Stop idle searches when rate-limited | Do not use backup accounts; do not replenish the pool even when empty |
| Same-gate rule: close the family that uses the same login code; do not run a full matrix on siblings (§4.1.2) | Switch as soon as a login page appears; skip business API extraction; skip unauthenticated exceptions (public list exposing another principal, missing-parameter request still returns list); ignore new paths/data after response changes |
| Separate rich and thin surfaces in this seed: quick-check thin shells, deep-test rich surfaces (§4.3) | Kill a rich surface if it has no report after 15 minutes; spawn a thread on a thin shell to spray the four-part set |

If a rule conflicts with live evidence or would risk losing a finding: **follow the breakthrough/evidence**; cadence constraints yield to results, and the reason for breaking the gate may be recorded afterward.

#### 2.1.1 Gate A — Live Surfaces Remain in This Seed → Stop FOFA on New Seeds

**If either condition is true → do not run FOFA on new seeds/new keywords this round; test only cleaned remaining assets in this seed:**

1. Any live host remains in the current seed after cleaning that has **not met §4.3 or been marked junk**.
2. Or, in the last 15 minutes, most work was scope expansion/live checks while **the endpoint list is missing and `报告/` has not grown**.

**FOFA/expansion is still allowed:** page through results for this seed; search another seed when this seed’s remaining live surfaces are done/marked junk; when the user names another line; when a site is entirely dead/permanently blocked and recorded as junk; or for the random/rescan flow under §1.3-C when all seeds are done.

#### 2.1.2 Gate B — Do Not Use New-Keyword FOFA to Pad Progress Before “Round Complete”

If the current doing host has **not met §4.3**, or live surfaces remain in this seed:

- **Do not** make “run more FOFA / search a batch of keywords / search all seeds first and come back” the main action.
- **Must first:** clarify scope under §4.0 → extract all JS (paths + keys, or valid fallback) → build the list (add response clues as they appear) → matrix (four-part set on differential surfaces under §4.2.1 + §4.2.2; with a session add §4.2.3; mark N/A if no entry) → save under the format or disprove; extend Medium issues on the same object; then switch to the next live URL in this seed (quick-check thin surfaces).
- Break through a high-value lead (stable injection/RCE/cross-tenant access/sensitive unauthenticated read/usable credentials (validated key + safe capability example, not merely a copied string)/arbitrary login/account takeover/password change) **before switching sites**, consistent with §4.3.

Starting FOFA for another seed before meeting the criteria = **fake expansion progress**, which triggers correction under §2.

#### 2.1.3 Gate C — Valuable Keywords / Search Domain R: Recycle Only; Do Not Change Course

Reinforce §1.1.1’s cadence:

1. A valuable term/search domain R → **immediately** persist it as pending in `种子队列.md` (note source host).
2. **Continue the current host’s matrix** and test remaining live surfaces in this seed; do not interrupt a half-finished deep test or go search R.
3. **Only when** all remaining live surfaces in this seed are finished / a route change is necessary / gate A allows searching, prioritize FOFA for `domain=R` or that term and follow §1.0.1 again.
4. **Prohibited:** run FOFA for every leaf host; mention a good term only in chat without saving it; persist it and never search it or mark it cold; search R while live surfaces remain in this seed.

#### 2.1.4 Gate D — FOFA 429 / Rate Limit / 820041 → Work the Pool

- Main account rate-limited → may switch to a **backup key** (see FOFA’s three accounts: main → backup → backup2) to fill the **current gap** (missing pages for this seed/R), not repeatedly change syntax and retry.
- All three accounts are limited or coverage is sufficient → **no more FOFA for this round**; test what is already in the pool + hosts from JS/crt.
- **Do not** idle while rate-limited, keep retrying, or treat “switch account and inflate result count” as progress.

#### 2.1.5 Gate E — Multiple Threads: Deep Testing First; No Dual FOFA Factory

When the user requests N threads (such as two), default to:

| Thread role | Only do | Prohibited |
|-------------|---------|------------|
| Deep-testing thread (default all/most threads) | Assigned host: §4.0 → JS (paths + keys) → endpoint list → matrix (four-part set on differential surfaces + §4.2.2; with session §4.2.3) → report; for High findings, add proposed/update/do-not-add under `hunt-iter` to DONE | Make new FOFA keywords the main work; **edit the short table directly**; run a full matrix on a thin shell |
| Optional one thread for additional coverage | Deep-test the next valuable queued host, or a host revealed by same-cluster JS | Run FOFA aggressively at the same time as the deep-testing thread |

FOFA is triggered **briefly** by the main controller (page through this seed / change seeds only after this one is done / prioritize recycled R for new hosts). After results enter the queue, threads **immediately resume deep-testing the rest of this seed**.
**Do not** have N threads all running FOFA/live checks while zero build endpoint lists.
**Do not** let the supplemental thread search the next seed.
Only the main controller (or a single session) edits the short table; subthread proposals follow `hunt-iter`. Spawn handoffs must include iteration.

#### 2.1.6 Persistence and Covered State (Make Pretend Work Impossible to Hide)

Suggested files for an in-progress site (useful for self-checking, but **do not substitute for actual testing**):

```
{任务}_dig/{host}/endpoints.md   # endpoint list
{任务}_dig/{host}/matrix.md      # type matrix; four-part set checked or N/A + reason
js/{host}/                       # all extracted JS (when a frontend exists)
```

- Marking `covered_hosts` means §4.3 is complete, not “curl homepage / saw it in FOFA.”
- Three progress indicators (heartbeat every 15–20 minutes): ① JS/list advanced (rich surfaces include object graph/signature recovery); ② four-part probes sent to differential surfaces or marked N/A; with session, changed IDs under §4.2.3; ③ report added or site matrix closed.
- None of the three occurred and only FOFA/live-check counts are visible → **force gate A**; return to an incomplete site.

---

## 3. Discard Junk · Establish Ownership · Ownership Gate

**After FOFA/supplemental collection, clean the results; do not investigate them all as-is.**

### 3.1 Deduplicate / Discard Junk — Keep Business Paths

| Remove | Do not remove |
|--------|---------------|
| Duplicate entries for the same host:port | Different paths/endpoints on the same host |
| Fake TLDs, business keys mistaken for domains | `/api/a` and `/api/b` |
| CDN/static/third-party library domains | Different ports on the same host when the business surface differs |
| Internal IPs, unrelated sites | Different business hosts under the same brand |
| Blank/garbled results, purely static shell with no endpoints, long-dead pages | **401/403/login walls/admin-console challenge pages can contain live surfaces; do not discard a whole site for a 4xx.** |

**Prohibited:** Misinterpret “deduplicate by host” to mean “test only the homepage and drop API paths.”

### 3.1.1 Global Deduplication Across the Entire Task (Hard Rule)

Multiple seeds in the same `*_SRC挖洞` folder will inevitably return duplicates — **do not** treat them as new assets and rerun the full matrix.

| Level | Handling |
|-------|----------|
| Same URL (normalized host+port+path) already in assets/queue/covered | Skip |
| Same host:port already completed under §4.3 | If found again → skip deep testing |
| Same host, **new path/endpoint** | **Do not skip** |
| Same host, **new port** or another application | Treat as a new surface |
| True identical-shell mirror (§3.3) with a tested representative in its family | Skip the full matrix for the rest |
| Issue already saved in this task’s `报告/` | Do not submit it as a new issue (whether it is identical and whether to report follow the format). Same type on a different URL and different type on the same URL should still be tested. |

Suggested state in dig: `资产/`, covered/done hosts, cold/black list, and the current site’s endpoint list.
After switching seeds, **deduplicate against the existing asset table before investigating**; duplicate counts do not count as progress.

### 3.2 Ownership (Lightweight by Default · Hummingbird Only When Uncertain)

Before deep testing: discard junk → **§3.2.0** → lightweight ownership check → add to pool. **Hummingbird is not the default step.**

#### 3.2.0 Ownership-Scope Gate (Hard Rule · Prevent Testing Minority-Owned Companies as In-Scope)

> Lesson: a minority-owned company’s own domain is not 100%-controlled by the group; it is outside the group’s SRC scope no matter how many vulnerabilities it has.

| Normally add to pool | Normally do not add |
|----------------------|----------------------|
| The group itself and **100%-owned** subsidiaries/grandchildren/great-grandchildren/fourth-generation entities (try to reach level 4; if ownership remains 100% beyond level 4, you may continue following it) | **Minority-owned affiliates**, strategic/financial investments, JVs that are not 100%-owned, associated companies |
| Brands/domains explicitly included on the SRC scope page | Run FOFA across all domains merely because they “received investment from X” |
| Known wholly owned brand roots and business subdomains | An investment portfolio company’s own domain (minority-owned) |

**User-named exception:** A minority-owned/affiliated asset enters the pool **only if the user explicitly says in the current request** “test company X / also test this affiliate.” If not named → skip it in the cold pool, however rich the surface.

| Signal | Action |
|--------|--------|
| SRC / known wholly owned brand/root domain | May pass |
| Corporate registry shows <100% ownership, minority ownership, strategic investment | Cold pool (unless user names it) |
| Registered entity is another independent company with only an investment relationship | Cold pool |
| Ownership unclear and not SRC/known wholly owned | Shallow probe; once confirmed not 100% owned → stop deep testing and do not report it as part of this group |
| Finding already confirmed under the wrong entity | Mark as outside the group / do not submit to that SRC; stop expanding this cluster in the task |

**Prohibited:** Treat “received investment” as a wholly owned subsidiary; use a minority-owned site’s JS to expand to its whole family; keep “breaking through first” when the scope is wrong; run FOFA on the entire Qichacha “related companies” list.

#### 3.2.1 Do Not Use Hummingbird by Default

If the surface reasonably appears within the target’s **wholly owned/SRC scope** (subdomain of a known root, certificate/title strongly tied to a wholly owned brand, FOFA results within this seed’s scope, or host revealed by an already confirmed in-scope site) → **skip Hummingbird** and visit/shallow-probe it.
Minority-owned/non-100%-owned → cold pool; do not use this section. Fake domains/CDNs/third parties already removed as junk → discard directly; no Hummingbird needed.

#### 3.2.2 Use Hummingbird Only When Ownership Is Uncertain

Title/certificate/domain conflict; does not resemble a known root and lacks brand evidence; looks like another company/outsourcer/shared shell; or a deep test suggests the company is a minority-owned affiliate → use Hummingbird or cross-check certificates/WHOIS once.
**Do not** use Hummingbird for every entry; if it fails, switch to certificate/title/shallow checks; do not get stuck. Foreign/no-ICP cases do not require Hummingbird. If there is a registered entity, still apply §3.2.0.

```
Uncertain host → certificate/title/known suffix (quick check) → still unclear → one Hummingbird check
  → unavailable/no ICP → WHOIS/org or shallow probe → business surface/rich component means deep test; dead shell goes to junk pool
```

**Good assets:** in scope + live on the public internet + login/business API/backend/component surface.
**Bad assets:** fake domains, third parties, empty shells, pure health checks; unauthorized pure file upload/download is not a finding.

### 3.3 Identical Shells / IP Ranges

Only when **fingerprints are materially identical** (title/skeleton/build hash/API prefix/login flow): run a full matrix on **2–3 representatives** of the family; on the rest, do only probes (homepage + JS hash + one or two business endpoints); if there is no differential → **skip the full matrix**.
**Same host ≠ same shell;** expand all paths on the same host.
**A proven same-gate (§4.1.2) overrides this section’s “find 2–3 more representatives”:** after checking the first site’s endpoint list and unauthenticated exceptions, if all other endpoints use the same login code, a sibling only needs a quick differential check; no need to find two more sites for full matrices. Escalate if the response code changes or a new path is extracted.

**Identical login shells** → do not retest the login form. If the representative site was tested under §4.2.2, compare siblings at a glance for new reset/rebind/ticket-replacement paths; if there is no new endpoint, do not rerun §4.2.2. Still check whether `jump` / `service=` / `moduleId` refers to a different business surface; if so, investigate that surface.

**Not a differential (do not start another full matrix):** entry point returns 200 vs 403; whether the same SPA opens; same gateway prefix + same login wall + same JS hash; another attempt at weak passwords/CAPTCHA.
**A differential:** new path / new port / another application / new business API; identical login shell but redirect to another business surface.

Separate rich/thin surfaces within the seed (§4.3): a quick disproof is enough for thin identical-shell/same-gate surfaces; do not spawn a thread to spray the four-part set at every leftover host.

---

## 4. Single-Site Main Path

Entry mantra (details follow; do not invent a separate workflow here): clarify scope → extract paths+keys → allocate effort according to `src-value` §1.1 → do not spray parameters at “please log in,” test empty lists, stop if there is no differential → without an account, once unauthorized access works change IDs (any field name); with an account, change IDs while carrying the session (any field name) → extend Medium issues first → deep-test rich surfaces, quick-check thin ones.

### 4.0 Clarify the Surface Before Entry (Hard Rule · Prevent Blind Scanning)

Before making the endpoint list, write 3–5 lines (not an essay) that clarify:

- Who uses it (consumer / merchant / operations)
- Core objects (orders, tickets, coupons, documents, etc.)
- Which fields hold money, permissions, and status
- What layer can be reached without logging in

If you cannot explain this, capture traffic first; do not start scanning with the section blank. Later parameter changes should revolve around these points.

If you see a mini program / app / client / GraphQL / WebSocket / bulk export / agent tool / template preview / file conversion / command RPC → follow it as a **surface on this site**, not as “another live surface” on the side. Treat `appId` / mini-program gateway / client salt inside an H5 login wall as keys for this site.

### 4.1 Build the Endpoint List First

**With a frontend:** open a business page → extract all JS → save under `js/` → analyze APIs/parameters/authentication → complete from captured traffic → build the list → §4.2.

Do **more than extract paths** from JS. Record each item if found; otherwise write “none”:

| What was found | What to do |
|----------------|------------|
| `/api/` paths | Add them to the endpoint list as usual. |
| Signature salt, hard-coded key | Compute it yourself; test endpoints that need “sign only, no Cookie.” |
| Encrypted ID + frontend public key | Encrypt an adjacent ID yourself, then change it (see `idor-test.md`). |
| Async chunks, source maps, hidden/admin routes | Call the corresponding APIs even if they are not displayed on the page. |
| Hard-coded demo account, test tenant, trial entry point | **Use it as a key.** This is different from spraying `admin/admin` at the login form. |
| Command execution / template rendering / expressions / file conversion / command-capable RPC / agent tools | If present, add to the list and open the relevant module (for conversational tools, see `agent-tool-exec-test.md`); **if absent, write “none.”** Examples are not a whitelist; investigate execution surfaces outside the list too. |

“Weak passwords are not mandatory” applies only to dictionary attacks on login forms. Credentials and demo tenants in JS/pages are not prohibited.

**Fallback paths (do not get stuck):**

| Situation | Approach |
|-----------|----------|
| Pure API / almost no business JS | Swagger/OpenAPI, captured traffic, response links, common gateway paths |
| Opens to a login page | **Follow §4.1.1:** find business surfaces first; main work is unauthenticated testing. Test visible login-form paths through success or disproof and stop. Session issuance/reset/rebinding/ticket replacement in the endpoint list follows **§4.2.2**; do not grind on the form. |
| JS download fails / CDN blocks it | Captured traffic + inline HTML + known paths; **do not wait idle for extraction.** |
| No comparison account | No account: once unauthorized access works, change IDs (§4.2.3). With an account, follow §4.2.3. Do not grind on registration for a second account. Do not stop at the login form to “get logged in first.” |

Frontend present but empty endpoint list and blind directory scans = prohibited; no JS and stopping the whole site = prohibited.

### 4.1.1 Opens to a Login Page (Hard Rule · Default for Free-Jump Is Unauthenticated)

**Do not misread this as “ignore everything related to login.”** What is not required is grinding through CAPTCHA, default credentials, or unlimited guesses on the form. Test visible paths such as empty passwords, skipped steps, modified responses, JS keys, `tenant`/`moduleId` on the login API, and nearby exposed lists/details.

If it opens to a login page → first find business APIs/business hosts reached after redirects → **main work** is unauthenticated testing and the four-part set on differential business surfaces. Once a session is obtained (user-provided account / JS demo account / empty-password success) → immediately switch to §4.2.3; do not keep sending quotes to business endpoints.
Login form: test visible paths through success or disproof, then stop; do not waste time on complex challenges / someone else’s identity page / same-shell login. Do not grind on another party’s login shell; test this business’s own ticket replacement / callback / binding only if they appear in the checklist (§4.2.2); otherwise mark N/A.
**A login page is not a reason to change assets.** It is an empty shell only if no business surface can be found. Same rule for locked scope without credentials. If the user supplies credentials during the task, add post-login testing.

The business surface may be another host, or a **gateway/API behind login on the same host**. Test business APIs for unauthenticated access; do not test the login form as if it were the business API. Failure to find another domain does not mean the surface is gone.
The login page may belong to this service or to another party (SSO/CAS/OAuth/unified authentication, or any third-party identity page). Do not audit another party’s login page itself—the criterion is “the page owner is not the current business,” not a product-name list. Follow the flow back to the current business surface.

**How to find business surfaces (without logging in):**
`service=` / `redirect_uri=` / `callback=` / `returnUrl=` / `jumpUrl=` / `next=`; JS / `env.js` / `baseURL` / `apiHost` / `/prod-api`; 302 Location; `X-Frame-Options: ALLOW-FROM`; same-brand `api` / `admin` / `gateway` / product hosts.

**You may test the form, but do not grind forever:**

| Test visible paths through success or disproof | Do not test / do not count as progress |
|-----------------------------------------------|-----------------------------------------|
| Empty password, skip password step, modify response to log in | CAPTCHA OCR, sliders, default-credential dictionaries, unlimited password guessing |
| Hard-coded JS keys | Inject into username/password fields and count it as the four-part set |
| Extra business fields on login endpoint (`tenant` / `corpId` / `moduleId`) using injection probes | Retest the login form on an identical shell |
| Unauthenticated access to business paths already visible beside login (lists/details/stats) | Grind login HTML via `401-403-bypass`; audit another party’s IdP itself |
| Obvious default access on middleware (e.g. Nacos) | — |

**Run the four-part set against business parameters on business APIs.** Username/password fields are not business parameters and are not injection four-part-set entry points. If empty password / skipped steps / login injection on the same pair of fields yields a session, classify it as **arbitrary login**; see §4.2.2. If all business endpoints require a token and there are no pre-login parameters → mark the four-part set N/A with the reason; do not invent parameters.
Testing or disproving the form shell and stopping does not mean authentication endpoints can be skipped. Session issuance/reset/rebinding/this service’s own ticket replacement/2FA follow §4.2.2.
**Only** when no business surface can be extracted from redirects/JS/configuration, no unauthenticated business endpoint exists beside it, and the checklist has no §4.2.2 endpoints → record an empty shell and move to the next URL. Finding a business surface but stopping at the login form or switching sites is a violation.

### 4.1.2 All Business Endpoints Return the Same Login Code (Stop at Same Gate · 2026-08-18)

> **Correct the cadence only; do not cut capability.** Lesson: reading “deep-test each domain independently / extract all JS” as “spawn a full-matrix thread for every FOFA leftover, 403, and login shell,” then having the endpoint list and four-part probes all hit the same login code, no report increase, and opening another host under the same product.

**Do this first (do not reduce capability):** extract JS, build the endpoint list, mark methods; test unauthenticated exceptions (public enumeration, static JSON, business JSON after missing parameters, lists/details/stats). Compare the response with the table below; do not treat “no data yet” as “please log in.”

| Response shape | Means | Action |
|----------------|-------|--------|
| Only one “please log in / NotLogin” message everywhere, with no list/total/name/detail | Login gate | Mark the entire four-part set N/A; do not send `'`. |
| Contains `list` / `total` / business fields, even `total=0` or an empty array | Differential surface | Still test (change parameters, IDs, injection). |
| Error/500/timeout, or length/time differs noticeably from a normal request | Differential surface | Still test. |
| Missing a parameter causes a list/detail to appear | Unauthenticated exception | Test under §4.2.1. |

**An empty list means “the structure was returned with zero items,” not “please log in.”** Skip the whole section only if the entire page consists of the same login code and you cannot extract a list elsewhere.
**Stop:** after trying parameter changes, if total remains zero and there is no stable error/time difference → disprove and move on; do not grind until data appears. Compare an intermittent 500/timeout once or twice; if it is not stable, stop—do not retry until it works.

**Same gate means all of these are true:**

1. Business paths have already been extracted from JS/gateway.
2. Unauthenticated exceptions have already been checked.
3. Remaining business endpoints return the **same response family** (login failure / please log in again / NotLogin / product upgrade in progress / system busy with no count/time differential).
4. There is **no** data about another principal without login and no list/detail exposed after omitting a parameter.

**Then stop testing this gate; do not run the full matrix again:** stop the four-part set against the same login code. If the checklist still has session issuance/reset/rebinding/ticket replacement and **§4.2.2 is incomplete**, finish that group first and then mark done—do not inject into “please log in.” On another host under the same product with the same `baseURL` + same response code → **quick differential check** (extract JS to see if there is a new path or response change); if there is no new exception, do not open another full matrix. FOFA **403 + homepage has no business scripts + Apache/TAPISIX directory denial** → disprove with one check, mark cold, and do not open a thread.

**Do not misread this as:** switch as soon as you see a login page (still violates §4.1.1); skip JS extraction for business APIs; skip unauthenticated exceptions; ignore new data after a response-code change, new paths, or new business hosts; **skip authentication endpoints (login/reset/rebinding/ticket replacement)**; or treat an empty list/error/timeout as “please log in” and skip the whole section. Rechecking the four-part set on the same gate, starting another sibling thread, or grinding on an empty list until data appears is idling, not deep testing.


### 4.1.3 Treat Responses as Leads (Hard Rule)

A new **ID / download URL / token / internal host / role field** in a response → immediately add it to the current site’s list and test it; do not discard it as junk or lose it by switching sites.
If a list is accessible, also test attachments and exports once (the parent object may have authorization checks that the child object lacks).
### 4.2 Full Type Matrix (Do Not Neglect Any Type)

What to prioritize follows `src-value-hunting.md` §1.1 (another principal’s data without login / authentication takeover / ID changes / four-part set including XSS all have the same priority tier). This section sets probe requirements by type; it does not create another priority list.

Use the union with `src-value-hunting.md`; at minimum: unauthorized access, authorization bypass/IDOR (**as mandatory as the four-part set when a session exists; see §4.2.3**), injection, SSRF, RCE chains, credentials (if credentials exist, keep investigating; “usable” = key validated + a capability example that does not affect production; reporting follows the format), weak passwords (login-form weak passwords are not mandatory, see §4.1.1; treat JS demo accounts as keys), sensitive paths, XSS/CSRF, uploads (do not stop at upload/download), traversal, privilege escalation/logic (with a session see §4.2.3), **authentication/sessions (arbitrary login/account takeover, see §4.2.2)**, JWT/OAuth, and other live surfaces (**do not test CORS**). Whether to report is governed only by `vuln-report-format.md`.

#### 4.2.1 The Four-Part Set Must Not Go Untested for Days (User 2026-08-07 · No Empty Claims)

> The user explicitly said: do not pile up only unauthorized reads/identical lists; **actually test injection, SSRF, XSS, and RCE.**
> **Do not** use vague rules such as “perform the minimum action”; the table below lists the **probes required for each of the four types when the matrix is run on each site** (if there is an entry point, test it; mark N/A + reason only if there is none).

| Type | Required probes when an entry exists | N/A only when no entry exists |
|------|-------------------------------------|-------------------------------|
| **Injection** | For an endpoint where result counts/content change or a URL is fetched, test **every** query/filter/sort/keyword parameter (baseline → probe → differential). If only “please log in” is returned and there are no business fields → mark the entire section N/A; do not send `'` to every field. **Choose probes by stack**; do not test only SQL single quotes on every site: operators (`$ne`/`$gt`) for JSON/Mongo; ES DSL or list OR+total for search boxes; HQL/SpEL for Java; SSTI if templates exist. For SQL surfaces, continue with `'` / boolean / delay (based on DB) → WAF encodings. If blocked, move the probe (query / json / header / path). If data appears or there is a stable timing difference → confirm the finding and report it. Username/password fields are not business parameters; test `tenant` / `corpId` / `moduleId` on a login endpoint if present (§4.1.1). | Pure static page with no business parameters in query/body; only a login form with no other business fields; response contains only a login code and no business fields (§4.1.2). |
| **SSRF** | All URL/callback/webhook/file/fetch/import/preview/proxy parameters: internal network, cloud metadata, out-of-band DNS/HTTP; check whether the request is made, reflected, or reaches your listener. | **Zero** URL-fetching/proxy fields in the endpoint list and captured traffic. |
| **XSS** | Reflected: test tags/events in reflected parameters; stored: write a profile nickname/note/message/filename and read it back; consider whether it is readable without login. | No reflected output and no user-controlled write point (record the reason). |
| **RCE chain** | If command/expression/template/deserialization/debug execution/dangerous upload is present, follow the sink to execution or disprove it; keep following incomplete chains. | No such sink in the **endpoint list + captured traffic + responses**. Not finding a keyword on the homepage is not enough; if a tool/template/exec appears in a later response, immediately test it. |

| Hard prohibition | Explanation |
|------------------|-------------|
| **Untested for days** | Multiple rounds/natural days of process/report contain only unauthorized reads/lists/configuration, and the four-part set has sent **no payload to real parameters on differential surfaces** → skewed testing; add the four-part set first next round. |
| **Fake progress** | Findings say “tested/N/A” while parameter table is blank; only test unauthorized HTTP 200 then switch; **send `'` to every path without a differential surface**. |
| **Skewed cadence** | Finish an unauthorized list and switch hosts/seeds while failing to glance at injection/SSRF/XSS/RCE on the same endpoints. |

**Complete** = differential parameters were exploited or disproved; it does not mean sending `'` to every path. Prioritize parameters with a `total`/count, reflected input, URL fetching, template/expression handling. If only “please log in” is returned and there are no business fields → N/A, reason “response is only a login gate.” Compare empty lists/errors/timeouts under §4.1.2; these are not “please log in”; stop if there is no differential.

Detailed parameter matrices follow `src-value` §§3–4; **completion means the parameter table is done on differential surfaces, not merely that something was tried.**

Deliverable: save confirmed findings under `vuln-report-format`.

#### 4.2.2 Arbitrary Login / Account Takeover (Finish if There Is an Entry · Prevent Idling)

> **This is not a fifth test set.** The four-part set remains injection / SSRF / XSS / RCE. This item belongs under “authentication/session”: finish it when an entry exists; otherwise N/A + reason. The table is a **minimum**, so follow new endpoints from the list; do not misread this as “with an account, only run these few probes.”
> **No conflict with §4.1.1:** §4.1.1 covers the form shell (CAPTCHA UI, default-password dictionaries, identical-shell login HTML, another party’s identity page). This section covers **endpoints** for session issuance, reset, rebinding, this service’s ticket replacement/callback, 2FA, etc. Stopping at the form shell does not mean these endpoints can be skipped.
> After code sending/CAPTCHA works, still check whether the response contains a code, whether the code is account-bound, whether reset switches users, and whether empty-password login works. Details: account section in `authbypass-test.md` and `logic-test.md`.

**No account** (default for free-jump; locked-scope task has no supplied account)—one attempt per probe; once it works or clearly fails, move on:

- **Arbitrary login:** empty password / remove `password` from the request; skip password step and call the endpoint that actually issues Cookie/token; modify response `success` and check for a real session; for login injection, check only whether a session is issued (do not count it as the injection four-part set); JWT `none` / hard-coded key / client-generated ticket; on this service’s ticket replacement/callback/binding, change `uid` / `openid` (do not audit another person’s identity page itself; see §4.1.1).
- **Takeover:** check whether reset/code-validation responses contain a token or code; if so, use it to log in or complete the last reset step; in the last step replace `uid` / `email` / `phone` with another person’s or an adjacent account; if two identities can be constructed, submit A’s code for B.

**With an account** (user provided an account, you logged in, or a session exists)—test authentication endpoints under this section using “my session + swap to someone else’s identity”; do not count merely opening the personal-center page as progress. **Authorization/logic on business objects follows §4.2.3**; do not stop after these few rebinding/password-change probes.

**Do not invalidate the user’s session:** Never call `/logout`, `/signout`, `/revoke`, or any logout/sign-out/revocation endpoint. Do not test whether a session survives logout (that probe requires signing out). For rebinding/password change, only check whether removing old-value verification is accepted; if it is, immediately restore the original value. If you cannot restore it (unknown original password/email), stop after checking whether the response accepted the request; do not leave the user’s account with a new password or attacker-controlled email. Do not change the user’s email to test whether an old reset token remains valid.

- Change phone/email binding: remove old-password/old-number verification (restore if accepted).
- Change password: omit `oldPassword` (restore if accepted; if unable to restore, stop at the response).
- Carry reset through the final step and substitute someone else’s identity (change another person’s UID, not the user’s own password).
- Keep your own Cookie unchanged; substitute another person’s ID in `uid` inside rebinding/password/profile requests.
- If the password passed but 2FA did not, check whether the session can call rebinding, password-change, or business-write endpoints.
- If arbitrary login is all that remains: change a low-privilege ticket’s `role` / exchange for a higher-privilege ticket; access to a page you can already access does not count.

Single known account: change `uid` to an adjacent one or one copied from a list; count it only if it maps to another person, otherwise record “ID unavailable.” **Do not** grind registration/CAPTCHA to get a second account for comparison.

**Stop (testing complete, not idling):** no endpoint of this type (N/A + reason) or every probe in the table has been disproved.
**Idling (stop immediately):** CAPTCHA OCR, default-password dictionaries, spraying the same-shell form again, sending the four-part set to “please log in,” grinding registration for a second account.

#### 4.2.3 Change IDs / Map the Object Graph (No Session ≠ No ID Changes)

Changing IDs means changing any parameter that identifies a person, item, tenant, or file; **field names do not matter**. It can be one parameter or a complex set.
**Without an account: change IDs after unauthorized access works.** If business data is already available without login, change the ID and check whether another person’s data appears. If unauthorized access did not work, do not change IDs.
**With an account** (Cookie/token/demo account/JS key), follow the table with the session. The main work is authorization/logic, not continuing to send quotes to business endpoints. Run the four-part set on these **business parameters**.

| Test when present | N/A only if no entry point |
|-------------------|---------------------------|
| **Object graph:** list → detail → attachment/export/approval; change IDs at every layer (any field name) | Pure login wall; no object can be extracted |
| **Parameters selecting who/what:** adjacent ID / ID copied from a response / `0` / `-1` / empty / parameters selecting people in complex bodies | No changeable value in responses or lists (for a single known ID, still copy it; **do not** give up immediately) |
| Add `role` / `isAdmin` / `amount` / `status` / `vip` to **write endpoints** (try further values after one works; change only values that can be restored) | No write endpoints |
| Skip multistep workflow; race claim/inventory/coupon/approval requests | No such endpoint |

Do not grind registration to get a second account. If comparison data is missing, use adjacent IDs and IDs from responses. Details: `idor-test.md`, `logic-test.md`.

### 4.3 “Round Complete” on This Site = Minimum to Switch Sites

Do not wait for exhaustive testing of every possible surface before switching. It is okay to switch once all conditions are met:

1. The lines from §4.0 are written (or it is recorded as an empty shell only after confirming no business object can be extracted).
2. The endpoint list is built (default or fallback); the JS key inventory was checked, or “none” is written.
3. The matrix under §4.2 / §4.2.1 / **§4.2.2** / **§4.2.3 when a session exists** has all probes checked (if no entry, write N/A + reason, not vague wording; the four-part set applies to differential surfaces, not every path indiscriminately).
4. High-value endpoint parameters are covered (injection uses the parameter table on differential surfaces).
5. Successful findings are saved under `vuln-report-format` (no empty report for no finding).
6. No time was wasted on identical-shell mirrors/already covered ranges.
7. **Opens to a login page** (§4.1.1): business surface was found and unauthenticated tests were performed (or it was confirmed that no surface can be extracted and recorded as an empty shell). For the login form, test visible paths through success or disproof; do not grind.
8. **Same gate** (§4.1.2): unauthenticated exceptions were checked, the remaining endpoints return the same login code → close the four-part set. If §4.2.2 endpoints remain on the list, finish them before marking done. A sibling under the same product with no new exception after a quick differential check → record as checked; it does not mean live surfaces remain in the seed.
9. **Authentication endpoints** (§4.2.2, do not put this under §4.1.1): if the list contains session issuance/reset/rebinding/ticket replacement/2FA → finish probes or mark N/A + reason. An entry point with zero payloads is a missed test, not round completion.
10. **A Medium finding has been saved**, and the same object has not reached “write / see another person’s data / takeover / execution” → **extend the chain first; do not switch sites.** If a file channel works, pivot to business impact.
11. **Separate rich/thin surfaces within the seed:** for rich surfaces (lots of JS/backend/business JSON), reach the object graph + differential parameters (add §4.2.3 with a session); for thin surfaces (403 shells, same gate, same shell with no new path), a quick disproof is enough; do not spend 15 minutes or open a thread for a full matrix.

If there is a High report, write one proposed/update/do-not-add line from `hunt-iter` before switching (30 seconds; “do not add” also completes this). **Merging the short table is not a condition for switching sites**; the main agent can merge it later. No High finding = no proposal.
**Break through high-value leads before switching:** stable injection/RCE chain/cross-tenant access/sensitive unauthenticated read/usable credential (validated key + safe capability example, not just a copied string)/arbitrary login/account access/takeover through password or binding change — validate and report it or disprove it first.
**A Medium issue is not an instruction to switch sites:** extend the chain on the same object first.
**Required route change:** site unreachable, human/bot challenge is fatal with no alternative host, or purely junk shell — record and switch without asking. Seeing a login page is **not** a reason to switch.

### 4.4 Site-Level Cadence

1. Decide locked-scope or free-jump.
2. **Free-jump:** §1.0 + **§1.0.1 one-seed loop** (resume/start new → prepare many seeds → live-filter → search only this S → deduplicate/discard junk/dead targets → ownership gate → finish every remaining live surface → then search the next seed → when all done, random search/rescan).
3. **Locked scope:** follow multiple hosts in the asset cluster/subdomains in business flows/multiple paths on the same host; do not expand with FOFA.
4. Clarify the surface under §4.0 → extract JS (paths + keys) → endpoint list (add response clues as they appear) → allocate effort under `src-value` §1.1: without an account, change IDs once unauthorized access works, then test authentication endpoints/four-part set; with an account, change IDs/add fields with the session, then test the four-part set (including XSS) → complete this round.
5. Save findings under `vuln-report-format`; extend Medium issues on the same object first.
6. If reports have not grown for a long time, self-check under §2; quick-check thin shells and complete unchecked types on rich surfaces.

---

## 5. Heartbeat Self-Check (Run Each Round)

- [ ] Is this locked scope or free-jump? Did locked scope become too narrow and miss hosts, or did FOFA expand out of scope?
- [ ] Open `*src经验.md` beside the short table only if it exists on disk; absence is not a gap and does not mean the matrix is incomplete.
- [ ] Were as many seeds as possible prepared (business names + wholly owned entities levels 1–4 + root/registered domains), with no quantity limit? Was the list accidentally cut to a fixed count?
- [ ] Did a valuable surface from FOFA/on-site work lead to a **new registered root domain** being recycled into the seed queue (search only after this seed is finished, §1.1.1)?
- [ ] **One-seed closed loop (§1.0.1):** did you search one → deduplicate/discard junk/dead targets → finish the remainder → only then search the next? Did you search multiple seeds first?
- [ ] Is the queue persisted? Were pending/doing/done updated? On resumption, did you skip done/covered entries first?
- [ ] Were P2 seeds live-filtered first? Are dead registrations marked cold?
- [ ] Did you use Hummingbird for every item? Did a minority-owned company get treated as a group target?
- [ ] After everything was tested, did you **first** try random new business names? Did a rescan have evidence of additions first? After two `0 additions` rescans **this round**, did you switch to random? Did you create a rescan factory just to pad covered entries?
- [ ] Were untested paths on the same host accidentally dropped? Did the endpoint list/matrix advance?
- [ ] After a seed just became done, did you start the next one? Does the final response contain a question about whether to continue?
- [ ] When FOFA quota ran out, did you stop searching and work the stored pool? Are result counts being used as progress?
- [ ] **§2.1:** Are you still opening new-seed FOFA while live surfaces remain in this seed? Are you using mapping/search to pad progress before §4.3? Was a valuable term only recycled without changing course? Are multiple threads forming a dual-FOFA factory?
- [ ] In the last 15–20 minutes, did at least one advance: endpoint list / four-part payload on a differential surface / ID changes with a session / report? (Extracting JS or mapping objects counts on rich surfaces; no progress in any of the three while expanding surfaces means return to deep testing.)
- [ ] **Were injection parameters / SSRF URL parameters / XSS reflection and storage / RCE sinks tested under §4.2.1 on differential surfaces?** Have there been days of unauthorized reads only? Did you send `'` to every path and claim completion?
- [ ] When a login page appeared, did you find and test the business surface (same-host gateway or redirect host)? Did you grind on the login form? (§4.1.1)
- [ ] If session issuance/reset/rebinding/ticket replacement/2FA is on the list, was §4.2.2 completed or marked N/A? An entry point with no payload is a missed test; form grinding/default-password attacks are idling.
- [ ] With a session, did you test the §4.2.3 object graph / change IDs / add fields? Did you grind registration for a second account?
- [ ] Were IDs/URLs/tokens in responses added to the site list? If a list worked, were attachments/exports tested?
- [ ] Did JS extraction stop at paths, without checking salts/encrypted IDs/hidden routes/demo accounts?
- [ ] After a Medium finding, did you first extend the chain on the same object? Did you treat a thin shell like a rich surface and run a full matrix?
- [ ] Have business endpoints already reached the same gate (same login code, unauthenticated exceptions checked) but you are still opening full matrices on sibling hosts from the same product? (§4.1.2) If authentication endpoints remain untested, did you finish §4.2.2 first?
- [ ] Was the cadence constraint misread as a reason to test/search fewer things? If so, correct it under §2.1.0; capability is not capped.

---

## 6. In One Line

First decide locked scope or free-jump.
**Free-jump:** prepare as many seeds as possible (business names + wholly owned levels 1–4 + root/registered domains, **no quantity limit; dozens/hundreds are normal for a large SRC**) → persist the queue → resume old folders and skip hosts already searched/tested → **one-seed closed loop (§1.0.1): search one seed → deduplicate/discard junk/dead targets → finish the rest → only then search the next** (live-filter P2 first; **never search multiple seeds first**) → verify 100% ownership (Hummingbird only if uncertain) → **full on-site matrix (deep-testing priority §2.1)** → **recycle valuable surfaces without changing course (§1.1.1)** → finish the remaining surfaces in the seed and immediately take the next entry (do not ask to continue) → when all are done, search randomly or rescan for additions.
**Locked scope:** follow all hosts in the asset cluster/business flow; do not expand with FOFA.
Save confirmed findings under `vuln-report-format`; self-correct idling.
**Opens to a login page:** find the business surface first; main work is unauthenticated testing; test visible login-form paths through success or disproof, then stop. A login page is not a reason to change assets (§4.1.1). If all business endpoints use the same login code and unauthenticated exceptions were checked → stop at the same gate, quickly compare siblings, and do not start another full matrix (§4.1.2).
**Authentication endpoints** (session issuance / reset / rebinding / ticket replacement / 2FA): if an entry exists, finish §4.2.2; otherwise N/A. Do not grind on the form shell. If you log in or change another person’s password/binding, save under the report format.
**Site-entry workflow:** clarify scope under §4.0; JS extraction is for keys, not only paths; add response clues to the current site’s queue. Allocate effort under `src-value` §1.1: without an account, change IDs once unauthorized access works; with an account, change IDs with the session, then test the four-part set (including XSS, with full effort). Extend Medium issues on the same object first. Deep-test rich surfaces; quick-check thin shells.
**Test injection/SSRF/XSS/RCE under §4.2.1 on differential surfaces; do not leave these untested for days.** Give all four equal urgency; choose probes by stack; do not pile up unauthorized reads or send `'` to every path and call it done.
**Cadence mantra:** search one seed → clean it → finish its remaining live surfaces → only then run FOFA for another seed; within a seed, build the list before the matrix, finish the matrix before switching sites; when rate-limited, work the pool. These cadence rules prevent expansion-only idling; they **do not cut** seed breadth, test depth, or ability to break through.