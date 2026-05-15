# Module 03 — Routing

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Foundational → Intermediate
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [Module 02 — Hono Fundamentals](./module-02.md)
→ Next: [Module 04 — Request & Response](./module-04.md)

---

## What You'll Learn

- How Hono's trie-based router works internally
- Path parameters (`:id`), wildcards, and optional segments
- Query string handling
- Route grouping with `app.route()`
- Chaining routes
- Regex patterns in routes
- Base paths and API versioning
- Production-grade route organization

---

## A. Mental Model — Routing as a Decision Tree

Think of routing like a file system:

```
/api
 ├── /v1
 │    ├── /users
 │    │    ├── GET     → listUsers()
 │    │    ├── POST    → createUser()
 │    │    └── /:id
 │    │         ├── GET     → getUser()
 │    │         ├── PATCH   → updateUser()
 │    │         └── DELETE  → deleteUser()
 │    └── /posts
 │         ├── GET     → listPosts()
 │         └── /:id
 │              └── GET     → getPost()
 └── /health
      └── GET → healthCheck()
```

Hono builds this tree structure internally using a **trie** (prefix tree). When a request comes in, it walks the tree character-by-character to find the matching handler — this is why it's O(1), not O(n).

> 💡 Unlike Express which checks routes in registration order (top to bottom), Hono's trie router finds the best match structurally. This is faster and less prone to ordering bugs.

---

## B. Path Parameters — Dynamic Segments

Path parameters let you capture parts of the URL:

```typescript
import { Hono } from "hono"

const app = new Hono()

// Single parameter
app.get("/users/:id", (c) => {
  const id = c.req.param("id")         // "42"
  return c.json({ userId: id })
})

// Multiple parameters
app.get("/users/:userId/posts/:postId", (c) => {
  const userId = c.req.param("userId")   // "5"
  const postId = c.req.param("postId")   // "99"
  return c.json({ userId, postId })
})

// Get all params at once
app.get("/orgs/:orgId/teams/:teamId/members/:memberId", (c) => {
  const params = c.req.param()
  // { orgId: "1", teamId: "7", memberId: "42" }
  return c.json({ params })
})
```

**Critical rule:** Parameters are always strings. Always parse and validate.

```typescript
// ❌ Dangerous — no validation
app.get("/users/:id", (c) => {
  const id = c.req.param("id")  // Could be "abc", "-1", or "'; DROP TABLE--"
  // Using this directly in a DB query is a security risk!
})

// ✅ Safe — validate and parse
app.get("/users/:id", (c) => {
  const raw = c.req.param("id")
  const id = Number(raw)

  if (isNaN(id) || id < 1) {
    return c.json({ error: "Invalid user ID" }, 400)
  }

  return c.json({ userId: id })
})
```

---

## C. Wildcards and Regex

### Wildcards

```typescript
// Matches any path starting with /static/
app.get("/static/*", (c) => {
  // c.req.path might be "/static/images/logo.png"
  return c.text(`Serving static: ${c.req.path}`)
})

// Matches everything (catch-all / fallback)
app.all("*", (c) => {
  return c.json({ error: "Not found" }, 404)
})
```

### Optional and Regex Patterns

```typescript
// Optional parameter segment with regex
app.get("/posts/:id{[0-9]+}", (c) => {
  // Only matches if :id is numeric
  // /posts/42   ✅ matches
  // /posts/abc  ❌ does not match
  const id = c.req.param("id")
  return c.json({ postId: Number(id) })
})

// Custom patterns
app.get("/files/:filename{.+\\.pdf$}", (c) => {
  // Only matches .pdf files
  const filename = c.req.param("filename")
  return c.text(`PDF file: ${filename}`)
})
```

---

## D. Query Parameters

Query parameters (`?key=value`) are accessed through `c.req.query()`:

```typescript
// GET /search?q=hono&page=2&limit=20
app.get("/search", (c) => {
  const q = c.req.query("q")           // "hono"
  const page = c.req.query("page")     // "2" (string!)
  const limit = c.req.query("limit")   // "20" (string!)
  const missing = c.req.query("nope")  // undefined

  return c.json({
    query: q,
    page: Number(page) || 1,
    limit: Number(limit) || 10,
  })
})

// Get all query params at once
app.get("/filter", (c) => {
  const queries = c.req.queries()  // object with all params
  return c.json({ filters: queries })
})

// Multiple values for same key: GET /tags?tag=js&tag=ts
app.get("/tags", (c) => {
  const tags = c.req.queries("tag")  // ["js", "ts"]
  return c.json({ tags })
})
```

### Path Params vs Query Params — When to Use Which

```
Path params → identify a SPECIFIC resource
  GET /users/42          → "Get user #42"
  GET /posts/7/comments  → "Get comments for post #7"

Query params → filter, sort, paginate, or configure
  GET /users?role=admin&page=2    → "List admins, page 2"
  GET /posts?sort=date&order=desc → "List posts sorted by date"

Rule of thumb:
  If removing it makes the URL meaningless → path param
  If removing it just changes the result set → query param
```

---

## E. Route Grouping — `app.route()`

As your API grows, you need to split routes into separate files. Hono uses `app.route()` for this:

```typescript
// src/routes/users.ts
import { Hono } from "hono"

const users = new Hono()

users.get("/", (c) => c.json({ data: [] }))
users.post("/", async (c) => {
  const body = await c.req.json()
  return c.json({ data: body }, 201)
})
users.get("/:id", (c) => {
  return c.json({ data: { id: c.req.param("id") } })
})

export default users
```

```typescript
// src/routes/posts.ts
import { Hono } from "hono"

const posts = new Hono()

posts.get("/", (c) => c.json({ data: [] }))
posts.get("/:id", (c) => {
  return c.json({ data: { id: c.req.param("id") } })
})

export default posts
```

```typescript
// src/index.ts — compose routes
import { Hono } from "hono"
import users from "./routes/users"
import posts from "./routes/posts"

const app = new Hono()

// Mount sub-routers with base paths
app.route("/api/v1/users", users)
app.route("/api/v1/posts", posts)

// Health check at root level
app.get("/health", (c) => c.json({ status: "ok" }))

export default app
```

**The resulting routes:**

```
GET    /api/v1/users         → users.get("/")
POST   /api/v1/users         → users.post("/")
GET    /api/v1/users/:id     → users.get("/:id")
GET    /api/v1/posts         → posts.get("/")
GET    /api/v1/posts/:id     → posts.get("/:id")
GET    /health               → healthCheck
```

> 💡 **This is the production pattern.** One file per resource, composed in a central `index.ts`. This scales to 50+ routes cleanly.

---

## F. Method Chaining

Hono supports chaining for cleaner code:

```typescript
const app = new Hono()

// Chain multiple methods on the same path
app
  .get("/api/posts", (c) => c.json({ data: [] }))
  .post("/api/posts", async (c) => {
    const body = await c.req.json()
    return c.json({ data: body }, 201)
  })
  .get("/api/posts/:id", (c) => {
    return c.json({ data: { id: c.req.param("id") } })
  })
```

---

## G. Base Path

Set a base path for all routes in an app:

```typescript
// All routes in this app are prefixed with /api/v1
const app = new Hono().basePath("/api/v1")

app.get("/users", handler)      // → GET /api/v1/users
app.get("/posts", handler)      // → GET /api/v1/posts
app.get("/health", handler)     // → GET /api/v1/health
```

### API Versioning Pattern

```typescript
// src/index.ts
import { Hono } from "hono"
import v1Routes from "./routes/v1"
import v2Routes from "./routes/v2"

const app = new Hono()

app.route("/api/v1", v1Routes)
app.route("/api/v2", v2Routes)

// v1 and v2 can coexist during migration
export default app
```

---

## H. Production Route Organization

For a real project, organize routes like this:

```
src/
├── index.ts                ← app entry, compose routes
├── routes/
│   ├── users.ts            ← /api/v1/users routes
│   ├── posts.ts            ← /api/v1/posts routes
│   ├── auth.ts             ← /api/v1/auth routes
│   └── health.ts           ← /health route
├── middleware/              ← (Module 05)
├── services/                ← (Module 12)
├── db/                      ← (Module 08)
└── types/                   ← shared TypeScript types
    └── index.ts
```

> ✅ **One file per resource.** Each file exports a `Hono` sub-app. The main `index.ts` composes them. This pattern scales.

---

## I. Route Precedence and Matching

Understanding which route matches is critical:

```typescript
app.get("/users/me", (c) => c.text("Current user profile"))
app.get("/users/:id", (c) => c.text(`User ${c.req.param("id")}`))

// GET /users/me   → "Current user profile" (exact match wins)
// GET /users/42   → "User 42" (param match)
```

**Hono's matching priority:**
1. **Exact match** — `/users/me` beats `/users/:id`
2. **Parameterized** — `/users/:id` matches remaining patterns
3. **Wildcard** — `/users/*` is the broadest match

> ⚠️ In Express, order matters — the first matching route wins. In Hono, the most *specific* match wins regardless of registration order. This is less error-prone.

---

## J. Common Mistakes & Debugging

### Mistake 1: Duplicate Route Confusion

```typescript
// Both handlers exist — which runs?
app.get("/items/:id", (c) => c.text("By ID"))
app.get("/items/:slug", (c) => c.text("By slug"))

// They're actually the SAME pattern — both match /items/anything
// Only the first registered handler runs
// Fix: differentiate with regex
app.get("/items/:id{[0-9]+}", (c) => c.text("By ID"))
app.get("/items/:slug{[a-z-]+}", (c) => c.text("By slug"))
```

### Mistake 2: Forgetting the Leading Slash

```typescript
// ❌ Missing leading slash
app.get("users", handler)   // May not match as expected

// ✅ Always start paths with /
app.get("/users", handler)
```

### Mistake 3: Sub-Router Paths

```typescript
// In the sub-router, paths are RELATIVE to the mount point
const users = new Hono()
users.get("/", handler)        // Matches: /api/v1/users
users.get("/:id", handler)    // Matches: /api/v1/users/42

// ❌ Don't repeat the prefix
users.get("/api/v1/users", handler)  // Would match: /api/v1/users/api/v1/users
```

---

## K. 🧠 Comprehension Check

1. Why is Hono's trie router faster than Express's linear route matching?
2. What's the difference between `/users/:id` and `/users/*`?
3. When should you use path parameters vs query parameters?
4. How does `app.route()` help organize a large API?
5. What happens if two routes have the same pattern but different parameter names?

---

## L. Exercises

### Exercise 1 — Multi-Resource API

Build an API with three resources using separate route files:
- `GET/POST /api/v1/products`
- `GET/PATCH/DELETE /api/v1/products/:id`
- `GET/POST /api/v1/categories`
- `GET /api/v1/categories/:id/products` (nested resource)

Use `app.route()` to compose them in `index.ts`.

### Exercise 2 — Search and Pagination

Add a `GET /api/v1/products` endpoint that supports:
- `?search=keyboard` — filter by name
- `?page=2&limit=10` — pagination
- `?sort=price&order=desc` — sorting

Return mock data with proper pagination metadata:
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 10,
    "total": 45,
    "totalPages": 5
  }
}
```

---

## Summary

- ✅ Hono's trie-based router — fast, specific-match-first
- ✅ Path parameters (`:id`), wildcards (`*`), regex constraints
- ✅ Query parameter reading and parsing
- ✅ Route grouping with `app.route()` — one file per resource
- ✅ Method chaining and base paths
- ✅ API versioning patterns
- ✅ Production folder structure for routes

---

## Navigation

← Previous: [Module 02 — Hono Fundamentals](./module-02.md)
→ Next: [Module 04 — Request & Response](./module-04.md)
