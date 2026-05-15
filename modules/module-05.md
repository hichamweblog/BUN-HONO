# Module 05 — Middleware

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Intermediate
> **Duration:** ~3–4 hours

---

## Navigation

← Previous: [Module 04 — Request & Response](./module-04.md)
→ Next: [Module 06 — Validation](./module-06.md)

---

## What You'll Learn

- What middleware is and why it's the backbone of backend architecture
- Hono's middleware execution model (the "onion" model)
- Built-in middleware: logger, cors, secure headers, etc.
- Writing custom middleware
- Passing data between middleware with `c.set()` / `c.get()`
- Middleware ordering and scoping
- Typed middleware with generics

---

## A. Mental Model — Airport Security Checkpoints

```
Passenger (Request) arrives at airport:

CHECK 1: Ticket Check (Logger)
  → "Passenger heading to Gate B7" (logs the request)
  → Continue ✅

CHECK 2: Security Scan (Auth Middleware)
  → Valid boarding pass? → Continue ✅
  → No pass? → REJECTED (401) 🚫

CHECK 3: Customs Declaration (Validation)
  → Carrying banned items? → REJECTED (400) 🚫
  → All clear? → Continue ✅

GATE: Board the Plane (Route Handler)
  → Process the request, return response

RETURN TRIP (Response flows back through middleware):
  → Customs stamps passport (add response headers)
  → Security logs departure (timing middleware)
  → Ticket desk records completion (logger)
```

This is the **onion model** — the request goes IN through layers, hits the handler, and the response comes OUT through the same layers in reverse.

```
        Request →
    ┌─────────────────────────────────────┐
    │  Logger Middleware                   │
    │  ┌───────────────────────────────┐  │
    │  │  CORS Middleware              │  │
    │  │  ┌─────────────────────────┐  │  │
    │  │  │  Auth Middleware        │  │  │
    │  │  │  ┌───────────────────┐  │  │  │
    │  │  │  │  Route Handler    │  │  │  │
    │  │  │  └───────────────────┘  │  │  │
    │  │  └─────────────────────────┘  │  │
    │  └───────────────────────────────┘  │
    └─────────────────────────────────────┘
        ← Response
```

---

## B. Built-in Middleware

Hono ships with production-ready middleware. Here are the essentials:

### Logger

```typescript
import { Hono } from "hono"
import { logger } from "hono/logger"

const app = new Hono()
app.use(logger())

// Output: <-- GET /api/users
//         --> GET /api/users 200 12ms
```

### CORS

```typescript
import { cors } from "hono/cors"

// Allow all origins (development only!)
app.use("/api/*", cors())

// Production CORS
app.use("/api/*", cors({
  origin: ["https://myapp.com", "https://admin.myapp.com"],
  allowMethods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
  allowHeaders: ["Content-Type", "Authorization"],
  maxAge: 86400,  // preflight cache: 24 hours
}))
```

### Secure Headers

```typescript
import { secureHeaders } from "hono/secure-headers"

app.use(secureHeaders())
// Adds: X-Frame-Options, X-Content-Type-Options,
//       Strict-Transport-Security, etc.
```

### Request ID

```typescript
import { requestId } from "hono/request-id"

app.use(requestId())

app.get("/api/data", (c) => {
  const id = c.get("requestId")  // UUID for tracing
  return c.json({ requestId: id })
})
```

### Timing

```typescript
import { timing, startTime, endTime } from "hono/timing"

app.use(timing())

app.get("/api/data", async (c) => {
  startTime(c, "db")
  const data = await fetchFromDB()
  endTime(c, "db")

  return c.json({ data })
  // Response header: Server-Timing: db;dur=23.5
})
```

---

## C. Writing Custom Middleware

A middleware is a function that receives `(c, next)`:

```typescript
import { createMiddleware } from "hono/factory"

// Simple logging middleware
const requestLogger = createMiddleware(async (c, next) => {
  const start = Date.now()
  console.log(`→ ${c.req.method} ${c.req.path}`)

  await next()  // ← Call the next middleware/handler

  const duration = Date.now() - start
  console.log(`← ${c.req.method} ${c.req.path} ${c.res.status} ${duration}ms`)
})

app.use(requestLogger)
```

**The key concept: `await next()`**

```
1. Code BEFORE `await next()` runs on the way IN (request phase)
2. `await next()` calls the next middleware or handler
3. Code AFTER `await next()` runs on the way OUT (response phase)

This is WHY you can time requests:
  start timer → await next() → stop timer
```

### Middleware That Short-Circuits

```typescript
const apiKeyAuth = createMiddleware(async (c, next) => {
  const apiKey = c.req.header("X-API-Key")

  if (!apiKey || apiKey !== Bun.env.API_KEY) {
    // Short-circuit — don't call next(), return early
    return c.json({ error: "Invalid API key" }, 401)
  }

  await next()  // Key is valid, continue to handler
})

app.use("/api/*", apiKeyAuth)
```

---

## D. Passing Data Through Middleware — `c.set()` / `c.get()`

Middleware can attach data to the context for downstream handlers:

```typescript
// Type-safe variables with Hono generics
type Variables = {
  user: { id: number; name: string; role: string }
  requestStartTime: number
}

const app = new Hono<{ Variables: Variables }>()

// Auth middleware sets user
const authMiddleware = createMiddleware<{ Variables: Variables }>(
  async (c, next) => {
    const token = c.req.header("Authorization")?.replace("Bearer ", "")

    if (!token) {
      return c.json({ error: "Unauthorized" }, 401)
    }

    // In real code: verify JWT, look up user
    const user = { id: 1, name: "Alice", role: "admin" }
    c.set("user", user)  // ← attach to context

    await next()
  }
)

app.use("/api/*", authMiddleware)

app.get("/api/me", (c) => {
  const user = c.get("user")  // ← read from context (fully typed!)
  return c.json({ data: user })
})
```

> 💡 The `Variables` type parameter makes `c.get("user")` return the correct type. No casting needed. This is Hono's TypeScript power.

---

## E. Middleware Scoping

```typescript
// Global — applies to ALL routes
app.use(logger())

// Path-scoped — applies to matching paths
app.use("/api/*", cors())
app.use("/api/*", authMiddleware)

// Route-specific — inline middleware
app.get("/admin/dashboard", adminOnly, (c) => {
  return c.json({ data: "admin panel" })
})

// Multiple middleware on one route
app.post("/api/users",
  rateLimiter,
  validateBody,
  async (c) => {
    const body = await c.req.json()
    return c.json({ data: body }, 201)
  }
)
```

### Execution Order

```typescript
app.use(logger())         // 1st — runs first
app.use(cors())           // 2nd
app.use(authMiddleware)   // 3rd

app.get("/api/users", (c) => { ... })  // 4th — the handler

// Request flow:  logger → cors → auth → handler
// Response flow: handler → auth → cors → logger
```

---

## F. Common Mistakes

### Mistake 1: Forgetting `await next()`

```typescript
// ❌ Missing await next() — downstream handlers never run
const broken = createMiddleware(async (c, next) => {
  console.log("I run, but nothing after me does")
  // Forgot to call next()!
})

// ✅ Always call next() unless intentionally short-circuiting
const working = createMiddleware(async (c, next) => {
  console.log("Before handler")
  await next()
  console.log("After handler")
})
```

### Mistake 2: Wrong Middleware Order

```typescript
// ❌ Auth after route — handler runs without auth!
app.get("/api/secret", (c) => c.json({ secret: "data" }))
app.use("/api/*", authMiddleware)  // Too late!

// ✅ Middleware BEFORE routes
app.use("/api/*", authMiddleware)
app.get("/api/secret", (c) => c.json({ secret: "data" }))
```

---

## G. Exercises

### Exercise 1 — Request Timer Middleware

Write middleware that measures request duration and adds a `X-Response-Time` header to every response (e.g., `X-Response-Time: 23ms`).

### Exercise 2 — API Key Guard

Write middleware that checks for `X-API-Key` header. If missing or wrong, return 401. If correct, allow through. Apply it only to `/api/*` routes.

---

## Summary

- ✅ Middleware — the onion model (in → handler → out)
- ✅ Built-in: logger, CORS, secure headers, request ID, timing
- ✅ Custom middleware with `createMiddleware`
- ✅ `await next()` — controls flow to downstream handlers
- ✅ `c.set()` / `c.get()` — pass data between middleware (type-safe)
- ✅ Scoping: global, path-based, route-specific
- ✅ Order matters — middleware before routes

---

## Navigation

← Previous: [Module 04 — Request & Response](./module-04.md)
→ Next: [Module 06 — Validation](./module-06.md)
