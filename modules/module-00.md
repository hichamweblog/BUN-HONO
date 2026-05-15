# Module 00 — Backend & HTTP Fundamentals

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Foundational
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [README / Course Roadmap](../README.md)
→ Next: [Module 01 — Bun Fundamentals](./module-01.md)

---

## What You'll Learn

- What a backend actually is and what it does
- How HTTP works — the protocol that powers the entire web
- The anatomy of a request and response
- HTTP methods, status codes, and headers — in depth
- What a web server is and how it listens for connections
- The TCP/IP model simplified for backend engineers
- REST API principles
- How all of this maps to Hono + Bun

---

## A. Mental Model — What IS a Backend?

Before touching a single line of code, let's build the right mental model.

### Frontend vs Backend — The Boundary

Every web application has two sides:

- **Frontend (client-side)**: What the user sees and interacts with — HTML, CSS, JavaScript running in the browser. It sends requests.
- **Backend (server-side)**: The invisible engine — it receives requests, processes data, talks to databases, enforces rules, and sends responses back.

The **boundary** between them is HTTP. The frontend sends an HTTP request; the backend returns an HTTP response. They never share memory, files, or state directly.

### What is a Server?

A "server" is just **a program that listens for incoming network connections on a specific port and responds to them**. That's it. No magic.

When you run `bun run src/index.ts`, Bun starts a process that binds to port 3000 and waits. When a request arrives at `localhost:3000`, your Hono code processes it and sends back a response.

> 💡 A "server" is not a special type of computer. It's a regular program. Your laptop becomes a server the moment it listens on a port.

### The Restaurant Analogy

Imagine a restaurant:

```
Customer (Browser / Client)
       │
       │  "I'd like a cheeseburger" (HTTP Request)
       ▼
  Waiter (HTTP Server / Your API)
       │
       │  Takes the order, goes to the kitchen
       ▼
   Kitchen (Business Logic / Service Layer)
       │
       │  Prepares the food using ingredients
       ▼
  Pantry/Fridge (Database)
       │
       │  Retrieves ingredients → cooks → plates
       ▼
   Waiter (Returns Response)
       │
       │  "Here's your cheeseburger" (HTTP Response)
       ▼
  Customer receives food (JSON data rendered in browser)
```

The **backend** is everything that happens AFTER the customer places an order — the waiter, the kitchen, the pantry. The customer only sees the final plate (JSON response).

### Key Insight

> The backend's job is to **receive requests, process them, and return responses**.
> Everything else — databases, auth, caching, validation — serves that core loop.

---

## B. Visual Architecture — How the Internet Connects You to a Server

```
Your Browser                     The Internet                   Your Server
─────────────────────────────────────────────────────────────────────────────
                                                               
[https://api.example.com/users]
       │
       │ 1. DNS Lookup
       │    "What IP is api.example.com?"
       │    DNS: "It's 192.168.1.100"
       │
       ▼
[TCP Connection]
       │
       │ 2. Three-Way Handshake
       │    Client: SYN →
       │    Server:         ← SYN-ACK
       │    Client: ACK →
       │    (Connection established)
       │
       ▼
[TLS Handshake] (for HTTPS)
       │
       │ 3. Encrypted channel negotiated
       │    (Certificate exchange, key agreement)
       │
       ▼
[HTTP Request Sent]
       │
       │ 4. Raw bytes travel across the internet
       │    GET /users HTTP/1.1
       │    Host: api.example.com
       │    Authorization: Bearer eyJ...
       │
       ▼
                                                    [Your Bun + Hono Server]
                                                           │
                                                           │ 5. Receives bytes
                                                           │ 6. Parses HTTP
                                                           │ 7. Routes request
                                                           │ 8. Runs handler
                                                           │ 9. Queries DB
                                                           │ 10. Builds response
                                                           ▼
                                                    [HTTP Response]
                                                           │
       ◄──────────────────────────────────────────────────┘
       │
       │ HTTP/1.1 200 OK
       │ Content-Type: application/json
       │
       │ {"users": [...]}
       │
       ▼
[Browser renders data]
```

This entire flow — from DNS lookup to JSON response — takes **milliseconds** in a well-optimized system.

---

## C. HTTP Deep Dive — The Protocol of the Web

### What is HTTP?

**HTTP (HyperText Transfer Protocol)** is a text-based, stateless request-response protocol.

- **Text-based**: Messages are human-readable text (before HTTP/2's binary framing)
- **Stateless**: Each request is completely independent — the server has no memory of previous requests
- **Request-Response**: One side always initiates, the other always responds

> **Stateless is crucial.** This is why we need tokens, sessions, and cookies — to artificially create "state" on top of a stateless protocol.

---

### The Anatomy of an HTTP Request

Every HTTP request has exactly this structure:

```
┌─────────────────────────────────────────────────┐
│  REQUEST LINE                                   │
│  GET /api/users?page=1&limit=10 HTTP/1.1        │
│  ─── ──────────────────────── ────────          │
│  Method     Path + Query String   Version       │
├─────────────────────────────────────────────────┤
│  HEADERS (key-value metadata)                   │
│  Host: api.example.com                          │
│  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9... │
│  Content-Type: application/json                 │
│  Accept: application/json                       │
│  User-Agent: Mozilla/5.0 ...                    │
├─────────────────────────────────────────────────┤
│  BLANK LINE (required separator)                │
│                                                 │
├─────────────────────────────────────────────────┤
│  BODY (optional, used in POST/PUT/PATCH)        │
│  {                                              │
│    "name": "Alice",                             │
│    "email": "alice@example.com"                 │
│  }                                              │
└─────────────────────────────────────────────────┘
```

**When does a request have a body?**
- `GET`, `DELETE` — No body (data goes in URL)
- `POST`, `PUT`, `PATCH` — Body contains data

---

### The Anatomy of an HTTP Response

```
┌─────────────────────────────────────────────────┐
│  STATUS LINE                                    │
│  HTTP/1.1 200 OK                                │
│  ──────── ─── ──                                │
│  Version  Code Reason                           │
├─────────────────────────────────────────────────┤
│  HEADERS                                        │
│  Content-Type: application/json                 │
│  Content-Length: 234                            │
│  Cache-Control: no-cache                        │
│  X-Request-Id: 550e8400-e29b-41d4-a716-...      │
├─────────────────────────────────────────────────┤
│  BLANK LINE                                     │
│                                                 │
├─────────────────────────────────────────────────┤
│  BODY                                           │
│  {                                              │
│    "id": 1,                                     │
│    "name": "Alice",                             │
│    "email": "alice@example.com"                 │
│  }                                              │
└─────────────────────────────────────────────────┘
```

---

## D. HTTP Methods — The Verbs of the Web

HTTP methods describe **what you want to do** with a resource.

| Method   | Purpose                          | Has Body? | Idempotent? | Safe? |
|----------|----------------------------------|-----------|-------------|-------|
| `GET`    | Retrieve a resource              | No        | ✅ Yes      | ✅ Yes |
| `POST`   | Create a new resource            | Yes       | ❌ No       | ❌ No  |
| `PUT`    | Replace a resource entirely      | Yes       | ✅ Yes      | ❌ No  |
| `PATCH`  | Partially update a resource      | Yes       | ❌ No       | ❌ No  |
| `DELETE` | Delete a resource                | Rare*     | ✅ Yes      | ❌ No  |
| `HEAD`   | Same as GET but no body returned | No        | ✅ Yes      | ✅ Yes |
| `OPTIONS`| Ask what methods are allowed     | No        | ✅ Yes      | ✅ Yes |

> \* **Note on DELETE bodies:** The HTTP spec (RFC 9110) does not forbid a body on DELETE requests, and some APIs use it (e.g., bulk delete with a list of IDs). However, most APIs put identifiers in the URL path instead. Know that both approaches exist.

**Idempotent** means: calling it multiple times has the same effect as calling it once.
- `DELETE /users/1` — Deleting the same user 10 times = user is gone (same result)
- `POST /users` — Creating a user 10 times = 10 users (different result each time)

**Safe** means: the operation doesn't modify server state.

> **🚨 Production Mistake Alert**: Many developers misuse HTTP methods. Using `GET` to delete data or `POST` for everything is an anti-pattern. Use methods semantically — it's what makes APIs predictable and RESTful.

---

## E. HTTP Status Codes — What Did the Server Actually Do?

Status codes tell the client what happened. They're grouped into ranges:

### 2xx — Success

| Code | Name | When to Use |
|------|------|-------------|
| `200` | OK | Generic success — GET, PUT, PATCH |
| `201` | Created | Resource was successfully created — POST |
| `204` | No Content | Success but nothing to return — DELETE |

### 3xx — Redirection

| Code | Name | When to Use |
|------|------|-------------|
| `301` | Moved Permanently | Resource URL changed forever |
| `302` | Found | Temporary redirect |
| `304` | Not Modified | Cached version is still valid |

### 4xx — Client Errors (YOU messed up)

| Code | Name | When to Use |
|------|------|-------------|
| `400` | Bad Request | Syntactically malformed request (invalid JSON, missing required header) |
| `401` | Unauthorized | Not authenticated (no/invalid token) |
| `403` | Forbidden | Authenticated but not authorized |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Conflict with existing state (e.g., duplicate email) |
| `422` | Unprocessable Entity | Request is well-formed but semantically invalid (e.g., `"age": -5`) |
| `429` | Too Many Requests | Rate limit exceeded |

> 💡 **400 vs 422**: `400` means the server can't even *parse* the request (broken JSON, wrong content type). `422` means the server parsed it fine but the *data itself* is invalid (failed business rules, schema validation). Many teams use `400` for both — just be consistent within your API.

### 5xx — Server Errors (WE messed up)

| Code | Name | When to Use |
|------|------|-------------|
| `500` | Internal Server Error | Unexpected server crash |
| `502` | Bad Gateway | Upstream service failed |
| `503` | Service Unavailable | Server is overloaded or in maintenance |

> **Senior Engineer Note:** 401 vs 403 is a common confusion point.
> - `401` = "I don't know who you are" (authenticate first)
> - `403` = "I know who you are, but you can't do this" (wrong permissions)

---

## F. HTTP Headers — Metadata That Controls Everything

Headers are **key-value pairs** that provide metadata about the request or response.

### Common Request Headers

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
  └── How the client proves identity

Content-Type: application/json
  └── Format of the request body

Accept: application/json
  └── What format the client can handle

User-Agent: Mozilla/5.0 (...)
  └── What browser/client is making the request

X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
  └── Unique ID to trace this request through your system
```

### Common Response Headers

```
Content-Type: application/json; charset=utf-8
  └── Format of the response body

Content-Length: 1234
  └── Size in bytes (important for performance)

Cache-Control: max-age=3600, must-revalidate
  └── How long to cache this response

Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
  └── Tells the browser to store a cookie

X-Rate-Limit-Remaining: 45
  └── Custom header (APIs often add these)
```

> **Security Note:** HTTP headers are a major attack surface. We'll cover `Strict-Transport-Security`, `Content-Security-Policy`, `X-Frame-Options` and more in Module 13.

---

## G. URLs Dissected

```
https://api.example.com:443/api/v1/users/42?include=posts&page=1#section

│     │ │               │  │             │  │                 │ │      │
│     │ │               │  │             │  └─ Query String   │ └─Hash │
│     │ │               │  │             └──── Path           │        │
│     │ │               │  └────────────────── Base Path      │        │
│     │ │               └─────────────────── Port (default 443 for HTTPS)
│     │ └─────────────────────────────────── Host / Domain
│     └───────────────────────────────────── Scheme / Protocol
└─────────────────────────────────────────── Full URL
```

Key parts:
- **Scheme**: `http` (unencrypted) or `https` (TLS encrypted)
- **Host**: The domain or IP address
- **Port**: 80 for HTTP, 443 for HTTPS (usually omitted when default)
- **Path**: The resource location (what Hono routes match against)
- **Query String**: Optional key-value pairs after `?`, separated by `&`
- **Fragment**: `#hash` — browser only, never sent to the server

> **This matters for Hono**: When you define `app.get('/api/v1/users/:id', ...)`, Hono matches the **path** part of the URL and gives you the `:id` as a route parameter.

---

## H. REST — The Rules We Agreed On

**REST (Representational State Transfer)** is an architectural style — not a standard, not a protocol. It's a set of conventions that make APIs predictable and interoperable.

### The 6 Constraints of REST

1. **Stateless** — Each request must carry all the information the server needs to process it. The server doesn't rely on stored context from previous requests. (You *can* have server-side sessions, but each request must still identify itself — e.g., via a session cookie.)
2. **Client-Server** — Frontend and backend are separate concerns with independent evolution
3. **Cacheable** — Responses should declare whether they can be cached
4. **Uniform Interface** — Consistent resource naming and HTTP verb usage
5. **Layered System** — Client doesn't know if it's talking to a load balancer, CDN, or app server
6. **Code on Demand** (optional) — Server can send executable code (e.g., JavaScript)

### RESTful Resource Design

Think in **nouns**, not verbs. The HTTP method IS the verb.

```
❌ Bad REST Design (verb-based)
GET    /getUsers
POST   /createUser
DELETE /deleteUser?id=1
GET    /getUserPosts?userId=5

✅ Good REST Design (noun-based)
GET    /users              → List all users
POST   /users              → Create a user
GET    /users/:id          → Get one user
PUT    /users/:id          → Replace a user
PATCH  /users/:id          → Update part of a user
DELETE /users/:id          → Delete a user
GET    /users/:id/posts    → Get this user's posts (nested resource)
```

### API Versioning Convention

Always version your APIs. Your clients depend on your contracts.

```
/api/v1/users   ← Version 1
/api/v2/users   ← Version 2 (when you have breaking changes)
```

---

## I. The Full Request Lifecycle in a Hono App

This is the mental model you'll carry through the entire course:

> ⚠️ **How Hono actually works:** Hono uses a trie-based router. When a request arrives, it first matches the route (fast tree lookup), then runs middleware and the handler as a combined chain. Global middleware (registered via `app.use()`) runs first, then route-specific handlers. The diagram below shows the *logical* stages your request passes through.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Client (Browser / Mobile / curl / Postman)                         │
│                                                                     │
│  POST /api/v1/users                                                 │
│  Authorization: Bearer eyJ...                                       │
│  Content-Type: application/json                                     │
│  {"name": "Alice", "email": "alice@example.com"}                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │ TCP + TLS
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Bun HTTP Server                                                    │
│  • Listens on port 3000                                             │
│  • Accepts TCP connections                                          │
│  • Parses raw bytes → Request object (Web Standard API)             │
│  • Calls: export default { fetch: app.fetch }                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Hono App.fetch(request)                                            │
│                                                                     │
│  Step 1: ROUTE MATCHING (trie lookup — extremely fast)              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  "POST /api/v1/users" → finds matching handler chain         │  │
│  │  Collects: global middleware + route middleware + handler     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Step 2: MIDDLEWARE CHAIN (runs in registration order)              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Logger Middleware        → logs: "POST /api/v1/users"        │  │
│  │  CORS Middleware          → sets CORS headers                 │  │
│  │  Request ID Middleware    → attaches unique request ID        │  │
│  │  Rate Limiter Middleware  → checks: has this IP hit limit?    │  │
│  │  Auth Middleware          → validates Bearer token, sets user │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Step 3: VALIDATION (inside handler or via validator middleware)    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Zod schema validates body                                   │  │
│  │  → name: string (min 2 chars) ✅                             │  │
│  │  → email: valid email format ✅                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Step 4: ROUTE HANDLER / CONTROLLER                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  createUserHandler(c)                                        │  │
│  │  → calls userService.createUser(data)                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Step 5: SERVICE LAYER                                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  userService.createUser(data)                                │  │
│  │  → hashes password                                           │  │
│  │  → checks for duplicate email                                │  │
│  │  → calls userRepository.insert(user)                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Step 6: DATA LAYER                                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Drizzle ORM generates SQL:                                  │  │
│  │  INSERT INTO users (name, email) VALUES ('Alice', '...')     │  │
│  │  RETURNING id, name, email, created_at                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Step 7: RESPONSE ASSEMBLY                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  return c.json({ data: newUser }, 201)                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  HTTP Response                                                      │
│                                                                     │
│  HTTP/1.1 201 Created                                               │
│  Content-Type: application/json                                     │
│  X-Request-ID: 550e8400-...                                         │
│                                                                     │
│  {"data": {"id": 1, "name": "Alice", "email": "alice@example.com"}} │
└─────────────────────────────────────────────────────────────────────┘
```

Study this diagram carefully. You'll revisit it in every module.

---

## J. TCP/IP — What Happens at the Network Level

You don't need to be a networking expert, but understanding the basics prevents confusion.

```
Application Layer     ← HTTP lives here (what we care about)
      ↓↑
Transport Layer       ← TCP ensures ordered, reliable delivery
      ↓↑
Internet Layer        ← IP routes packets across networks
      ↓↑
Link / Access Layer   ← Physical bits over cables/wifi (Ethernet, Wi-Fi)
```

**Why TCP matters for backend engineers:**
- TCP guarantees delivery and order — HTTP relies on this
- Each HTTP request = a TCP connection (or reuses one via HTTP keep-alive)
- Connection establishment takes ~1 RTT — this is latency you optimize around
- HTTP/2 multiplexes multiple requests over one TCP connection (more efficient)
- HTTP/3 uses QUIC (UDP-based) for even less latency

> **TLS note**: HTTPS adds TLS encryption between TCP and HTTP. Bun handles TLS natively — you just pass a certificate.

---

## K. Synchronous vs Asynchronous I/O — The Foundation of Performance

This is one of the most important backend engineering concepts.

### The Naive Approach (Synchronous / Blocking)

```txt
(pseudo-code — conceptual, not real API)
Thread 1: Handle Request A
  → Waits for DB query (100ms)  ← BLOCKED, doing nothing
  → Waits for file read (50ms)  ← BLOCKED, still nothing
  → Builds response

While Thread 1 is blocked, Request B just waits...
```

To handle multiple requests simultaneously, this model needs **multiple threads** — expensive in memory and CPU context switching.

### The Modern Approach (Asynchronous / Non-Blocking)

```
Event Loop:
  → Start handling Request A
  → Initiate DB query (non-blocking) → "call me when done"
  → Start handling Request B (while waiting for Request A's DB)
  → Start handling Request C
  → DB for Request A responds → resume Request A
  → Build response for A
  → Build response for B and C
```

One thread, hundreds of concurrent requests. This is how Node.js and Bun work.

**In code (conceptual):**
```typescript
// BLOCKING — avoid this in a server (pseudo-code to illustrate the concept)
const user = db.querySync('SELECT * FROM users WHERE id = 1') // blocks the event loop

// NON-BLOCKING — the correct approach in Bun/Node.js
const user = await db.query('SELECT * FROM users WHERE id = 1') // yields control back
// While waiting for the DB, Bun can handle other requests on the same thread
```

**Bun's advantage**: Bun uses JavaScriptCore (Safari's JS engine) instead of V8 (Chrome's engine used by Node.js). JSC generally provides faster startup times and lower baseline memory usage. Async I/O performance is comparable — the real-world difference depends on your workload.

---

## L. JSON — The Language of APIs

Modern APIs primarily communicate in **JSON (JavaScript Object Notation)**.

```json
{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin",
  "createdAt": "2024-01-15T09:30:00Z",
  "profile": {
    "bio": "Backend engineer",
    "avatar": "https://cdn.example.com/avatars/alice.jpg"
  },
  "tags": ["typescript", "hono", "bun"]
}
```

**JSON rules:**
- Keys are always double-quoted strings
- Values can be: string, number, boolean, null, array, or object
- No trailing commas
- No comments

**⚠️ JSON gotchas for backend engineers:**
- **No Date type** — dates are serialized as ISO 8601 strings (e.g., `"2024-01-15T09:30:00Z"`). You must parse them back on the client.
- **No `undefined`** — only `null` exists. Properties set to `undefined` are omitted during serialization.
- **BigInt breaks** — `JSON.stringify(42n)` throws an error. You must convert to string first.
- **Precision loss** — Numbers beyond `Number.MAX_SAFE_INTEGER` lose precision. IDs from databases sometimes hit this limit.

**In Hono**, you'll primarily use `c.json()` to return JSON responses and `await c.req.json()` to parse request bodies. Hono handles serialization/deserialization internally.

---

## M. Best Practices — Your First Engineering Habits

Start forming these habits now:

✅ **Always return consistent response shapes.** Pick a format and stick with it:

```typescript
// Success
{ "data": { ... }, "error": null }

// Failure
{ "data": null, "error": { "code": "NOT_FOUND", "message": "User not found" } }
```

> 💡 Note: the JSON above uses double-quoted keys because that's valid JSON. In TypeScript code, you write object literals with unquoted keys — TypeScript/JavaScript handles the serialization when you call `c.json()`.

✅ **Always use proper HTTP status codes** — not just 200 for everything

✅ **Always validate input** — never trust client data

✅ **Always handle errors** — unhandled promise rejections crash servers

✅ **Always log requests** — you can't debug what you can't see

✅ **Always version your API** — `/api/v1/...`

✅ **Never expose internal errors to clients:**
- ❌ `"Error: column 'pasword' doesn't exist in table users"`
- ✅ `"Internal server error"`

✅ **Treat all user input as untrusted** — query params, path params, headers, body

---

## N. Common Mistakes & Debugging

### Mistake 1: Using 200 for Everything

```
❌ return c.json({ error: "User not found" }, 200)
✅ return c.json({ error: "User not found" }, 404)
```

HTTP clients (including your frontend) check status codes. Return the right one.

### Mistake 2: Not Handling async Errors

If a function can throw (and database calls almost always can), you need to either `await` it inside a `try/catch` or let Hono's global error handler catch it. The key difference is `async` — when you mark a handler `async`, Hono will catch rejected promises for you. But explicit error handling is still better practice.

```typescript
// ❌ Dangerous — if db.getUsers() returns a rejected Promise,
//    and the handler isn't async, the rejection is unhandled
app.get('/users', (c) => {
  const users = db.getUsers() // this is a Promise, not awaited!
  return c.json(users) // returns the Promise object, not the data
})

// ✅ Correct — async handler with proper await and error handling
app.get('/users', async (c) => {
  try {
    const users = await db.getUsers() // awaits the Promise
    return c.json(users)              // returns actual data
  } catch (error) {
    return c.json({ error: 'Failed to fetch users' }, 500)
  }
})
```

### Mistake 3: Exposing Stack Traces in Production

```typescript
// ❌ NEVER do this in production
catch (error) {
  return c.json({ error: error.stack }, 500)
}

// ✅ Log internally, return safe message
catch (error) {
  console.error('Database error:', error) // goes to your logs
  return c.json({ error: 'Internal server error' }, 500) // safe for client
}
```

### Mistake 4: Confusing 401 and 403

- No auth token provided → `401 Unauthorized`
- Invalid auth token → `401 Unauthorized`
- Valid token, but wrong role → `403 Forbidden`
- Resource exists but belongs to another user → `403 Forbidden`

---

## O. 🧠 Comprehension Check — Think Before Moving On

Before jumping to Module 01, answer these in your head (or write them down):

1. What is the difference between a **stateless** and **stateful** protocol?
2. When should you use `POST` vs `PUT` vs `PATCH`?
3. What's the difference between `401` and `403`?
4. Why do modern backend servers use **async I/O** instead of threads?
5. What's the difference between a **path parameter** (`/users/:id`) and a **query parameter** (`/users?id=1`)? When do you use each?
6. Why should you never expose error stack traces to clients?
7. What does **idempotent** mean? Give an example of an idempotent HTTP operation.

---

## P. Exercises

### Exercise 1 — Map Real Endpoints

Design a RESTful API for a **Blog** system. Define endpoints for:
- Users (CRUD)
- Posts (CRUD)
- Comments on a post

For **each** endpoint, write down:
- The HTTP method
- The URL path
- What status code a successful response returns
- Whether the request has a body

**Rubric** (check yourself after):
- Did you use nouns, not verbs, in your paths?
- Did you use `201` for creation, `204` for deletion?
- Did you nest comments under posts (`/posts/:id/comments`)?
- Did you avoid putting verbs like `/deleteUser` in URLs?

Try it yourself first. Don't look at solutions yet.

---

### Exercise 2 — Analyze Real HTTP Traffic

1. Open your browser DevTools → Network tab
2. Visit any website (try https://jsonplaceholder.typicode.com/users)
3. Find an HTTP request and identify:
   - The method
   - The URL (path, query string)
   - The request headers
   - The response status code
   - The response headers
   - The response body format

---

### Mini Challenge — curl Exploration

If you have curl installed (you likely do on Linux):

```bash
# Make a GET request and see the full response including headers
curl -v https://jsonplaceholder.typicode.com/users/1

# Make a POST request with JSON body
curl -v -X POST https://jsonplaceholder.typicode.com/posts \
  -H "Content-Type: application/json" \
  -d '{"title": "My Post", "body": "Hello World", "userId": 1}'

# What status code did you get?
# What headers were in the response?
# What was in the response body?
```

---

### Exercise 3 — Design an Error Response

You're building an API endpoint `POST /api/v1/users` that creates a user. Write out the exact HTTP response (status line, headers, and JSON body) for each of these scenarios:

1. **Success** — the user was created
2. **Validation failure** — the email field is missing
3. **Conflict** — a user with that email already exists
4. **Server error** — the database is down

For each, decide: What status code? What response body shape? What headers?

---

## Q. Production Notes — What Senior Engineers Think About

When you're building real production APIs, these HTTP fundamentals translate to concrete decisions:

**Response Time SLAs**: Knowing the request lifecycle helps you find where time is spent.
- Network: ~50-200ms (you can't control this)
- DB queries: often 5-50ms (optimize with indexes)
- Your code: should be <5ms (usually)

**Content Negotiation**: Clients can request specific formats via `Accept` header. Your server can respond differently based on it.

**Idempotency Keys**: For payment APIs, you pass a unique key with POST requests so retries don't double-charge. Built on HTTP idempotency concepts.

**HTTP Caching**: `Cache-Control`, `ETag`, `Last-Modified` headers let browsers and CDNs cache your responses. Understanding HTTP is the foundation of web performance.

---

## R. Glossary

Terms introduced in this module. Refer back here when you encounter them later.

| Term | Definition |
|------|------------|
| **API** | Application Programming Interface — a contract that defines how two systems communicate |
| **Async I/O** | Input/Output operations that don't block the thread while waiting for completion |
| **CRUD** | Create, Read, Update, Delete — the four basic operations on data |
| **DNS** | Domain Name System — translates domain names (e.g., `example.com`) to IP addresses |
| **Endpoint** | A specific URL path + HTTP method combination that your API responds to |
| **Event Loop** | The mechanism that allows single-threaded runtimes (Bun, Node.js) to handle concurrent I/O |
| **HTTP** | HyperText Transfer Protocol — the application-layer protocol for web communication |
| **HTTPS** | HTTP over TLS — encrypted HTTP |
| **Idempotent** | An operation that produces the same result whether called once or multiple times |
| **JSON** | JavaScript Object Notation — a text-based data interchange format |
| **REST** | Representational State Transfer — an architectural style for designing web APIs |
| **RTT** | Round-Trip Time — the time for a packet to travel from client to server and back |
| **SLA** | Service Level Agreement — a commitment to specific performance metrics (e.g., 99.9% uptime) |
| **Stateless** | A protocol where each request is independent — the server retains no memory of previous requests |
| **TCP** | Transmission Control Protocol — ensures reliable, ordered data delivery |
| **TLS** | Transport Layer Security — encryption layer between TCP and HTTP |
| **URL** | Uniform Resource Locator — the full address used to access a resource |

---

## Summary

You now understand:
- ✅ What a backend server is and what it does
- ✅ The boundary between frontend and backend
- ✅ What a "server" literally is — a program listening on a port
- ✅ The full request-response lifecycle (DNS → TCP → TLS → HTTP → Response)
- ✅ HTTP methods and when to use each (including idempotency and safety)
- ✅ HTTP status codes — the right ones for the right situations
- ✅ Request and response anatomy (request line, headers, body)
- ✅ URLs dissected — scheme, host, port, path, query, fragment
- ✅ REST principles and the 6 constraints
- ✅ Async I/O — why it matters for performance
- ✅ JSON as the API communication format (and its gotchas)
- ✅ Core backend engineering habits from day one

---

## Navigation

← Previous: [README / Course Roadmap](../README.md)
→ Next: [Module 01 — Bun Fundamentals](./module-01.md)

---

> **Mentor Note**: Don't rush forward until you can explain HTTP to someone else.
> The engineers who understand fundamentals deeply are the ones who debug fast
> and design systems that last. The next module dives into Bun — your runtime.
