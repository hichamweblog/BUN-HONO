# Module 10 — Authorization

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [Module 09 — Authentication](./module-09.md)
→ Next: [Module 11 — Testing](./module-11.md)

---

## What You'll Learn

- RBAC (Role-Based Access Control)
- Permission guards as middleware
- Resource ownership checks
- The principle of least privilege

---

## A. Mental Model — Concert Venue Access

```
General Admission ticket → can enter main floor
VIP ticket               → main floor + VIP lounge
Backstage pass           → main floor + VIP + backstage
Artist                   → everywhere + stage

Each area has a GUARD that checks your ticket level.
```

```
In API terms:

GET  /api/posts         → any authenticated user
POST /api/posts         → any authenticated user
DELETE /api/posts/:id   → only the author OR admin
GET  /api/admin/users   → admin only
DELETE /api/admin/users/:id → admin only
```

---

## B. Role-Based Guards

```typescript
// src/middleware/guards.ts
import { createMiddleware } from "hono/factory"

type AuthVariables = {
  userId: number
  userRole: string
}

// Require specific roles
export function requireRole(...roles: string[]) {
  return createMiddleware<{ Variables: AuthVariables }>(async (c, next) => {
    const userRole = c.get("userRole")

    if (!roles.includes(userRole)) {
      return c.json({
        data: null,
        error: { code: "FORBIDDEN", message: "Insufficient permissions" },
      }, 403)
    }

    await next()
  })
}

// Usage:
app.get("/api/admin/users", authMiddleware, requireRole("admin"), handler)
app.delete("/api/admin/users/:id", authMiddleware, requireRole("admin"), handler)
app.post("/api/posts", authMiddleware, requireRole("user", "admin"), handler)
```

---

## C. Resource Ownership

```typescript
// Only allow users to modify their OWN resources
app.patch("/api/posts/:id", authMiddleware, async (c) => {
  const postId = Number(c.req.param("id"))
  const userId = c.get("userId")

  const [post] = await db.select()
    .from(posts)
    .where(eq(posts.id, postId))
    .limit(1)

  if (!post) {
    return c.json({ data: null, error: { code: "NOT_FOUND", message: "Post not found" } }, 404)
  }

  // Ownership check — admins can edit any post
  if (post.authorId !== userId && c.get("userRole") !== "admin") {
    return c.json({
      data: null,
      error: { code: "FORBIDDEN", message: "You can only edit your own posts" },
    }, 403)
  }

  const body = c.req.valid("json")
  const [updated] = await db.update(posts)
    .set({ ...body, updatedAt: new Date() })
    .where(eq(posts.id, postId))
    .returning()

  return c.json({ data: updated, error: null })
})
```

---

## D. Guard Pattern Summary

```
Public routes:       no middleware
Authenticated:       authMiddleware
Role-restricted:     authMiddleware → requireRole("admin")
Owner-only:          authMiddleware → ownership check in handler
Owner OR admin:      authMiddleware → ownership check with admin bypass
```

---

## E. Exercises

### Exercise 1 — RBAC System

Implement a complete authorization system for the blog API:
- Users can create posts and edit/delete only their own
- Admins can edit/delete any post and manage users
- Unauthenticated users can only read published posts

### Exercise 2 — Dynamic Permissions

Design a permission system where roles have specific permissions (`create:post`, `delete:post`, `manage:users`). Implement a `requirePermission()` guard.

---

## Summary

- ✅ Authorization ≠ Authentication
- ✅ RBAC with `requireRole()` middleware
- ✅ Resource ownership checks
- ✅ Admin bypass patterns
- ✅ Guard middleware composition

---

## Navigation

← Previous: [Module 09 — Authentication](./module-09.md)
→ Next: [Module 11 — Testing](./module-11.md)
