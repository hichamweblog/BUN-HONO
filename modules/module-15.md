# Module 15 — OpenAPI

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~2 hours

---

## Navigation

← Previous: [Module 14 — Performance](./module-14.md)
→ Next: [Module 16 — Deployment](./module-16.md)

---

## What You'll Learn

- What OpenAPI is and why APIs need documentation
- Schema-first API design
- `@hono/zod-openapi` — auto-generate docs from Zod schemas
- Swagger UI integration
- API documentation best practices

---

## A. Mental Model — The API Menu

```
A restaurant without a menu:
  "What do you have?" → "Uh... a lot of things. Try ordering something."

A restaurant with a menu:
  ┌─────────────────────────────────────────────────┐
  │  MENU                                           │
  │                                                 │
  │  Appetizers                                     │
  │    GET /users      → List users (paginated)     │
  │    GET /users/:id  → Get user by ID             │
  │                                                 │
  │  Main Courses                                   │
  │    POST /users     → Create user                │
  │      Body: { name: string, email: string }      │
  │      Returns: 201 { data: User }                │
  │                                                 │
  │  Sides                                          │
  │    GET /health     → System health check        │
  └─────────────────────────────────────────────────┘

OpenAPI IS your API's menu.
```

---

## B. Setting Up @hono/zod-openapi

```bash
bun add @hono/zod-openapi
bun add @hono/swagger-ui
```

### Define Routes with OpenAPI Metadata

```typescript
// src/routes/user.openapi.ts
import { createRoute, z } from "@hono/zod-openapi"

const UserSchema = z.object({
  id: z.number().openapi({ example: 1 }),
  name: z.string().openapi({ example: "Alice" }),
  email: z.string().email().openapi({ example: "alice@example.com" }),
  role: z.enum(["user", "admin"]).openapi({ example: "user" }),
})

export const listUsersRoute = createRoute({
  method: "get",
  path: "/api/v1/users",
  tags: ["Users"],
  summary: "List all users",
  description: "Returns a paginated list of users.",
  request: {
    query: z.object({
      page: z.string().optional().default("1").openapi({ example: "1" }),
      limit: z.string().optional().default("20").openapi({ example: "20" }),
    }),
  },
  responses: {
    200: {
      description: "Successful response",
      content: {
        "application/json": {
          schema: z.object({
            data: z.array(UserSchema),
            pagination: z.object({
              page: z.number(),
              limit: z.number(),
              total: z.number(),
            }),
          }),
        },
      },
    },
  },
})

export const createUserRoute = createRoute({
  method: "post",
  path: "/api/v1/users",
  tags: ["Users"],
  summary: "Create a new user",
  request: {
    body: {
      content: {
        "application/json": {
          schema: z.object({
            name: z.string().min(2),
            email: z.string().email(),
          }),
        },
      },
    },
  },
  responses: {
    201: {
      description: "User created",
      content: {
        "application/json": {
          schema: z.object({ data: UserSchema }),
        },
      },
    },
    409: {
      description: "Email already exists",
    },
  },
})
```

### Register Routes with OpenAPI App

```typescript
// src/app.ts
import { OpenAPIHono } from "@hono/zod-openapi"
import { swaggerUI } from "@hono/swagger-ui"
import { listUsersRoute, createUserRoute } from "./routes/user.openapi"

const app = new OpenAPIHono()

// Register routes with handlers
app.openapi(listUsersRoute, async (c) => {
  const { page, limit } = c.req.valid("query")
  // ... fetch users
  return c.json({ data: users, pagination: { page: 1, limit: 20, total: 100 } })
})

app.openapi(createUserRoute, async (c) => {
  const body = c.req.valid("json")
  // ... create user
  return c.json({ data: newUser }, 201)
})

// Generate OpenAPI spec
app.doc("/api/doc", {
  openapi: "3.1.0",
  info: {
    title: "My API",
    version: "1.0.0",
    description: "A production-grade REST API built with Hono + Bun",
  },
})

// Serve Swagger UI
app.get("/api/ui", swaggerUI({ url: "/api/doc" }))

export default app
```

Now visit:
- `http://localhost:3000/api/doc` — raw OpenAPI JSON spec
- `http://localhost:3000/api/ui` — interactive Swagger UI

---

## C. Benefits of OpenAPI

```
✅ Frontend team can start building UI before backend is done
✅ Auto-generated client SDKs (openapi-typescript, orval)
✅ Interactive documentation for testing
✅ API contract validation
✅ Single source of truth — schema = docs = validation = types
```

---

## D. Exercises

### Exercise 1 — Document the Blog API

Add OpenAPI metadata to all blog API routes. Generate a complete Swagger UI. Include:
- Authentication schemas (Bearer token)
- Error response schemas
- Pagination metadata

### Exercise 2 — Generate Client Types

Use `openapi-typescript` to generate TypeScript types from your OpenAPI spec:
```bash
bunx openapi-typescript http://localhost:3000/api/doc -o ./client-types.ts
```

---

## Summary

- ✅ OpenAPI — the industry standard for API documentation
- ✅ `@hono/zod-openapi` — schemas are docs, validation, AND types
- ✅ Swagger UI — interactive documentation
- ✅ Schema-first design — define the contract before the code

---

## Navigation

← Previous: [Module 14 — Performance](./module-14.md)
→ Next: [Module 16 — Deployment](./module-16.md)
