# Module 08 — Databases

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Intermediate → Advanced
> **Duration:** ~4–5 hours
> **Project:** Blog Backend (database layer)

---

## Navigation

← Previous: [Module 07 — Error Handling](./module-07.md)
→ Next: [Module 09 — Authentication](./module-09.md)

---

## What You'll Learn

- SQL fundamentals for backend engineers
- PostgreSQL setup (local or Docker)
- Drizzle ORM — schema definition, queries, relations
- Database migrations
- Transactions
- Connection management
- Schema design and normalization
- Indexing and query performance basics

---

## A. Mental Model — The Structured Filing Cabinet

```
In-Memory Array (what we've been doing):
┌──────────────────────────────────┐
│  const users = [...]             │
│  • Lost when server restarts     │
│  • No concurrent access safety   │
│  • No querying beyond .filter()  │
│  • Doesn't scale past RAM       │
└──────────────────────────────────┘

Database (what production uses):
┌──────────────────────────────────┐
│  PostgreSQL                      │
│  • Data persists across restarts │
│  • ACID transactions             │
│  • Powerful SQL querying         │
│  • Concurrent access handled     │
│  • Scales to billions of rows    │
│  • Indexes for fast lookups      │
└──────────────────────────────────┘
```

---

## B. PostgreSQL Setup

### Option 1: Docker (Recommended)

```bash
docker run -d \
  --name postgres-dev \
  -e POSTGRES_USER=dev \
  -e POSTGRES_PASSWORD=devpass \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:16-alpine
```

### Option 2: Neon (Serverless — No Install)

1. Go to [neon.tech](https://neon.tech)
2. Create a free project
3. Copy the connection string

Add to `.env`:

```
DATABASE_URL=postgres://dev:devpass@localhost:5432/myapp
```

---

## C. Drizzle ORM Setup

```bash
bun add drizzle-orm postgres
bun add -d drizzle-kit
```

### Schema Definition

```typescript
// src/db/schema.ts
import { pgTable, serial, text, timestamp, integer, boolean } from "drizzle-orm/pg-core"

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  passwordHash: text("password_hash").notNull(),
  role: text("role", { enum: ["user", "admin"] }).notNull().default("user"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
})

export const posts = pgTable("posts", {
  id: serial("id").primaryKey(),
  title: text("title").notNull(),
  content: text("content").notNull(),
  published: boolean("published").notNull().default(false),
  authorId: integer("author_id").notNull().references(() => users.id),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
})

export const comments = pgTable("comments", {
  id: serial("id").primaryKey(),
  content: text("content").notNull(),
  postId: integer("post_id").notNull().references(() => posts.id),
  authorId: integer("author_id").notNull().references(() => users.id),
  createdAt: timestamp("created_at").notNull().defaultNow(),
})
```

### Database Connection

```typescript
// src/db/index.ts
import { drizzle } from "drizzle-orm/postgres-js"
import postgres from "postgres"
import * as schema from "./schema"

const connectionString = Bun.env.DATABASE_URL!

// Connection pool
const client = postgres(connectionString, {
  max: 10,  // max connections in pool
})

export const db = drizzle(client, { schema })
```

### Drizzle Config

```typescript
// drizzle.config.ts
import { defineConfig } from "drizzle-kit"

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./src/db/migrations",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
})
```

---

## D. Migrations

```bash
# Generate migration from schema changes
bunx drizzle-kit generate

# Apply migrations to database
bunx drizzle-kit migrate

# Open Drizzle Studio (visual DB browser)
bunx drizzle-kit studio
```

> ✅ **Always use migrations.** Never modify production databases manually. Migrations are version-controlled, reversible, and auditable.

---

## E. CRUD Operations with Drizzle

```typescript
import { db } from "./db"
import { users, posts } from "./db/schema"
import { eq, desc, like, and, count } from "drizzle-orm"

// ─── CREATE ─────────────────────────────
const newUser = await db.insert(users).values({
  name: "Alice",
  email: "alice@example.com",
  passwordHash: await Bun.password.hash("secret123"),
}).returning()
// returning() gives back the inserted row with generated ID

// ─── READ (single) ─────────────────────
const user = await db.select()
  .from(users)
  .where(eq(users.id, 1))
  .limit(1)
// Returns array — use [0] or .then(rows => rows[0])

// ─── READ (list with pagination) ────────
const page = 1
const limit = 20
const userList = await db.select()
  .from(users)
  .orderBy(desc(users.createdAt))
  .limit(limit)
  .offset((page - 1) * limit)

// ─── READ (with conditions) ─────────────
const admins = await db.select()
  .from(users)
  .where(and(
    eq(users.role, "admin"),
    like(users.name, "%ali%")
  ))

// ─── UPDATE ─────────────────────────────
const updated = await db.update(users)
  .set({ name: "Alice Updated", updatedAt: new Date() })
  .where(eq(users.id, 1))
  .returning()

// ─── DELETE ─────────────────────────────
await db.delete(users).where(eq(users.id, 1))

// ─── COUNT ──────────────────────────────
const [{ total }] = await db.select({ total: count() }).from(users)
```

---

## F. Relations and Joins

```typescript
// Query with relations (Drizzle relational queries)
import { relations } from "drizzle-orm"

// Define relations
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}))

export const postsRelations = relations(posts, ({ one, many }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
  comments: many(comments),
}))

// Use relational query API
const postsWithAuthor = await db.query.posts.findMany({
  with: { author: true },
  where: eq(posts.published, true),
  orderBy: [desc(posts.createdAt)],
  limit: 10,
})
// Each post now includes { author: { id, name, email, ... } }
```

---

## G. Transactions

```typescript
// When multiple operations must succeed or fail together
await db.transaction(async (tx) => {
  const [user] = await tx.insert(users).values({
    name: "Bob",
    email: "bob@example.com",
    passwordHash: "...",
  }).returning()

  await tx.insert(posts).values({
    title: "My First Post",
    content: "Hello world!",
    authorId: user.id,
  })

  // If either fails, BOTH are rolled back
})
```

> 💡 **When to use transactions:** Creating related records (user + profile), transferring money (debit + credit), any operation where partial completion would leave data inconsistent.

---

## H. Schema Design Principles

```
1. Every table has a primary key (serial id or UUID)
2. Use foreign keys to establish relationships
3. Add NOT NULL unless the field is truly optional
4. Add UNIQUE constraints for emails, usernames, slugs
5. Add created_at and updated_at to every table
6. Use enums for fixed sets of values (role, status)
7. Index columns you frequently filter/sort by
```

### Indexing

```typescript
import { index } from "drizzle-orm/pg-core"

export const posts = pgTable("posts", {
  // ...columns
}, (table) => ({
  authorIdx: index("posts_author_id_idx").on(table.authorId),
  publishedIdx: index("posts_published_idx").on(table.published),
}))
```

> 🚨 **Without indexes**, queries on large tables scan every row. With an index on `authorId`, finding a user's posts goes from O(n) to O(log n). Always index foreign keys and frequently filtered columns.

---

## I. Exercises

### Exercise 1 — Blog Backend

Build the database layer for a blog:
1. Define schema: `users`, `posts`, `comments`
2. Run migrations
3. Create Hono routes that perform real CRUD with the database
4. Implement `GET /api/posts` with pagination
5. Implement `GET /api/posts/:id` with author and comments included

### Exercise 2 — Data Integrity

Create a route `POST /api/users/:id/posts` that:
1. Validates the user exists (404 if not)
2. Creates a post in a transaction
3. Returns the post with the author included

---

## Summary

- ✅ PostgreSQL — the production database
- ✅ Drizzle ORM — type-safe schema, queries, migrations
- ✅ CRUD with `select`, `insert`, `update`, `delete`
- ✅ Relational queries with `db.query.*.findMany()`
- ✅ Transactions for atomic operations
- ✅ Migrations — never modify production DBs manually
- ✅ Indexing — essential for query performance

---

## Navigation

← Previous: [Module 07 — Error Handling](./module-07.md)
→ Next: [Module 09 — Authentication](./module-09.md)
