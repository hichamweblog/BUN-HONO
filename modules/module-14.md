# Module 14 — Performance

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [Module 13 — Security](./module-13.md)
→ Next: [Module 15 — OpenAPI](./module-15.md)

---

## What You'll Learn

- Where time is spent in a request lifecycle
- Database query optimization and indexing
- Caching strategies (in-memory and HTTP)
- Streaming responses
- Compression
- Pagination performance
- Bun-specific performance tips

---

## A. Mental Model — Where Does Time Go?

```
Total request time: 150ms

  Network latency:           50ms  ████████████░░░░░░░░░░░░░░░  (33%)
  Middleware chain:            2ms  █░░░░░░░░░░░░░░░░░░░░░░░░░░  (1%)
  Input validation:            1ms  ░░░░░░░░░░░░░░░░░░░░░░░░░░░  (<1%)
  Database query:             80ms  ████████████████████░░░░░░░  (53%)
  Business logic:              5ms  ██░░░░░░░░░░░░░░░░░░░░░░░░░  (3%)
  JSON serialization:          2ms  █░░░░░░░░░░░░░░░░░░░░░░░░░░  (1%)
  Network (response):        10ms  ███░░░░░░░░░░░░░░░░░░░░░░░░  (7%)

The database is almost always the bottleneck.
```

> 💡 **Optimize the biggest slice first.** A 50% faster DB query saves 40ms. Making your handler 50% faster saves 2.5ms. Focus on what matters.

---

## B. Database Optimization

### Indexing

```typescript
// Every foreign key should have an index
// Every column you filter by frequently should have an index

export const posts = pgTable("posts", {
  id: serial("id").primaryKey(),
  title: text("title").notNull(),
  authorId: integer("author_id").notNull().references(() => users.id),
  published: boolean("published").notNull().default(false),
  createdAt: timestamp("created_at").notNull().defaultNow(),
}, (table) => ({
  authorIdx: index("idx_posts_author_id").on(table.authorId),
  publishedIdx: index("idx_posts_published").on(table.published),
  createdAtIdx: index("idx_posts_created_at").on(table.createdAt),
}))
```

### Select Only What You Need

```typescript
// ❌ Fetches ALL columns (including password hash!)
const users = await db.select().from(users)

// ✅ Select only needed columns
const users = await db.select({
  id: users.id,
  name: users.name,
  email: users.email,
}).from(users)
```

### Pagination — Offset vs Cursor

```typescript
// Offset pagination — simple but slow on large datasets
// Page 10,000 still scans 200,000 rows (offset = 200000)
const posts = await db.select().from(posts)
  .orderBy(desc(posts.createdAt))
  .limit(20)
  .offset((page - 1) * 20)

// Cursor pagination — constant performance regardless of page
// Uses the last item's ID/timestamp as a cursor
const posts = await db.select().from(posts)
  .where(lt(posts.createdAt, cursorTimestamp))
  .orderBy(desc(posts.createdAt))
  .limit(20)
```

> ✅ **Use cursor pagination for large datasets** (feeds, search results, logs). Use offset for small datasets or admin panels where users jump to specific pages.

---

## C. Caching

### HTTP Cache Headers

```typescript
// Static or rarely-changing data
app.get("/api/config", (c) => {
  c.header("Cache-Control", "public, max-age=3600")  // cache 1 hour
  return c.json({ data: appConfig })
})

// User-specific data — don't cache in shared caches
app.get("/api/me", authMiddleware, (c) => {
  c.header("Cache-Control", "private, no-cache")
  return c.json({ data: user })
})

// Never cache
app.post("/api/users", (c) => {
  c.header("Cache-Control", "no-store")
  // ...
})
```

### In-Memory Cache (Simple)

```typescript
const cache = new Map<string, { data: unknown; expiresAt: number }>()

function getCached<T>(key: string, ttlMs: number, fetchFn: () => Promise<T>): Promise<T> {
  const cached = cache.get(key)
  if (cached && Date.now() < cached.expiresAt) {
    return cached.data as T
  }

  return fetchFn().then((data) => {
    cache.set(key, { data, expiresAt: Date.now() + ttlMs })
    return data
  })
}

// Usage
app.get("/api/stats", async (c) => {
  const stats = await getCached("app-stats", 60_000, async () => {
    return db.select({ total: count() }).from(users)
  })
  return c.json({ data: stats })
})
```

---

## D. Streaming Responses

For large datasets, stream instead of buffering:

```typescript
import { streamText, streamSSE } from "hono/streaming"

// Stream a large response
app.get("/api/export", (c) => {
  return streamText(c, async (stream) => {
    // Write CSV header
    await stream.write("id,name,email\n")

    // Stream rows in batches
    let offset = 0
    const batchSize = 1000
    while (true) {
      const batch = await db.select().from(users).limit(batchSize).offset(offset)
      if (batch.length === 0) break

      for (const user of batch) {
        await stream.write(`${user.id},${user.name},${user.email}\n`)
      }
      offset += batchSize
    }
  })
})
```

---

## E. Compression

```typescript
import { compress } from "hono/compress"

// Compress responses > 1KB automatically
app.use(compress())
// Supports gzip, deflate, and brotli
// Typical compression ratio: 70-90% for JSON/text
```

---

## F. Performance Checklist

```
✅ Index foreign keys and frequently filtered columns
✅ Select only needed columns
✅ Use cursor pagination for large datasets
✅ Cache expensive queries (in-memory or HTTP cache headers)
✅ Enable compression for API responses
✅ Stream large responses instead of buffering
✅ Use connection pooling for database
✅ Avoid N+1 queries (use joins/relations)
✅ Monitor query performance (Drizzle Studio, logging)
```

---

## G. Exercises

### Exercise 1 — Optimize Queries

Profile the blog API. Add indexes, convert offset pagination to cursor-based, and add appropriate `Cache-Control` headers to each endpoint.

### Exercise 2 — Benchmark

Use a tool like `wrk` or `bombardier` to benchmark your API:
```bash
# Install bombardier
bun install -g bombardier

# Benchmark
bombardier -c 50 -d 10s http://localhost:3000/api/posts
```

Compare before and after your optimizations.

---

## Summary

- ✅ Database is usually the bottleneck — optimize there first
- ✅ Indexing — O(log n) instead of O(n) lookups
- ✅ Cursor pagination for large datasets
- ✅ HTTP caching with Cache-Control headers
- ✅ In-memory caching for expensive computations
- ✅ Streaming for large responses
- ✅ Compression reduces transfer size by 70-90%

---

## Navigation

← Previous: [Module 13 — Security](./module-13.md)
→ Next: [Module 15 — OpenAPI](./module-15.md)
