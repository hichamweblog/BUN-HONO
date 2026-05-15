# Module 06 — Validation

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Intermediate
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [Module 05 — Middleware](./module-05.md)
→ Next: [Module 07 — Error Handling](./module-07.md)

---

## What You'll Learn

- Why validation is non-negotiable in production
- Zod — the standard for TypeScript validation
- Hono's built-in `zValidator` middleware
- Validating body, query params, path params, and headers
- Schema design best practices
- Type inference from schemas (no duplicate types)
- Custom error messages and formatting

---

## A. Mental Model — The Bouncer at the Door

```
Client sends data →  🚪 BOUNCER (Validation) → Handler

  "I'm 25" (valid)     → ✅ Let in
  "I'm -3" (invalid)   → 🚫 Rejected: "Age must be positive"
  "" (empty)            → 🚫 Rejected: "Age is required"
  "abc" (wrong type)    → 🚫 Rejected: "Age must be a number"
```

> 🚨 **Rule #1 of backend engineering: Never trust client data.** Every field, every parameter, every header could contain garbage, attacks, or mistakes. Validation is your first line of defense.

---

## B. Zod — Schema-First Validation

Zod is a TypeScript-first schema validation library. Define a schema once, get both validation AND TypeScript types:

```bash
bun add zod
```

```typescript
import { z } from "zod"

// Define a schema
const createUserSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email format"),
  age: z.number().int().min(18, "Must be at least 18").optional(),
  role: z.enum(["user", "admin"]).default("user"),
})

// Infer the TypeScript type FROM the schema
type CreateUserInput = z.infer<typeof createUserSchema>
// → { name: string; email: string; age?: number; role: "user" | "admin" }

// Validate data
const result = createUserSchema.safeParse({
  name: "Alice",
  email: "alice@example.com",
  role: "admin",
})

if (result.success) {
  console.log(result.data)  // fully typed, validated data
} else {
  console.log(result.error.issues)  // array of validation errors
}
```

> 💡 **Key insight**: With Zod, you write the schema ONCE and get both runtime validation AND compile-time types. No more maintaining separate interfaces and validation logic.

---

## C. Hono + Zod Integration — `zValidator`

Hono has official Zod integration:

```bash
bun add @hono/zod-validator
```

```typescript
import { Hono } from "hono"
import { zValidator } from "@hono/zod-validator"
import { z } from "zod"

const app = new Hono()

const createNoteSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  tags: z.array(z.string()).optional().default([]),
})

// Validation as middleware — runs BEFORE your handler
app.post(
  "/api/notes",
  zValidator("json", createNoteSchema),
  async (c) => {
    // If we reach here, body is GUARANTEED valid and typed
    const body = c.req.valid("json")
    //    ^? { title: string; content: string; tags: string[] }

    return c.json({ data: { id: 1, ...body } }, 201)
  }
)
```

### Validating Different Parts of the Request

```typescript
// Validate path params
const idParamSchema = z.object({
  id: z.string().regex(/^\d+$/, "ID must be numeric"),
})

app.get(
  "/api/notes/:id",
  zValidator("param", idParamSchema),
  (c) => {
    const { id } = c.req.valid("param")
    return c.json({ noteId: Number(id) })
  }
)

// Validate query params
const paginationSchema = z.object({
  page: z.string().regex(/^\d+$/).optional().default("1"),
  limit: z.string().regex(/^\d+$/).optional().default("20"),
  sort: z.enum(["date", "title"]).optional().default("date"),
})

app.get(
  "/api/notes",
  zValidator("query", paginationSchema),
  (c) => {
    const query = c.req.valid("query")
    return c.json({
      page: Number(query.page),
      limit: Number(query.limit),
      sort: query.sort,
    })
  }
)

// Validate headers
const authHeaderSchema = z.object({
  authorization: z.string().startsWith("Bearer "),
})

app.get(
  "/api/protected",
  zValidator("header", authHeaderSchema),
  (c) => {
    const { authorization } = c.req.valid("header")
    return c.json({ token: authorization.replace("Bearer ", "") })
  }
)
```

---

## D. Custom Error Responses

By default, `zValidator` returns a 400 with Zod's error format. Customize it:

```typescript
app.post(
  "/api/notes",
  zValidator("json", createNoteSchema, (result, c) => {
    if (!result.success) {
      const errors = result.error.issues.map((issue) => ({
        field: issue.path.join("."),
        message: issue.message,
      }))

      return c.json({
        data: null,
        error: {
          code: "VALIDATION_ERROR",
          message: "Invalid request body",
          details: errors,
        },
      }, 422)
    }
  }),
  async (c) => {
    const body = c.req.valid("json")
    return c.json({ data: body }, 201)
  }
)
```

**Example error response:**

```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request body",
    "details": [
      { "field": "title", "message": "String must contain at least 1 character(s)" },
      { "field": "content", "message": "Required" }
    ]
  }
}
```

---

## E. Schema Design Best Practices

### Organize Schemas by Resource

```
src/
├── schemas/
│   ├── user.schema.ts
│   ├── note.schema.ts
│   └── common.schema.ts     ← shared schemas (pagination, ID params)
```

```typescript
// schemas/note.schema.ts
import { z } from "zod"

export const createNoteSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  tags: z.array(z.string().max(50)).max(10).optional().default([]),
})

export const updateNoteSchema = createNoteSchema.partial()
// partial() makes all fields optional — perfect for PATCH

export type CreateNoteInput = z.infer<typeof createNoteSchema>
export type UpdateNoteInput = z.infer<typeof updateNoteSchema>
```

```typescript
// schemas/common.schema.ts
import { z } from "zod"

export const idParamSchema = z.object({
  id: z.string().regex(/^\d+$/, "ID must be numeric"),
})

export const paginationSchema = z.object({
  page: z.string().regex(/^\d+$/).optional().default("1"),
  limit: z.string().regex(/^\d+$/).optional().default("20"),
})
```

---

## F. Common Mistakes

### Mistake 1: Trusting `c.req.json()` Directly

```typescript
// ❌ No validation — body could be anything
app.post("/api/users", async (c) => {
  const body = await c.req.json()
  db.insert(body)  // SQL injection risk, crash risk
})

// ✅ Always validate first
app.post("/api/users", zValidator("json", userSchema), async (c) => {
  const body = c.req.valid("json")  // guaranteed safe
  db.insert(body)
})
```

### Mistake 2: Forgetting That Query Params Are Strings

```typescript
// ❌ query("page") returns string, not number
const page = c.req.query("page")  // "2", not 2

// ✅ Use z.coerce for automatic type conversion
const querySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
})
```

---

## G. Exercises

### Exercise 1 — Validated Notes API

Take the Notes API from Module 02 and add full Zod validation:
- `POST /api/notes` — validate body (title required, 1-200 chars; content required)
- `PATCH /api/notes/:id` — validate body (all fields optional) and params (numeric ID)
- `GET /api/notes` — validate query params (page, limit, sort)

Return custom error responses with field-level details.

### Exercise 2 — Schema Challenge

Write a Zod schema for a user registration endpoint that validates:
- `email` — valid email, lowercase
- `password` — min 8 chars, must contain uppercase, lowercase, and number
- `username` — 3-20 chars, alphanumeric + underscore only
- `dateOfBirth` — ISO date string, must be 18+ years old

---

## Summary

- ✅ Validation is mandatory — never trust client data
- ✅ Zod — define schema once, get validation + types
- ✅ `zValidator` — validate body, params, query, headers as middleware
- ✅ `c.req.valid("json")` — access validated, typed data
- ✅ Custom error formatting for consistent API responses
- ✅ `.partial()` for PATCH schemas, `z.coerce` for type conversion
- ✅ Schema organization — one file per resource

---

## Navigation

← Previous: [Module 05 — Middleware](./module-05.md)
→ Next: [Module 07 — Error Handling](./module-07.md)
