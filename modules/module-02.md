# Module 02 — Hono Fundamentals

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Foundational
> **Duration:** ~2–3 hours
> **Project:** Hello API

---

## Navigation

← Previous: [Module 01 — Bun Fundamentals](./module-01.md)
→ Next: [Module 03 — Routing](./module-03.md)

---

## What You'll Learn

- What Hono is and its design philosophy
- Why Hono over Express, Fastify, or Elysia
- Web Standards at the core of Hono
- Setting up a Hono + Bun project
- The Hono `App` object
- Your first routes and responses
- The Context object (`c`) — your primary tool
- Project structure for production apps

---

## A. Mental Model — What IS Hono?

### The Switchboard Operator Analogy

Imagine a telephone switchboard operator from the 1950s:

```
Incoming call (HTTP Request)
       │
       ▼
  Switchboard Operator (Hono)
       │
       ├── "Connect me to Sales"     → routes to Sales handler
       ├── "Connect me to Support"   → routes to Support handler
       ├── "Connect me to Billing"   → routes to Billing handler
       └── "Unknown department"      → "Sorry, not found" (404)
```

Hono is the switchboard operator for your HTTP traffic. It:
1. **Receives** incoming HTTP requests
2. **Matches** them to the right handler based on path and method
3. **Runs** middleware (security checks, logging) along the way
4. **Returns** the response

> 💡 Hono (炎) means "flame" in Japanese. It's designed to be small, fast, and burn through requests with minimal overhead.

---

## B. Why Hono? — Honest Comparison

| Feature | Express | Fastify | Hono |
|---------|---------|---------|------|
| **Runtime** | Node.js only | Node.js only | Any (Bun, Deno, Workers, Node) |
| **TypeScript** | Add-on types | Good support | First-class, deeply typed |
| **API Style** | Callback-based (`req, res`) | Schema-based | Web Standards (`Request → Response`) |
| **Bundle Size** | ~200KB+ | ~300KB+ | ~14KB (ultra-small) |
| **Middleware** | Massive ecosystem | Plugin system | Built-in essentials + ecosystem |
| **Performance** | Baseline | Fast | Extremely fast (trie router) |
| **Learning Curve** | Low | Medium | Low-Medium |
| **Edge/Serverless** | Poor fit | Poor fit | Built for it |

### When to Choose Hono

```
✅ Choose Hono when:
• Building APIs with Bun, Deno, or Cloudflare Workers
• You want multi-runtime portability
• You want type-safe routes and middleware
• You value small bundle size (edge/serverless)
• You want modern Web Standard APIs

⚠️ Consider alternatives when:
• You need a massive Express middleware ecosystem (legacy plugins)
• Your team only knows Express and has no time to learn
• You're extending an existing Express/Fastify codebase
```

---

## C. Hono's Core Philosophy

From the official docs, Hono is built on three pillars:

### 1. Web Standards
Hono uses only Web Standard APIs (`Request`, `Response`, `Headers`, `URL`). This means the same code runs everywhere — Bun, Deno, Cloudflare Workers, Node.js.

### 2. Performance
Hono uses an ultra-fast **trie-based router** (RegExpRouter) that benchmarks faster than almost every other framework. Route matching is O(1) via precompiled regex, not O(n) path scanning.

### 3. Developer Experience
Full TypeScript inference — when you define a route with validation, the types flow all the way through. No manual type casting.

```
                    Web Standards
                    /           \
                   /             \
          Performance ←——————→ Developer
                                Experience

           "Fast, portable, and a joy to use"
```

---

## D. Project Setup — Hello API

Let's build your first Hono project:

```bash
# Navigate to workspace
cd /path/to/your/workspace

# Create project
mkdir hello-api && cd hello-api

# Initialize
bun init -y

# Add Hono
bun add hono

# Create source directory
mkdir src
```

Your project structure:

```
hello-api/
├── src/
│   └── index.ts      ← your app entry point
├── package.json
├── tsconfig.json
└── bun.lockb
```

Update `package.json` scripts:

```json
{
  "name": "hello-api",
  "scripts": {
    "dev": "bun run --hot src/index.ts",
    "start": "bun run src/index.ts",
    "test": "bun test"
  },
  "dependencies": {
    "hono": "^4.0.0"
  }
}
```

---

## E. Your First Hono App — Line by Line

```typescript
// src/index.ts
import { Hono } from "hono"

const app = new Hono()

app.get("/", (c) => {
  return c.text("Hello Hono!")
})

export default app
```

**Line-by-line breakdown:**

```typescript
import { Hono } from "hono"
```
Imports the Hono class. This is the core — everything starts here.

```typescript
const app = new Hono()
```
Creates a new Hono application instance. This object holds all your routes, middleware, and configuration. You'll have exactly **one** of these per application (or per sub-app — more on that in Module 03).

```typescript
app.get("/", (c) => {
  return c.text("Hello Hono!")
})
```
Registers a route handler:
- `app.get` — listens for GET requests
- `"/"` — matches the root path
- `(c) => { ... }` — the handler function. `c` is the **Context** object
- `c.text("Hello Hono!")` — returns a plain text response with `Content-Type: text/plain`

```typescript
export default app
```
Exports the app. Bun sees a default export with a `.fetch()` method and automatically starts an HTTP server on port 3000.

### Run It

```bash
bun run dev
# Server running at http://localhost:3000
```

```bash
# Test with curl
curl http://localhost:3000
# Hello Hono!
```

---

## F. The Context Object (`c`) — Deep Dive

The Context object `c` is your **primary interface** in every Hono handler. It wraps the raw `Request` and provides helpers to build `Response` objects.

```
┌─────────────────────────────────────────────────────────┐
│  Context (c)                                             │
│                                                          │
│  ┌─────────────────────────┐ ┌────────────────────────┐ │
│  │  c.req (HonoRequest)    │ │  Response Helpers      │ │
│  │  • .method              │ │  • c.text()            │ │
│  │  • .url                 │ │  • c.json()            │ │
│  │  • .path               │ │  • c.html()            │ │
│  │  • .header()            │ │  • c.redirect()        │ │
│  │  • .param()             │ │  • c.body()            │ │
│  │  • .query()             │ │  • c.notFound()        │ │
│  │  • .json()              │ │                        │ │
│  │  • .text()              │ │  Headers & Status      │ │
│  │  • .parseBody()         │ │  • c.header()          │ │
│  └─────────────────────────┘ │  • c.status()          │ │
│                              └────────────────────────┘ │
│  ┌─────────────────────────┐ ┌────────────────────────┐ │
│  │  Variables              │ │  Env / Bindings        │ │
│  │  • c.set(key, value)    │ │  • c.env               │ │
│  │  • c.get(key)           │ │                        │ │
│  │  • c.var                │ │                        │ │
│  └─────────────────────────┘ └────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Response Helpers

```typescript
// Plain text
app.get("/text", (c) => c.text("Hello"))
// Content-Type: text/plain

// JSON (most common for APIs)
app.get("/json", (c) => c.json({ message: "Hello", status: "ok" }))
// Content-Type: application/json

// JSON with status code
app.post("/users", (c) => c.json({ id: 1, name: "Alice" }, 201))
// Status: 201 Created

// HTML
app.get("/page", (c) => c.html("<h1>Hello</h1>"))
// Content-Type: text/html

// Redirect
app.get("/old", (c) => c.redirect("/new"))
// Status: 302 Found

// Redirect with specific status
app.get("/legacy", (c) => c.redirect("/modern", 301))
// Status: 301 Moved Permanently

// No content
app.delete("/users/:id", (c) => c.body(null, 204))
// Status: 204 No Content

// Custom headers
app.get("/custom", (c) => {
  c.header("X-Custom-Header", "my-value")
  c.header("Cache-Control", "no-cache")
  return c.json({ data: "with custom headers" })
})
```

### Reading Request Data

```typescript
// Query parameters: GET /search?q=hono&page=2
app.get("/search", (c) => {
  const query = c.req.query("q")       // "hono"
  const page = c.req.query("page")     // "2" (always a string!)
  return c.json({ query, page })
})

// Path parameters: GET /users/42
app.get("/users/:id", (c) => {
  const id = c.req.param("id")         // "42" (always a string!)
  return c.json({ userId: id })
})

// Request headers
app.get("/headers", (c) => {
  const auth = c.req.header("Authorization")  // "Bearer eyJ..."
  const ua = c.req.header("User-Agent")
  return c.json({ auth: !!auth, userAgent: ua })
})

// JSON body: POST /users with {"name": "Alice"}
app.post("/users", async (c) => {
  const body = await c.req.json()       // { name: "Alice" }
  return c.json({ received: body }, 201)
})

// HTTP method and path
app.all("/info", (c) => {
  return c.json({
    method: c.req.method,    // "GET", "POST", etc.
    path: c.req.path,        // "/info"
    url: c.req.url,          // "http://localhost:3000/info?x=1"
  })
})
```

> ⚠️ **Important**: `c.req.param()` and `c.req.query()` always return `string | undefined`. Never assume a number — always parse and validate. We cover this properly in Module 06 (Validation).

---

## G. HTTP Methods in Hono

Hono provides methods for all standard HTTP verbs:

```typescript
const app = new Hono()

app.get("/users", handler)       // GET     — list/retrieve
app.post("/users", handler)      // POST    — create
app.put("/users/:id", handler)   // PUT     — replace
app.patch("/users/:id", handler) // PATCH   — partial update
app.delete("/users/:id", handler)// DELETE  — remove
app.options("/users", handler)   // OPTIONS — CORS preflight
app.all("/any", handler)         // ALL     — matches any method
```

### Building a Complete CRUD Example

```typescript
// src/index.ts — In-memory Notes API
import { Hono } from "hono"

const app = new Hono()

// In-memory data store (for now — replaced with DB in Module 08)
interface Note {
  id: number
  title: string
  content: string
  createdAt: string
}

let notes: Note[] = []
let nextId = 1

// List all notes
app.get("/api/notes", (c) => {
  return c.json({ data: notes })
})

// Get a single note
app.get("/api/notes/:id", (c) => {
  const id = Number(c.req.param("id"))
  const note = notes.find((n) => n.id === id)

  if (!note) {
    return c.json({ error: "Note not found" }, 404)
  }

  return c.json({ data: note })
})

// Create a note
app.post("/api/notes", async (c) => {
  const body = await c.req.json()

  const note: Note = {
    id: nextId++,
    title: body.title,
    content: body.content,
    createdAt: new Date().toISOString(),
  }

  notes.push(note)
  return c.json({ data: note }, 201)
})

// Update a note
app.patch("/api/notes/:id", async (c) => {
  const id = Number(c.req.param("id"))
  const noteIndex = notes.findIndex((n) => n.id === id)

  if (noteIndex === -1) {
    return c.json({ error: "Note not found" }, 404)
  }

  const body = await c.req.json()
  notes[noteIndex] = { ...notes[noteIndex], ...body }

  return c.json({ data: notes[noteIndex] })
})

// Delete a note
app.delete("/api/notes/:id", (c) => {
  const id = Number(c.req.param("id"))
  const noteIndex = notes.findIndex((n) => n.id === id)

  if (noteIndex === -1) {
    return c.json({ error: "Note not found" }, 404)
  }

  notes.splice(noteIndex, 1)
  return c.body(null, 204)
})

export default app
```

> 🚨 **This is NOT production code yet.** There's no validation, no error handling, no auth. We're building foundations. Each subsequent module adds a professional layer.

---

## H. Configuring the Port

```typescript
// Method 1: export default with port
export default {
  port: 8080,
  fetch: app.fetch,
}

// Method 2: environment variable
export default {
  port: Number(Bun.env.PORT) || 3000,
  fetch: app.fetch,
}
```

> ✅ **Best Practice**: Always allow port configuration via environment variables. Hard-coded ports cause conflicts in Docker and CI environments.

---

## I. Common Mistakes & Debugging

### Mistake 1: Forgetting to Return the Response

```typescript
// ❌ Missing return — handler returns undefined
app.get("/oops", (c) => {
  c.json({ message: "hello" })  // no return!
})

// ✅ Always return the response
app.get("/correct", (c) => {
  return c.json({ message: "hello" })
})
```

### Mistake 2: Not Awaiting the Body

```typescript
// ❌ c.req.json() is async — must await
app.post("/users", (c) => {
  const body = c.req.json()  // Promise, not data!
  return c.json({ received: body })  // sends a Promise object
})

// ✅ Mark handler as async and await
app.post("/users", async (c) => {
  const body = await c.req.json()
  return c.json({ received: body })
})
```

### Mistake 3: Wrong Path Matching

```typescript
// ❌ Trailing slash matters!
app.get("/users", handler)    // matches /users
                               // does NOT match /users/

// ✅ Hono has Trailing Slash middleware if you need flexibility
import { trimTrailingSlash } from "hono/trailing-slash"
app.use(trimTrailingSlash())
```

### Mistake 4: Modifying Response After Sending

```typescript
// ❌ Can't modify headers after returning
app.get("/oops", (c) => {
  const res = c.json({ data: "hello" })
  c.header("X-After", "too-late")  // this won't work!
  return res
})

// ✅ Set headers BEFORE building the response
app.get("/correct", (c) => {
  c.header("X-Custom", "value")
  return c.json({ data: "hello" })
})
```

---

## J. 🧠 Comprehension Check

1. What are the three pillars of Hono's philosophy?
2. What does `export default app` do in a Bun environment?
3. What is the Context object (`c`) and what two things does it wrap?
4. What's the difference between `c.text()`, `c.json()`, and `c.html()`?
5. Why do `c.req.param()` and `c.req.query()` always return strings?
6. What HTTP status code should you use for a successful `POST` that creates a resource?

---

## K. Exercises

### Exercise 1 — Bookmarks API

Build an in-memory Bookmarks API with these endpoints:
- `GET /api/bookmarks` — list all bookmarks
- `POST /api/bookmarks` — create a bookmark (`{ url, title, tags }`)
- `GET /api/bookmarks/:id` — get one bookmark
- `DELETE /api/bookmarks/:id` — delete a bookmark

Use proper status codes (200, 201, 204, 404).

### Exercise 2 — Health Check Endpoint

Add a `GET /health` endpoint that returns:
```json
{
  "status": "ok",
  "uptime": 12345,
  "timestamp": "2024-01-15T09:30:00Z"
}
```

The `uptime` should be real (use `process.uptime()`).

---

## Summary

You now understand:
- ✅ Hono's philosophy — Web Standards, performance, DX
- ✅ How to set up a Hono + Bun project from scratch
- ✅ The `Hono` app object and route registration
- ✅ The Context object (`c`) — request reading and response building
- ✅ All HTTP method handlers (`get`, `post`, `put`, `patch`, `delete`)
- ✅ A complete CRUD API (in-memory, no validation yet)
- ✅ The `export default` pattern for Bun

---

## Navigation

← Previous: [Module 01 — Bun Fundamentals](./module-01.md)
→ Next: [Module 03 — Routing](./module-03.md)

---

> **Mentor Note**: You now have a working API. It's ugly — no validation,
> no error handling, no auth. That's intentional. Each module adds one
> professional layer. By Module 12, this same API will be production-grade.
