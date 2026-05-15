# Module 09 — Authentication

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Intermediate → Advanced
> **Duration:** ~3–4 hours
> **Project:** Auth API

---

## Navigation

← Previous: [Module 08 — Databases](./module-08.md)
→ Next: [Module 10 — Authorization](./module-10.md)

---

## What You'll Learn

- Authentication vs Authorization (the critical difference)
- JWT (JSON Web Tokens) — how they work, structure, and tradeoffs
- Implementing signup, login, and protected routes
- Password hashing with Bun's built-in Argon2
- Refresh tokens
- Cookie-based vs header-based auth
- JWT security best practices

---

## A. Mental Model — ID Card vs Access Badge

```
Authentication = "WHO are you?"
  → Proving your identity (login)
  → Like showing your government ID at the door

Authorization = "WHAT can you do?"
  → Checking your permissions (Module 10)
  → Like having a VIP badge vs general admission
```

```
Login Flow:

  Client                                    Server
    │                                          │
    │  POST /auth/login                        │
    │  { email, password }                     │
    │ ───────────────────────────────────────→  │
    │                                          │  1. Find user by email
    │                                          │  2. Verify password hash
    │                                          │  3. Generate JWT token
    │                                          │
    │  { token: "eyJhbGc..." }                 │
    │ ←───────────────────────────────────────  │
    │                                          │
    │  GET /api/users/me                       │
    │  Authorization: Bearer eyJhbGc...        │
    │ ───────────────────────────────────────→  │
    │                                          │  4. Verify JWT
    │                                          │  5. Extract user from token
    │  { data: { id: 1, name: "Alice" } }      │
    │ ←───────────────────────────────────────  │
```

---

## B. JWT — How It Works

A JWT has three parts, separated by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxIiwiZW1haWwiOiJhbGljZUBleGFtcGxlLmNvbSJ9.signature

 ──────── Header ──────── . ──────────── Payload ──────────── . ── Signature ──

Header:   { "alg": "HS256", "typ": "JWT" }        → base64 encoded
Payload:  { "sub": "1", "email": "alice@..." }     → base64 encoded
Signature: HMACSHA256(header + "." + payload, SECRET_KEY)
```

> ⚠️ **JWTs are NOT encrypted.** The payload is just base64-encoded — anyone can read it. The signature proves it wasn't tampered with. Never put secrets (passwords, credit cards) in a JWT payload.

---

## C. Implementation — Signup & Login

### Password Hashing Service

```typescript
// src/services/auth.service.ts
export async function hashPassword(password: string): Promise<string> {
  return Bun.password.hash(password, { algorithm: "argon2id" })
}

export async function verifyPassword(
  password: string,
  hash: string
): Promise<boolean> {
  return Bun.password.verify(password, hash)
}
```

### JWT Utilities

```typescript
// src/lib/jwt.ts
import { sign, verify } from "hono/jwt"

const JWT_SECRET = Bun.env.JWT_SECRET!

export interface JWTPayload {
  sub: string        // user ID
  email: string
  role: string
  exp: number        // expiration timestamp
}

export async function createToken(user: {
  id: number
  email: string
  role: string
}): Promise<string> {
  const payload: JWTPayload = {
    sub: String(user.id),
    email: user.email,
    role: user.role,
    exp: Math.floor(Date.now() / 1000) + 60 * 60, // 1 hour
  }

  return sign(payload, JWT_SECRET)
}

export async function verifyToken(token: string): Promise<JWTPayload> {
  return verify(token, JWT_SECRET) as Promise<JWTPayload>
}
```

### Auth Routes

```typescript
// src/routes/auth.ts
import { Hono } from "hono"
import { zValidator } from "@hono/zod-validator"
import { z } from "zod"
import { db } from "../db"
import { users } from "../db/schema"
import { eq } from "drizzle-orm"
import { hashPassword, verifyPassword } from "../services/auth.service"
import { createToken } from "../lib/jwt"

const auth = new Hono()

const signupSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  password: z.string().min(8),
})

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
})

// POST /auth/signup
auth.post("/signup", zValidator("json", signupSchema), async (c) => {
  const body = c.req.valid("json")

  // Check for existing user
  const existing = await db.select()
    .from(users)
    .where(eq(users.email, body.email))
    .limit(1)

  if (existing.length > 0) {
    return c.json({
      data: null,
      error: { code: "CONFLICT", message: "Email already registered" },
    }, 409)
  }

  // Create user
  const [user] = await db.insert(users).values({
    name: body.name,
    email: body.email,
    passwordHash: await hashPassword(body.password),
  }).returning({ id: users.id, name: users.name, email: users.email, role: users.role })

  const token = await createToken(user)

  return c.json({
    data: { user, token },
    error: null,
  }, 201)
})

// POST /auth/login
auth.post("/login", zValidator("json", loginSchema), async (c) => {
  const body = c.req.valid("json")

  const [user] = await db.select()
    .from(users)
    .where(eq(users.email, body.email))
    .limit(1)

  if (!user) {
    return c.json({
      data: null,
      error: { code: "UNAUTHORIZED", message: "Invalid credentials" },
    }, 401)
  }

  const valid = await verifyPassword(body.password, user.passwordHash)

  if (!valid) {
    // Same error message — don't reveal if email exists
    return c.json({
      data: null,
      error: { code: "UNAUTHORIZED", message: "Invalid credentials" },
    }, 401)
  }

  const token = await createToken(user)

  return c.json({
    data: {
      user: { id: user.id, name: user.name, email: user.email, role: user.role },
      token,
    },
    error: null,
  })
})

export default auth
```

> 🚨 **Security Rule**: Always return the same error message for "email not found" and "wrong password." If you say "email not found," attackers can enumerate which emails are registered.

### Auth Middleware

```typescript
// src/middleware/auth.ts
import { createMiddleware } from "hono/factory"
import { verifyToken } from "../lib/jwt"

type AuthVariables = {
  userId: number
  userEmail: string
  userRole: string
}

export const authMiddleware = createMiddleware<{
  Variables: AuthVariables
}>(async (c, next) => {
  const header = c.req.header("Authorization")

  if (!header?.startsWith("Bearer ")) {
    return c.json({
      data: null,
      error: { code: "UNAUTHORIZED", message: "Missing or invalid token" },
    }, 401)
  }

  try {
    const token = header.replace("Bearer ", "")
    const payload = await verifyToken(token)

    c.set("userId", Number(payload.sub))
    c.set("userEmail", payload.email)
    c.set("userRole", payload.role)

    await next()
  } catch {
    return c.json({
      data: null,
      error: { code: "UNAUTHORIZED", message: "Invalid or expired token" },
    }, 401)
  }
})
```

---

## D. JWT Security Best Practices

```
✅ DO:
• Use strong secrets (32+ random bytes): openssl rand -hex 32
• Set short expiration (15min–1hr for access tokens)
• Use HTTPS in production (tokens in plain HTTP can be intercepted)
• Validate ALL claims (exp, sub, etc.)

❌ DON'T:
• Store sensitive data in JWT payload (it's readable)
• Use JWT for sessions that need instant revocation
• Use weak secrets like "mysecret123"
• Set tokens to never expire
```

> ⚠️ **JWT limitation**: JWTs can't be revoked once issued (they're stateless). For logout/revocation, you need either short-lived tokens with refresh tokens, or a token blacklist.

---

## E. Exercises

### Exercise 1 — Full Auth Flow

Implement the complete auth system:
1. `POST /auth/signup` — create user, return token
2. `POST /auth/login` — verify credentials, return token
3. `GET /api/me` — protected route, return current user

### Exercise 2 — Refresh Tokens

Extend the auth system with refresh tokens:
- Access token expires in 15 minutes
- Refresh token expires in 7 days
- `POST /auth/refresh` — exchanges refresh token for new access token

---

## Summary

- ✅ Authentication = proving identity (login)
- ✅ JWT structure — header, payload, signature
- ✅ Password hashing with Bun's Argon2id
- ✅ Signup/login flow with proper error handling
- ✅ Auth middleware — extract and verify tokens
- ✅ Security rules — same error messages, strong secrets, short expiry

---

## Navigation

← Previous: [Module 08 — Databases](./module-08.md)
→ Next: [Module 10 — Authorization](./module-10.md)
