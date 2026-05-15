# Module 04 — Request & Response

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Intermediate
> **Duration:** ~2 hours

---

## Navigation

← Previous: [Module 03 — Routing](./module-03.md)
→ Next: [Module 05 — Middleware](./module-05.md)

---

## What You'll Learn

- The `HonoRequest` object in depth
- Parsing request bodies (JSON, form data, raw)
- The full Response API — every helper method
- Setting headers, cookies, and status codes
- Streaming responses
- Content negotiation
- The raw Web Standard objects underneath

---

## A. Mental Model — Request/Response as Envelopes

```
Request Envelope (what the client sends):
┌────────────────────────────────────────────┐
│  TO:   POST /api/users                     │  ← Method + Path
│  FROM: 192.168.1.10                        │  ← Client IP
│  METADATA:                                 │  ← Headers
│    Content-Type: application/json          │
│    Authorization: Bearer eyJ...            │
│  ─────────────────────────────────────     │
│  CONTENTS:                                 │  ← Body
│    { "name": "Alice", "email": "a@b.com" } │
└────────────────────────────────────────────┘

Response Envelope (what your server sends back):
┌────────────────────────────────────────────┐
│  STATUS: 201 Created                       │  ← Status Code
│  METADATA:                                 │  ← Headers
│    Content-Type: application/json          │
│    X-Request-ID: abc-123                   │
│  ─────────────────────────────────────     │
│  CONTENTS:                                 │  ← Body
│    { "data": { "id": 1, "name": "Alice" }} │
└────────────────────────────────────────────┘
```

---

## B. HonoRequest — Reading the Request

### The Full Request API

```typescript
app.post("/api/debug", async (c) => {
  // ─── Method & URL ────────────────────────
  c.req.method                      // "POST"
  c.req.url                         // "http://localhost:3000/api/debug?x=1"
  c.req.path                        // "/api/debug"
  c.req.routePath                   // "/api/debug" (the matched route pattern)

  // ─── Path Parameters ────────────────────
  c.req.param("id")                 // for /users/:id → "42"
  c.req.param()                     // all params as object

  // ─── Query Parameters ───────────────────
  c.req.query("page")               // "2" | undefined
  c.req.query()                     // all queries as object
  c.req.queries("tag")              // ["js", "ts"] (multi-value)

  // ─── Headers ────────────────────────────
  c.req.header("Authorization")     // "Bearer eyJ..."
  c.req.header("Content-Type")      // "application/json"
  c.req.header()                    // all headers as object

  // ─── Body Parsing ──────────────────────
  const json = await c.req.json()         // parse as JSON
  const text = await c.req.text()         // parse as text
  const form = await c.req.parseBody()    // parse form data
  const blob = await c.req.blob()         // raw binary
  const buffer = await c.req.arrayBuffer() // raw bytes

  // ─── Raw Web Standard Request ──────────
  c.req.raw                         // the underlying Request object

  return c.json({ method: c.req.method })
})
```

> ⚠️ **You can only read the body ONCE.** Calling `c.req.json()` consumes the stream. Calling it again will throw. If you need the body in multiple places, read it once and pass the parsed data.

### Parsing Different Body Types

```typescript
// JSON body (most common for APIs)
app.post("/api/users", async (c) => {
  const body = await c.req.json<{ name: string; email: string }>()
  //                          ↑ TypeScript generic for type hint
  return c.json({ received: body })
})

// Form data (HTML forms, file uploads)
app.post("/api/upload", async (c) => {
  const body = await c.req.parseBody()
  // For <input name="username"> → body["username"]
  // For <input type="file" name="avatar"> → body["avatar"] is a File
  return c.json({ fields: Object.keys(body) })
})

// Raw text
app.post("/api/webhook", async (c) => {
  const rawBody = await c.req.text()
  // Useful for webhook signature verification
  return c.text(`Received ${rawBody.length} characters`)
})
```

---

## C. Response Helpers — Complete Reference

```typescript
// ─── Text Response ──────────────────────────
c.text("Hello")                    // 200, text/plain
c.text("Not Found", 404)           // 404, text/plain

// ─── JSON Response ──────────────────────────
c.json({ data: [] })               // 200, application/json
c.json({ data: user }, 201)        // 201, application/json
c.json({ error: "Bad" }, 400)      // 400, application/json

// ─── HTML Response ──────────────────────────
c.html("<h1>Hello</h1>")           // 200, text/html

// ─── Redirect ───────────────────────────────
c.redirect("/new-path")            // 302 Found (temporary)
c.redirect("/new-path", 301)       // 301 Moved Permanently

// ─── No Content ─────────────────────────────
c.body(null, 204)                  // 204 No Content

// ─── Raw Response ───────────────────────────
// For full control, return a Web Standard Response
return new Response("raw", {
  status: 200,
  headers: { "X-Custom": "value" },
})
```

### Setting Headers

```typescript
app.get("/api/data", (c) => {
  // Set individual headers
  c.header("X-Request-ID", crypto.randomUUID())
  c.header("Cache-Control", "public, max-age=3600")
  c.header("X-API-Version", "1.0.0")

  // Append to existing header (for Set-Cookie, etc.)
  c.header("Set-Cookie", "a=1", { append: true })
  c.header("Set-Cookie", "b=2", { append: true })

  return c.json({ data: "with headers" })
})
```

### Setting Status Code

```typescript
app.post("/api/users", async (c) => {
  // Method 1: Pass status to response helper
  return c.json({ data: user }, 201)

  // Method 2: Set status separately (less common)
  c.status(201)
  return c.json({ data: user })
})
```

---

## D. Consistent Response Shapes

Production APIs need predictable response formats:

```typescript
// types/api.ts — define your response envelope
interface ApiSuccess<T> {
  data: T
  error: null
}

interface ApiError {
  data: null
  error: {
    code: string
    message: string
  }
}

type ApiResponse<T> = ApiSuccess<T> | ApiError

// Helper functions
function success<T>(c: Context, data: T, status: number = 200) {
  return c.json({ data, error: null }, status)
}

function error(c: Context, code: string, message: string, status: number = 400) {
  return c.json({ data: null, error: { code, message } }, status)
}
```

```typescript
// Usage in handlers
app.get("/api/users/:id", (c) => {
  const user = findUser(c.req.param("id"))

  if (!user) {
    return error(c, "NOT_FOUND", "User not found", 404)
  }

  return success(c, user)
})
```

> ✅ **Every response has the same shape.** Clients can always check `response.data` or `response.error` without guessing. This is what professional APIs do.

---

## E. Common Mistakes

### Mistake 1: Reading Body Twice

```typescript
// ❌ Second read throws — body stream consumed
app.post("/api/data", async (c) => {
  const body1 = await c.req.json()
  const body2 = await c.req.json()  // Throws!
})

// ✅ Read once, pass the parsed data
app.post("/api/data", async (c) => {
  const body = await c.req.json()
  doSomething(body)
  doSomethingElse(body)
  return c.json({ ok: true })
})
```

### Mistake 2: Not Handling Missing Data

```typescript
// ❌ Crashes if query param is missing
app.get("/search", (c) => {
  const page = Number(c.req.query("page"))  // NaN if missing!
})

// ✅ Provide defaults
app.get("/search", (c) => {
  const page = Number(c.req.query("page")) || 1
  const limit = Math.min(Number(c.req.query("limit")) || 20, 100)
})
```

---

## F. Exercises

### Exercise 1 — Request Inspector

Build a `POST /api/inspect` endpoint that returns everything about the request:
```json
{
  "method": "POST",
  "path": "/api/inspect",
  "headers": { "content-type": "application/json", "..." },
  "query": { "debug": "true" },
  "body": { "name": "test" }
}
```

### Exercise 2 — Response Helpers

Create `success()` and `error()` helper functions. Use them in a small API with at least 3 endpoints. Verify every response has a consistent shape.

---

## Summary

- ✅ Full `HonoRequest` API — params, queries, headers, body
- ✅ Body parsing — JSON, form data, text, raw
- ✅ Response helpers — `text()`, `json()`, `html()`, `redirect()`
- ✅ Headers and status codes
- ✅ Consistent response envelope pattern
- ✅ Body-can-only-be-read-once rule

---

## Navigation

← Previous: [Module 03 — Routing](./module-03.md)
→ Next: [Module 05 — Middleware](./module-05.md)
