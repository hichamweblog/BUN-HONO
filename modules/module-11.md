# Module 11 — Testing

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Intermediate → Advanced
> **Duration:** ~3–4 hours

---

## Navigation

← Previous: [Module 10 — Authorization](./module-10.md)
→ Next: [Module 12 — Production Architecture](./module-12.md)

---

## What You'll Learn

- Testing philosophy — what to test, what not to
- Bun's built-in test runner (`bun:test`)
- Unit testing services and utilities
- Integration testing Hono endpoints
- Hono's `app.request()` test helper
- Mocking and test isolation
- Test organization and patterns

---

## A. Mental Model — The Safety Harness

```
Without tests:
  "I changed the auth middleware... does login still work?"
  → Deploy and pray 🙏

With tests:
  "I changed the auth middleware."
  → bun test → 47 tests pass, 0 fail → Deploy with confidence ✅
```

---

## B. Testing Philosophy

```
What TO test:                         What NOT to test:
✅ Business logic (services)          ❌ Framework internals (Hono itself)
✅ API endpoints (integration)        ❌ Third-party libraries
✅ Validation schemas                 ❌ Simple getters/setters
✅ Auth flows (login, guards)         ❌ Database engine behavior
✅ Error handling paths               ❌ TypeScript types (compiler does this)
✅ Edge cases and boundaries
```

> 💡 **The testing pyramid**: Many unit tests (fast, isolated), fewer integration tests (slower, more realistic), very few E2E tests (slowest, most brittle).

---

## C. Testing Hono Apps — `app.request()`

Hono provides a built-in test helper — no HTTP server needed:

```typescript
// src/index.ts
import { Hono } from "hono"

const app = new Hono()

app.get("/api/health", (c) => c.json({ status: "ok" }))

app.get("/api/users/:id", (c) => {
  const id = c.req.param("id")
  if (id === "999") {
    return c.json({ error: "Not found" }, 404)
  }
  return c.json({ data: { id: Number(id), name: "Alice" } })
})

export default app
```

```typescript
// src/index.test.ts
import { describe, expect, it } from "bun:test"
import app from "./index"

describe("Health Check", () => {
  it("returns ok status", async () => {
    const res = await app.request("/api/health")
    expect(res.status).toBe(200)

    const body = await res.json()
    expect(body.status).toBe("ok")
  })
})

describe("GET /api/users/:id", () => {
  it("returns user for valid ID", async () => {
    const res = await app.request("/api/users/1")
    expect(res.status).toBe(200)

    const body = await res.json()
    expect(body.data.id).toBe(1)
    expect(body.data.name).toBe("Alice")
  })

  it("returns 404 for non-existent user", async () => {
    const res = await app.request("/api/users/999")
    expect(res.status).toBe(404)
  })
})

describe("POST /api/users", () => {
  it("creates a user with valid data", async () => {
    const res = await app.request("/api/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name: "Bob", email: "bob@test.com" }),
    })

    expect(res.status).toBe(201)
    const body = await res.json()
    expect(body.data.name).toBe("Bob")
  })

  it("rejects invalid data", async () => {
    const res = await app.request("/api/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name: "" }),  // invalid
    })

    expect(res.status).toBe(422)
  })
})
```

```bash
bun test
# ✓ Health Check > returns ok status
# ✓ GET /api/users/:id > returns user for valid ID
# ✓ GET /api/users/:id > returns 404 for non-existent user
# ✓ POST /api/users > creates a user with valid data
# ✓ POST /api/users > rejects invalid data
# 5 pass, 0 fail
```

> 💡 **`app.request()`** creates a real `Request` object and passes it through Hono's full middleware chain and router — no HTTP server needed. This is fast and tests the real code path.

---

## D. Testing with Auth

```typescript
// Helper to create authenticated requests
function authRequest(path: string, options: RequestInit = {}) {
  return app.request(path, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: "Bearer valid-test-token",
    },
  })
}

describe("Protected routes", () => {
  it("returns 401 without auth", async () => {
    const res = await app.request("/api/me")
    expect(res.status).toBe(401)
  })

  it("returns user with valid token", async () => {
    const res = await authRequest("/api/me")
    expect(res.status).toBe(200)
  })
})
```

---

## E. Unit Testing Services

```typescript
// src/services/user.service.test.ts
import { describe, expect, it } from "bun:test"

// Pure function — easy to unit test
function formatUserResponse(user: { id: number; name: string; email: string }) {
  return {
    id: user.id,
    displayName: user.name,
    email: user.email,
    initials: user.name.split(" ").map(n => n[0]).join("").toUpperCase(),
  }
}

describe("formatUserResponse", () => {
  it("formats user correctly", () => {
    const result = formatUserResponse({
      id: 1, name: "Alice Johnson", email: "alice@test.com"
    })

    expect(result.displayName).toBe("Alice Johnson")
    expect(result.initials).toBe("AJ")
  })

  it("handles single name", () => {
    const result = formatUserResponse({
      id: 2, name: "Bob", email: "bob@test.com"
    })
    expect(result.initials).toBe("B")
  })
})
```

---

## F. Test Organization

```
src/
├── routes/
│   ├── users.ts
│   └── users.test.ts        ← co-located tests
├── services/
│   ├── user.service.ts
│   └── user.service.test.ts
└── index.test.ts             ← integration tests
```

> ✅ **Co-locate tests with source files.** `users.ts` → `users.test.ts`. This makes tests easy to find and maintain.

---

## G. Exercises

### Exercise 1 — Test Suite

Write tests for the Notes API:
- `GET /api/notes` returns 200 with array
- `POST /api/notes` with valid data returns 201
- `POST /api/notes` with invalid data returns 422
- `GET /api/notes/:id` with non-existent ID returns 404
- `DELETE /api/notes/:id` returns 204

### Exercise 2 — Auth Tests

Write tests that verify:
- Unauthenticated requests to protected routes return 401
- Authenticated users can access their own resources
- Users cannot access other users' resources (403)

---

## Summary

- ✅ Testing philosophy — test behavior, not implementation
- ✅ `app.request()` — test Hono apps without HTTP server
- ✅ Unit tests for services and utilities
- ✅ Integration tests for full endpoint flows
- ✅ Auth-aware test helpers
- ✅ Co-located test file organization

---

## Navigation

← Previous: [Module 10 — Authorization](./module-10.md)
→ Next: [Module 12 — Production Architecture](./module-12.md)
