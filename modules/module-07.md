# Module 07 — Error Handling

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Intermediate
> **Duration:** ~2 hours

---

## Navigation

← Previous: [Module 06 — Validation](./module-06.md)
→ Next: [Module 08 — Databases](./module-08.md)

---

## What You'll Learn

- Why error handling separates junior from senior code
- Hono's `HTTPException` class
- Global error handler with `app.onError()`
- The `notFound` handler
- Typed, consistent error responses
- Error hierarchy and custom error classes
- Async error handling patterns

---

## A. Mental Model — The Safety Net

```
Without error handling:                With error handling:
                                       
  Request                                Request
    ↓                                      ↓
  Handler                               Handler
    ↓                                      ↓
  💥 CRASH                              ⚠️ Error caught
  Server dies                              ↓
  All users affected                   Error handler
                                           ↓
                                       Clean JSON error response
                                       Server continues running ✅
```

> 🚨 An unhandled error in production doesn't just affect one user — it can crash your entire server process, taking down everyone.

---

## B. Hono's `HTTPException`

Hono provides `HTTPException` for controlled error throwing:

```typescript
import { Hono } from "hono"
import { HTTPException } from "hono/http-exception"

const app = new Hono()

app.get("/api/users/:id", (c) => {
  const id = c.req.param("id")
  const user = findUser(id)

  if (!user) {
    throw new HTTPException(404, { message: "User not found" })
  }

  return c.json({ data: user })
})
```

`HTTPException` is caught automatically by Hono and converted to an error response. You can throw it from anywhere — handlers, middleware, services.

---

## C. Global Error Handler — `app.onError()`

Define one centralized error handler for your entire app:

```typescript
app.onError((err, c) => {
  // Handle known HTTPExceptions
  if (err instanceof HTTPException) {
    return c.json({
      data: null,
      error: {
        code: err.status.toString(),
        message: err.message,
      },
    }, err.status)
  }

  // Handle unknown errors (bugs, crashes)
  console.error("Unhandled error:", err)

  return c.json({
    data: null,
    error: {
      code: "INTERNAL_ERROR",
      message: "An unexpected error occurred",
    },
  }, 500)
})
```

**This achieves three things:**
1. Known errors → clean, specific responses
2. Unknown errors → safe 500 response (no stack trace leak)
3. Server stays alive — no crash

---

## D. Not Found Handler

```typescript
app.notFound((c) => {
  return c.json({
    data: null,
    error: {
      code: "NOT_FOUND",
      message: `Route ${c.req.method} ${c.req.path} not found`,
    },
  }, 404)
})
```

---

## E. Custom Error Classes

For large apps, create a hierarchy of application errors:

```typescript
// errors/app-error.ts
export class AppError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly code: string,
    message: string,
  ) {
    super(message)
    this.name = "AppError"
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string | number) {
    super(404, "NOT_FOUND", `${resource} with ID ${id} not found`)
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(409, "CONFLICT", message)
  }
}

export class ForbiddenError extends AppError {
  constructor(message = "You don't have permission") {
    super(403, "FORBIDDEN", message)
  }
}
```

```typescript
// Updated global error handler
app.onError((err, c) => {
  if (err instanceof AppError) {
    return c.json({
      data: null,
      error: { code: err.code, message: err.message },
    }, err.statusCode)
  }

  if (err instanceof HTTPException) {
    return c.json({
      data: null,
      error: { code: String(err.status), message: err.message },
    }, err.status)
  }

  console.error("Unhandled:", err)
  return c.json({
    data: null,
    error: { code: "INTERNAL_ERROR", message: "An unexpected error occurred" },
  }, 500)
})
```

Now in handlers:

```typescript
app.get("/api/users/:id", (c) => {
  const user = findUser(c.req.param("id"))
  if (!user) throw new NotFoundError("User", c.req.param("id"))
  return c.json({ data: user })
})

app.post("/api/users", async (c) => {
  const body = c.req.valid("json")
  const existing = findByEmail(body.email)
  if (existing) throw new ConflictError("Email already registered")
  // ...create user
})
```

> ✅ **This is the production pattern.** Throw specific errors in your business logic, catch them centrally. Your handlers stay clean.

---

## F. Common Mistakes

### Mistake 1: Swallowing Errors

```typescript
// ❌ Catches error but returns nothing useful
app.get("/api/users", async (c) => {
  try {
    const users = await getUsers()
    return c.json({ data: users })
  } catch (error) {
    return c.json({ error: "Something went wrong" }, 500)
    // What went wrong? No one knows. No logging!
  }
})

// ✅ Log the error, return clean response
app.get("/api/users", async (c) => {
  try {
    const users = await getUsers()
    return c.json({ data: users })
  } catch (error) {
    console.error("Failed to fetch users:", error)  // ← for your logs
    return c.json({
      data: null,
      error: { code: "DB_ERROR", message: "Failed to fetch users" },
    }, 500)  // ← for the client
  }
})
```

### Mistake 2: Exposing Internal Details

```typescript
// ❌ Stack traces reveal file paths, library versions, DB structure
return c.json({ error: error.stack }, 500)

// ✅ Generic message to client, details in logs
console.error(error)  // full error in server logs
return c.json({
  error: { code: "INTERNAL_ERROR", message: "Internal server error" },
}, 500)
```

---

## G. Exercises

### Exercise 1 — Error Hierarchy

Create an error class hierarchy for a blog API: `NotFoundError`, `ValidationError`, `UnauthorizedError`, `ConflictError`. Implement a global error handler that handles all of them. Throw them from different routes and verify the responses.

### Exercise 2 — Error Middleware

Write middleware that wraps every handler in a try-catch and logs errors with the request ID, method, path, and duration.

---

## Summary

- ✅ `HTTPException` for throwing controlled HTTP errors
- ✅ `app.onError()` for centralized error handling
- ✅ `app.notFound()` for 404 responses
- ✅ Custom error classes for clean business logic
- ✅ Never expose stack traces — log internally, return safe messages
- ✅ Always log errors — silent failures are the worst kind

---

## Navigation

← Previous: [Module 06 — Validation](./module-06.md)
→ Next: [Module 08 — Databases](./module-08.md)
