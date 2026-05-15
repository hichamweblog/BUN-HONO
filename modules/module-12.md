# Module 12 — Production Architecture

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~3–4 hours

---

## Navigation

← Previous: [Module 11 — Testing](./module-11.md)
→ Next: [Module 13 — Security](./module-13.md)

---

## What You'll Learn

- Layered architecture — why and how
- Separation of concerns (routes → services → repositories)
- Dependency injection patterns
- Production folder structure
- Configuration management
- Logging strategy
- API response consistency

---

## A. Mental Model — The Factory Assembly Line

```
Without architecture (spaghetti):
  Route handler does EVERYTHING:
    validates → queries DB → hashes passwords → sends emails → builds response
    → 300-line handler functions
    → untestable, unmaintainable

With layered architecture:
  Route Handler   → "What should happen?" (thin, orchestrates)
  Service Layer   → "How to do it" (business logic)
  Repository      → "Where to get/put data" (database access)
  
  Each layer has ONE job. Each is testable independently.
```

---

## B. The Three Layers

```
  ┌──────────────────────────────────────────┐
  │  Routes / Controllers                     │
  │  • Parse request (already validated)      │
  │  • Call service methods                   │
  │  • Return response                        │
  │  • NO business logic                      │
  │  • NO direct database access              │
  └──────────────────┬───────────────────────┘
                     │ calls
  ┌──────────────────▼───────────────────────┐
  │  Services                                 │
  │  • Business logic lives here              │
  │  • Validation rules                       │
  │  • Orchestrates multiple repositories     │
  │  • Throws business errors                 │
  │  • NO HTTP concepts (no c.json, no 404)   │
  └──────────────────┬───────────────────────┘
                     │ calls
  ┌──────────────────▼───────────────────────┐
  │  Repositories / Data Access               │
  │  • Database queries only                  │
  │  • Pure data operations                   │
  │  • Returns plain objects                  │
  │  • NO business logic                      │
  └──────────────────────────────────────────┘
```

---

## C. Production Folder Structure

```
src/
├── index.ts                    ← app entry point
├── app.ts                      ← Hono app setup (middleware, routes)
├── config/
│   └── env.ts                  ← validated environment config
├── db/
│   ├── index.ts                ← database connection
│   ├── schema.ts               ← Drizzle schema
│   └── migrations/             ← migration files
├── routes/
│   ├── auth.routes.ts          ← auth endpoints
│   ├── user.routes.ts          ← user endpoints
│   └── post.routes.ts          ← post endpoints
├── services/
│   ├── auth.service.ts         ← auth business logic
│   ├── user.service.ts         ← user business logic
│   └── post.service.ts         ← post business logic
├── repositories/
│   ├── user.repository.ts      ← user DB queries
│   └── post.repository.ts      ← post DB queries
├── middleware/
│   ├── auth.ts                 ← JWT verification
│   ├── guards.ts               ← role guards
│   └── error-handler.ts        ← global error handler
├── schemas/
│   ├── user.schema.ts          ← Zod schemas for user
│   └── post.schema.ts          ← Zod schemas for post
├── errors/
│   └── app-error.ts            ← custom error classes
├── types/
│   └── index.ts                ← shared types
└── lib/
    ├── jwt.ts                  ← JWT utilities
    └── response.ts             ← response helpers
```

---

## D. Layer Implementation Example

### Repository — Pure Data Access

```typescript
// src/repositories/user.repository.ts
import { db } from "../db"
import { users } from "../db/schema"
import { eq } from "drizzle-orm"

export const userRepository = {
  async findById(id: number) {
    const [user] = await db.select().from(users).where(eq(users.id, id)).limit(1)
    return user ?? null
  },

  async findByEmail(email: string) {
    const [user] = await db.select().from(users).where(eq(users.email, email)).limit(1)
    return user ?? null
  },

  async create(data: { name: string; email: string; passwordHash: string }) {
    const [user] = await db.insert(users).values(data).returning()
    return user
  },

  async update(id: number, data: Partial<{ name: string; email: string }>) {
    const [user] = await db.update(users)
      .set({ ...data, updatedAt: new Date() })
      .where(eq(users.id, id))
      .returning()
    return user ?? null
  },
}
```

### Service — Business Logic

```typescript
// src/services/user.service.ts
import { userRepository } from "../repositories/user.repository"
import { NotFoundError, ConflictError } from "../errors/app-error"

export const userService = {
  async getById(id: number) {
    const user = await userRepository.findById(id)
    if (!user) throw new NotFoundError("User", id)

    // Don't expose password hash
    const { passwordHash, ...safeUser } = user
    return safeUser
  },

  async updateProfile(id: number, data: { name?: string; email?: string }) {
    if (data.email) {
      const existing = await userRepository.findByEmail(data.email)
      if (existing && existing.id !== id) {
        throw new ConflictError("Email already in use")
      }
    }

    const user = await userRepository.update(id, data)
    if (!user) throw new NotFoundError("User", id)

    const { passwordHash, ...safeUser } = user
    return safeUser
  },
}
```

### Route — Thin Controller

```typescript
// src/routes/user.routes.ts
import { Hono } from "hono"
import { zValidator } from "@hono/zod-validator"
import { authMiddleware } from "../middleware/auth"
import { userService } from "../services/user.service"
import { updateUserSchema, idParamSchema } from "../schemas/user.schema"

const userRoutes = new Hono()

userRoutes.use(authMiddleware)

userRoutes.get("/me", async (c) => {
  const userId = c.get("userId")
  const user = await userService.getById(userId)
  return c.json({ data: user, error: null })
})

userRoutes.patch(
  "/me",
  zValidator("json", updateUserSchema),
  async (c) => {
    const userId = c.get("userId")
    const body = c.req.valid("json")
    const user = await userService.updateProfile(userId, body)
    return c.json({ data: user, error: null })
  }
)

export default userRoutes
```

> 💡 **Notice**: The route handler is ~5 lines. It validates input, calls the service, and returns the response. All business logic is in the service. All DB access is in the repository. Each piece is independently testable.

---

## E. Environment Configuration

```typescript
// src/config/env.ts
import { z } from "zod"

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  CORS_ORIGIN: z.string().default("http://localhost:3000"),
})

const parsed = envSchema.safeParse(Bun.env)

if (!parsed.success) {
  console.error("❌ Invalid environment variables:")
  console.error(parsed.error.flatten().fieldErrors)
  process.exit(1)
}

export const env = parsed.data
```

> ✅ **Validate env vars at startup.** Fail fast with clear errors rather than crashing 3 hours later when someone hits the DB endpoint and DATABASE_URL is undefined.

---

## F. Exercises

### Exercise 1 — Refactor to Layers

Take the Notes API from earlier modules and refactor it into the three-layer architecture:
- `note.repository.ts` — all Drizzle queries
- `note.service.ts` — business logic, error throwing
- `note.routes.ts` — thin route handlers

### Exercise 2 — Environment Validation

Create `src/config/env.ts` that validates all required environment variables at startup using Zod. Make the app refuse to start if any are missing.

---

## Summary

- ✅ Three-layer architecture — routes, services, repositories
- ✅ Each layer has ONE job — no business logic in routes
- ✅ Production folder structure
- ✅ Environment validation at startup
- ✅ This pattern makes code testable, maintainable, and scalable

---

## Navigation

← Previous: [Module 11 — Testing](./module-11.md)
→ Next: [Module 13 — Security](./module-13.md)
