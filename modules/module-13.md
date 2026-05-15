# Module 13 — Security

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [Module 12 — Production Architecture](./module-12.md)
→ Next: [Module 14 — Performance](./module-14.md)

---

## What You'll Learn

- CORS — what it is and how to configure it properly
- CSRF protection
- Rate limiting
- Security headers
- Input sanitization
- SQL injection prevention (with ORMs)
- XSS prevention
- Helmet-style protection in Hono

---

## A. Mental Model — The Castle Defense

```
Your API is a castle. Security is layers of defense:

  🏰 Castle Wall (HTTPS / TLS)
    └── 🛡️ Gate Guards (Rate Limiting)
         └── 📋 ID Check (Authentication — Module 09)
              └── 🎫 Permission Check (Authorization — Module 10)
                   └── 🔍 Luggage Scan (Input Validation — Module 06)
                        └── 🏠 Inner Chambers (Your Business Logic)

No single defense is enough. Security is LAYERS.
```

---

## B. CORS — Cross-Origin Resource Sharing

### What is CORS?

Browsers block JavaScript from making requests to different domains by default. CORS headers tell browsers "it's okay, this origin is allowed."

```typescript
import { cors } from "hono/cors"

// ❌ Development only — allows everything
app.use("/api/*", cors())

// ✅ Production — explicit whitelist
app.use("/api/*", cors({
  origin: ["https://myapp.com", "https://admin.myapp.com"],
  allowMethods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
  allowHeaders: ["Content-Type", "Authorization"],
  exposeHeaders: ["X-Request-ID", "X-Rate-Limit-Remaining"],
  credentials: true,   // allow cookies/auth headers
  maxAge: 86400,        // cache preflight for 24 hours
}))
```

> 🚨 **Never use `origin: "*"` with `credentials: true`** in production. This is a security vulnerability — any site could make authenticated requests to your API.

---

## C. Rate Limiting

Prevent abuse by limiting requests per client:

```typescript
// Simple in-memory rate limiter (for single-server deployments)
const rateLimitStore = new Map<string, { count: number; resetAt: number }>()

const rateLimiter = createMiddleware(async (c, next) => {
  const ip = c.req.header("x-forwarded-for") ?? "unknown"
  const now = Date.now()
  const windowMs = 60_000  // 1 minute
  const maxRequests = 100

  const entry = rateLimitStore.get(ip)

  if (!entry || now > entry.resetAt) {
    rateLimitStore.set(ip, { count: 1, resetAt: now + windowMs })
  } else if (entry.count >= maxRequests) {
    c.header("Retry-After", String(Math.ceil((entry.resetAt - now) / 1000)))
    return c.json({
      data: null,
      error: { code: "RATE_LIMITED", message: "Too many requests" },
    }, 429)
  } else {
    entry.count++
  }

  c.header("X-Rate-Limit-Limit", String(maxRequests))
  c.header("X-Rate-Limit-Remaining", String(
    maxRequests - (rateLimitStore.get(ip)?.count ?? 0)
  ))

  await next()
})

app.use("/api/*", rateLimiter)
```

> ⚠️ For multi-server deployments, use Redis-backed rate limiting instead of in-memory.

---

## D. Security Headers

```typescript
import { secureHeaders } from "hono/secure-headers"

app.use(secureHeaders({
  // Prevent the page from being embedded in iframes (clickjacking)
  xFrameOptions: "DENY",
  // Prevent MIME-type sniffing
  xContentTypeOptions: "nosniff",
  // Control referrer information
  referrerPolicy: "strict-origin-when-cross-origin",
  // Force HTTPS
  strictTransportSecurity: "max-age=31536000; includeSubDomains",
}))
```

---

## E. Input Sanitization

ORMs like Drizzle use **parameterized queries** by default, preventing SQL injection:

```typescript
// ✅ Drizzle — automatically parameterized (SAFE)
const user = await db.select().from(users).where(eq(users.email, userInput))
// Generated: SELECT * FROM users WHERE email = $1
// $1 is bound separately — no injection possible

// ❌ Raw SQL with string concatenation (DANGEROUS)
const user = await db.execute(`SELECT * FROM users WHERE email = '${userInput}'`)
// If userInput = "'; DROP TABLE users; --"  →  💥 DATA DELETED
```

> ✅ **Rule**: Always use parameterized queries. ORMs do this for you. If you write raw SQL, use parameter bindings — never string concatenation.

---

## F. CSRF Protection

```typescript
import { csrf } from "hono/csrf"

// Protects against cross-site request forgery
app.use(csrf({
  origin: ["https://myapp.com"],
}))
```

---

## G. Security Checklist

```
✅ HTTPS everywhere (TLS termination at load balancer or Bun)
✅ CORS configured with explicit origins
✅ Rate limiting on all API endpoints
✅ Security headers (secureHeaders middleware)
✅ Input validation on every endpoint (Zod — Module 06)
✅ Parameterized queries (Drizzle handles this)
✅ Password hashing with Argon2id (never store plaintext)
✅ JWT secrets stored in env vars (never in code)
✅ Error messages don't leak internals
✅ Dependencies audited regularly (bun audit)
✅ CSRF protection for cookie-based auth
```

---

## H. Exercises

### Exercise 1 — Security Hardening

Take the blog API and add:
1. CORS with explicit origin whitelist
2. Rate limiting (100 req/min)
3. Secure headers
4. CSRF protection

### Exercise 2 — Security Audit

Review your existing code for:
- Any place where user input is used without validation
- Any place where error details are exposed to clients
- Any hardcoded secrets

---

## Summary

- ✅ CORS — explicit origin whitelist in production
- ✅ Rate limiting — protect against abuse
- ✅ Security headers — prevent clickjacking, MIME sniffing
- ✅ Parameterized queries — prevent SQL injection
- ✅ CSRF — protect cookie-based auth
- ✅ Security is layers — no single defense is enough

---

## Navigation

← Previous: [Module 12 — Production Architecture](./module-12.md)
→ Next: [Module 14 — Performance](./module-14.md)
