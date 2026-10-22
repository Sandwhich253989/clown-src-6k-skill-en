# JavaScript Reverse Engineering for API Discovery

> Mandatory on-site steps are in `dig-scope` §4.1: **do not extract only `/api/` paths**. Record salts, encrypted IDs, public keys, hidden/admin routes, and hard-coded demo accounts when present; otherwise record “none.” Treat demo accounts as keys, not as a login-form password dictionary.

## Use cases

- The page API has encrypted parameters and cannot be replayed directly with curl
- Find hidden APIs in front-end JavaScript
- Understand signature/token generation to construct requests
- Hidden/admin routes, webpack async chunks, and hard-coded demo accounts / test tenants in the route table

---

## Workflow 1: Discover APIs

### 1. Inspect network requests

Use the js-reverse MCP tool when the browser has the target page open:

```
Action: list_network_requests()
Filter: resourceTypes=["xhr", "fetch"]
Look for endpoints containing user data (/user/, /api/, /order/, /account/)
```

### 2. Extract API paths in bulk from JavaScript source

```javascript
// Run in evaluate_script to extract all XHR paths from the page
() => {
  const scripts = Array.from(document.querySelectorAll('script[src]'))
    .map(s => s.src);
  return scripts;
}
```

```bash
# Download all JavaScript files and grep for API paths
for url in $(cat js_files.txt); do
  curl -s "$url" | grep -oP '"(/api/[^"]+)"' | tr -d '"'
done | sort -u > discovered_apis.txt

# Search for keywords
grep -E "(userId|uid|token|sign|order|payment)" discovered_apis.txt
```

### 3. Search with search_in_sources

```
search_in_sources("userId")          // Find endpoints related to user IDs
search_in_sources("/api/")           // Find all API paths
search_in_sources("Authorization")   // Find where the token is set
search_in_sources("signature")       // Find signature parameters
```

---

## Workflow 2: Analyze encrypted parameters

Use this workflow when requests contain encrypted parameters such as `sign` / `_token` / `x-sign`.

### Step 1: Set an XHR breakpoint

```
1. break_on_xhr("/api/target-endpoint")
2. Trigger the corresponding action on the page
3. When execution pauses, call: get_paused_info()
4. Inspect the call stack and find the function that sets the encrypted parameter
```

### Step 2: Analyze the call stack

```
Example result from get_paused_info():
  Frame 0: setRequestHeader (XMLHttpRequest)
  Frame 1: signRequest (utils.js:342)     ← Focus here
  Frame 2: sendApiRequest (api.js:89)
  Frame 3: onClick (page.js:234)
```

Locate Frame 1 and read its source:

```
get_script_source(url="utils.js", startLine=335, endLine=355)
```

### Step 3: Extract the signing logic

Common signature patterns:

```javascript
// Pattern 1: sort parameters + MD5
function signRequest(params) {
  const sorted = Object.keys(params).sort().map(k => `${k}=${params[k]}`).join('&');
  return md5(sorted + SECRET_KEY);
}

// Pattern 2: timestamp + nonce + HMAC
function sign(data) {
  const ts = Date.now();
  const nonce = Math.random().toString(36).substr(2);
  return hmacSha256(ts + nonce + JSON.stringify(data), APP_SECRET);
}

// Pattern 3: append a fixed salt
const sign = md5(userId + ':' + timestamp + ':' + SALT);
```

### Step 4: Run the signing function in the browser

```javascript
// Call the page signing function directly with evaluate_script
() => {
  // If the function is in global scope
  return window.signRequest({userId: "victim_id", action: "getInfo"});
}
```

### Step 5: Reproduce the signature in Python

```python
import hashlib
import hmac
import time
import random
import string

# Reproduce an MD5 signature
def sign_request(params: dict, secret_key: str) -> str:
    sorted_str = '&'.join(f"{k}={params[k]}" for k in sorted(params.keys()))
    return hashlib.md5((sorted_str + secret_key).encode()).hexdigest()

# Reproduce an HMAC-SHA256 signature
def sign_hmac(data: str, app_secret: str) -> str:
    ts = str(int(time.time() * 1000))
    nonce = ''.join(random.choices(string.ascii_lowercase, k=8))
    msg = ts + nonce + data
    return hmac.new(app_secret.encode(), msg.encode(), hashlib.sha256).hexdigest()

# Verify that the Python result matches the browser JavaScript result
```

---

## Workflow 3: Find hidden APIs

### Extract from Webpack chunks

```bash
# Find chunk files
curl -s "https://target.com" | grep -oP 'chunk\.[a-z0-9]+\.js'

# Download all chunks
for chunk in $(curl -s "https://target.com" | grep -oP '"/static/js/[^"]+\.js"' | tr -d '"'); do
  curl -s "https://target.com$chunk" >> all_js.txt
done

# Extract paths
grep -oP '"(/[a-z]+){1,5}"' all_js.txt | sort -u | grep -v node_modules
```

### Extract from route configuration

```bash
# Vue/React route configuration
grep -oP 'path:\s*["\x27][^"'\'']+' all_js.txt
grep -oP '"route":\s*["\x27][^"'\'']+' all_js.txt

# Axios base URL
grep -oP 'baseURL:\s*["\x27][^"'\'']+' all_js.txt
grep -oP 'BASE_API\s*=\s*["\x27][^"'\'']+' all_js.txt
```

---

## Workflow 4: Enumerate API parameters

After discovering an endpoint, enumerate parameters to find authorization / injection issues:

```python
import requests

# Test each discovered endpoint
discovered_apis = [
    "/api/v1/user/info",
    "/api/v1/order/list",
    "/api/v2/account/profile",
]

session = requests.Session()
session.headers.update({"Authorization": "Bearer YOUR_TOKEN"})

for api in discovered_apis:
    r = session.get(f"https://target.com{api}")
    print(f"[{r.status_code}] {api} - {len(r.text)} bytes")
    if r.status_code == 200:
        # Record ID fields in the response for later authorization tests
        data = r.json()
        print(f"  Response fields: {list(data.keys()) if isinstance(data, dict) else 'array'}")
```
