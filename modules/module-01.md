# Module 01 — Bun Fundamentals

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Foundational
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [Module 00 — Backend & HTTP Fundamentals](./module-00.md)
→ Next: [Module 02 — Hono Fundamentals](./module-02.md)

---

## What You'll Learn

- What Bun is and why it exists
- Bun vs Node.js vs Deno — honest tradeoffs
- Bun as a runtime, package manager, bundler, and test runner
- Bun's native TypeScript support
- Bun APIs: file I/O, environment variables, hashing, and more
- How Bun serves HTTP (the `fetch` convention)
- Setting up your development environment

---

## A. Mental Model — What IS Bun?

### The All-in-One Toolkit Analogy

Think of traditional Node.js development like a construction site where you bring separate tools from different vendors:

```
Node.js Development:
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│  Node.js   │  │    npm     │  │   esbuild  │  │    Jest    │
│  (Runtime) │  │  (Packages)│  │  (Bundler) │  │  (Testing) │
└────────────┘  └────────────┘  └────────────┘  └────────────┘
       +               +              +               +
┌────────────┐  ┌────────────┐  ┌────────────┐
│    tsx     │  │   dotenv   │  │  nodemon   │
│ (TS exec)  │  │  (env vars)│  │ (hot reload)│
└────────────┘  └────────────┘  └────────────┘

Bun Development:
┌──────────────────────────────────────────────────────────────┐
│                          Bun                                  │
│  Runtime + Package Manager + Bundler + Test Runner            │
│  + Native TypeScript + Env Vars + Hot Reload + File I/O       │
└──────────────────────────────────────────────────────────────┘
```

> 💡 Bun replaces 6+ tools with a single binary. That's not just convenience — it's fewer dependency conflicts, faster startup, and a unified developer experience.

---

## B. Why Bun Exists — The Motivation

Bun was created by Jarred Sumner to solve real pain points with Node.js:

| Pain Point | Node.js | Bun |
|-----------|---------|-----|
| TypeScript | Needs `tsx`, `ts-node`, or compile step | Native — just run `.ts` files directly |
| Package install speed | npm/yarn: often 30+ seconds | `bun install`: often under 1 second |
| Startup time | ~40ms minimum | ~6ms (uses JavaScriptCore, not V8) |
| Hot reload | Needs `nodemon` or `--watch` flag | Built-in `--hot` and `--watch` |
| Test runner | Needs Jest/Vitest (separate install) | Built-in `bun test` |
| .env files | Needs `dotenv` package | Built-in — reads `.env` automatically |
| File I/O | `fs` module (callback-based legacy API) | `Bun.file()` — modern, fast API |

### Honest Tradeoffs

Bun is excellent, but you should understand the tradeoffs:

```
✅ Bun Advantages:
• Much faster startup and install times
• Native TypeScript — no compile step
• All-in-one tooling
• Web Standard APIs (fetch, Request, Response)
• Great DX (developer experience)

⚠️ Bun Considerations:
• Younger ecosystem — some edge cases may exist
• Some Node.js packages with native C++ addons may not work
• Smaller community than Node.js (but growing fast)
• Not all Node.js APIs are 100% compatible (most are)
• Production track record is shorter than Node.js
```

> 🧠 **Engineering mindset**: Always understand the tradeoffs of your tools. No technology is universally better — only better *for specific contexts*. Bun is excellent for new projects, APIs, and backend services.

---

## C. Installing and Verifying Bun

```bash
# Install Bun (Linux/macOS/WSL)
curl -fsSL https://bun.sh/install | bash

# Verify installation
bun --version

# See all available commands
bun --help
```

After installation, Bun gives you these commands:

| Command | Purpose |
|---------|---------|
| `bun run <file>` | Execute a TypeScript/JavaScript file |
| `bun install` | Install dependencies (reads `package.json`) |
| `bun add <pkg>` | Add a dependency |
| `bun remove <pkg>` | Remove a dependency |
| `bun test` | Run tests |
| `bun init` | Initialize a new project |
| `bun build` | Bundle your project |
| `bunx <pkg>` | Run a package without installing (like `npx`) |

---

## D. Your First Bun Project — Hands-On

Let's create a project from scratch to understand how Bun works:

```bash
# Create project directory
mkdir hello-bun && cd hello-bun

# Initialize (creates package.json and tsconfig.json)
bun init
```

`bun init` creates this structure:

```
hello-bun/
├── index.ts         ← entry point (TypeScript by default!)
├── package.json     ← project metadata and scripts
├── tsconfig.json    ← TypeScript configuration
├── README.md
└── .gitignore
```

> 💡 Notice: `index.ts`, not `index.js`. Bun uses TypeScript by default. No compilation step needed.

### Your First TypeScript File

```typescript
// index.ts
const greeting: string = "Hello from Bun!"
const version: string = Bun.version

console.log(greeting)
console.log(`Bun version: ${version}`)
console.log(`Process ID: ${process.pid}`)
```

Run it:

```bash
bun run index.ts
# Output:
# Hello from Bun!
# Bun version: 1.x.x
# Process ID: 12345
```

**Line-by-line breakdown:**
- `const greeting: string` — TypeScript type annotation. Bun runs this directly — no `tsc` compilation.
- `Bun.version` — a global Bun API. `Bun` is a global object with utilities.
- `process.pid` — Node.js compatibility. Bun supports most `process` APIs.

---

## E. package.json and Scripts

The `package.json` is your project's configuration file:

```json
{
  "name": "hello-bun",
  "version": "1.0.0",
  "scripts": {
    "dev": "bun run --hot src/index.ts",
    "start": "bun run src/index.ts",
    "test": "bun test"
  },
  "dependencies": {},
  "devDependencies": {
    "@types/bun": "latest"
  }
}
```

**Key scripts explained:**

- **`dev`**: Uses `--hot` for hot module reloading (preserves state, reloads changed modules)
- **`start`**: Production run (no hot reload)
- **`test`**: Runs Bun's built-in test runner

```bash
# Run scripts
bun run dev     # or just: bun dev
bun run start
bun test
```

### `--hot` vs `--watch`

```
--hot    → Hot module reloading. Replaces changed modules WITHOUT
           restarting the process. State is preserved.
           Best for: development servers

--watch  → File watcher. Restarts the ENTIRE process when files change.
           State is reset.
           Best for: scripts, CLIs, one-off tasks
```

> 🚨 **Production Rule**: Never use `--hot` or `--watch` in production. They add overhead and are designed for development only.

---

## F. Bun as a Package Manager

Bun's package manager is a drop-in replacement for npm/yarn:

```bash
# Install all dependencies from package.json
bun install

# Add a dependency
bun add hono

# Add a dev dependency
bun add -d @types/bun

# Remove a dependency
bun remove some-package

# Install a specific version
bun add hono@4.0.0
```

### Why Bun Install Is So Fast

```
npm install:
1. Resolve dependencies      → network calls
2. Download packages          → one at a time (mostly)
3. Extract to node_modules    → disk I/O
4. Run postinstall scripts    → sequential
Total: 10-60 seconds (typical)

bun install:
1. Resolve + download         → parallel, cached globally
2. Hard-link from global cache → near-instant (no copying)
3. Run postinstall scripts    → parallel
Total: 0.5-3 seconds (typical)
```

Bun uses a **global cache** and **hard links** instead of copying files into `node_modules` each time. If you've installed a package once, it's essentially instant on subsequent installs.

### bun.lockb

Bun generates `bun.lockb` (binary lockfile) instead of `package-lock.json`. It's faster to read/write but not human-readable.

```bash
# If you need a readable lockfile (e.g., for auditing)
bun install --yarn  # generates yarn.lock instead
```

> ✅ **Always commit your lockfile** (`bun.lockb`) to version control. It ensures deterministic installs across machines.

---

## G. Bun APIs — The Built-in Toolkit

Bun provides global APIs on the `Bun` object. Here are the ones you'll use most:

### File I/O — `Bun.file()` and `Bun.write()`

```typescript
// Reading a file
const file = Bun.file("./data.json")
const text = await file.text()        // as string
const json = await file.json()        // parsed JSON
const bytes = await file.arrayBuffer() // raw bytes

console.log(file.size)  // file size in bytes
console.log(file.type)  // MIME type (e.g., "application/json")

// Writing a file
await Bun.write("./output.txt", "Hello, file!")
await Bun.write("./data.json", JSON.stringify({ name: "Alice" }))

// Writing to stdout
await Bun.write(Bun.stdout, "Printed to terminal\n")
```

**Why this is better than Node.js `fs`:**
- No callback hell — always returns Promises
- Lazy loading — file content isn't read until you call `.text()` or `.json()`
- Type-safe — `Bun.file()` returns a `BunFile` object with methods

### Environment Variables

```typescript
// Bun reads .env files automatically — no dotenv needed!

// .env file:
// DATABASE_URL=postgres://localhost:5432/mydb
// API_KEY=secret123
// PORT=3000

// Access in code:
const dbUrl = Bun.env.DATABASE_URL   // or process.env.DATABASE_URL
const port = Bun.env.PORT ?? "3000"

// Type-safe check
if (!Bun.env.DATABASE_URL) {
  console.error("Missing DATABASE_URL")
  process.exit(1)
}
```

**Env file loading order** (Bun loads automatically):
1. `.env.local` — local overrides (gitignored)
2. `.env.development` / `.env.production` — mode-specific
3. `.env` — default values

> 🚨 **Security Rule**: Never commit `.env` files with real secrets. Add `.env` to `.gitignore`. Commit `.env.example` with placeholder values instead.

### Password Hashing

```typescript
// Hash a password (uses Argon2id by default — very secure)
const hash = await Bun.password.hash("my-password")
// "$argon2id$v=19$m=65536,t=2,p=1$..."

// Verify a password
const isValid = await Bun.password.verify("my-password", hash)
// true

const isWrong = await Bun.password.verify("wrong-password", hash)
// false
```

> 💡 Bun uses **Argon2id** by default, which is the recommended algorithm for password hashing (winner of the Password Hashing Competition). No need for `bcrypt` package.

### Hashing (Non-Password)

```typescript
// For data integrity, checksums, etc.
const hasher = new Bun.CryptoHasher("sha256")
hasher.update("hello world")
const digest = hasher.digest("hex")
// "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9"

// Quick one-liner
const quick = Bun.hash("hello world")  // fast non-crypto hash (Wyhash)
```

### Sleep and Timing

```typescript
// Sleep (async, non-blocking)
await Bun.sleep(1000) // sleep 1 second
await Bun.sleep(50)   // sleep 50ms

// Precise timing
const start = Bun.nanoseconds()
// ... do work ...
const elapsed = Bun.nanoseconds() - start
console.log(`Took ${elapsed / 1_000_000}ms`)
```

---

## H. Bun's HTTP Server — The `fetch` Convention

This is the key pattern you need to understand before learning Hono.

Bun serves HTTP using the **Web Standard `fetch` API**:

```typescript
// src/server.ts — A raw Bun HTTP server (no framework)
const server = Bun.serve({
  port: 3000,
  fetch(request: Request): Response {
    const url = new URL(request.url)

    if (url.pathname === "/") {
      return new Response("Hello World!")
    }

    if (url.pathname === "/json") {
      return Response.json({ message: "Hello JSON!" })
    }

    return new Response("Not Found", { status: 404 })
  },
})

console.log(`Server running at http://localhost:${server.port}`)
```

**The critical insight:**

```
Every Bun HTTP server is a function:

   (Request) → Response

That's it. You receive a Web Standard Request object,
and you return a Web Standard Response object.

This is EXACTLY what Hono wraps — with routing,
middleware, and helpers to make it ergonomic.
```

### Request and Response — Web Standard APIs

These are **not** Bun-specific. They're Web Standards that work in browsers, Deno, Cloudflare Workers, and Bun:

```typescript
// Request — what you RECEIVE
const req = new Request("https://example.com/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Alice" }),
})

req.method      // "POST"
req.url         // "https://example.com/api/users"
req.headers     // Headers object
await req.json() // { name: "Alice" }

// Response — what you RETURN
const res = new Response(JSON.stringify({ id: 1 }), {
  status: 201,
  headers: { "Content-Type": "application/json" },
})

// Shorthand for JSON responses
const res2 = Response.json({ id: 1 }, { status: 201 })
```

> 🧠 **Why this matters**: Hono is built on top of these exact objects. When you learn `c.req` and `c.json()` in Hono, they're thin wrappers around `Request` and `Response`. Understanding the raw objects makes Hono click.

---

## I. The `export default` Pattern

Bun has a special convention for HTTP servers:

```typescript
// Method 1: Bun.serve() — explicit
const server = Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response("Hello")
  },
})

// Method 2: export default — implicit (preferred with Hono)
export default {
  port: 3000,
  fetch(req: Request) {
    return new Response("Hello")
  },
}
```

Both do the same thing. `export default` is the convention Hono uses:

```typescript
// This is what a Hono app looks like in Bun:
import { Hono } from "hono"

const app = new Hono()
app.get("/", (c) => c.text("Hello Hono!"))

export default app  // Hono's app object has a .fetch() method
```

Bun sees the `export default` with a `fetch` method and automatically starts an HTTP server. Clean and declarative.

---

## J. Bun's Built-in Test Runner

```typescript
// math.ts
export function add(a: number, b: number): number {
  return a + b
}

export function divide(a: number, b: number): number {
  if (b === 0) throw new Error("Cannot divide by zero")
  return a / b
}
```

```typescript
// math.test.ts
import { describe, expect, it } from "bun:test"
import { add, divide } from "./math"

describe("add", () => {
  it("adds two positive numbers", () => {
    expect(add(2, 3)).toBe(5)
  })

  it("handles negative numbers", () => {
    expect(add(-1, 1)).toBe(0)
  })
})

describe("divide", () => {
  it("divides correctly", () => {
    expect(divide(10, 2)).toBe(5)
  })

  it("throws on division by zero", () => {
    expect(() => divide(10, 0)).toThrow("Cannot divide by zero")
  })
})
```

```bash
# Run all tests
bun test

# Run specific test file
bun test math.test.ts

# Watch mode (re-runs on file changes)
bun test --watch
```

> 💡 The test API (`describe`, `it`, `expect`) is intentionally similar to Jest. If you've used Jest, you already know Bun's test runner.

---

## K. TypeScript in Bun — Zero Config

Bun runs TypeScript natively. No `tsc`, no build step, no configuration needed.

```typescript
// types.ts — TypeScript works out of the box
interface User {
  id: number
  name: string
  email: string
  role: "admin" | "user"
}

function greetUser(user: User): string {
  return `Hello, ${user.name}! You are a ${user.role}.`
}

const alice: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  role: "admin",
}

console.log(greetUser(alice))
```

```bash
bun run types.ts
# Hello, Alice! You are a admin.
```

**Important distinction:**

```
Bun TRANSPILES TypeScript — it strips types and runs the JS.
Bun does NOT type-check at runtime.

This means:
✅ Fast execution (no type checking overhead)
⚠️ Type errors are NOT caught at runtime

To catch type errors, use your editor (VS Code) or run:
  bunx tsc --noEmit    (type-check without compiling)
```

> 🚨 **Production Tip**: Add `bunx tsc --noEmit` to your CI pipeline. Bun won't catch type errors — your CI should.

---

## L. Common Mistakes & Debugging

### Mistake 1: Forgetting That Bun Doesn't Type-Check

```typescript
const name: number = "Alice"  // TypeScript ERROR — wrong type
// Bun will run this WITHOUT error!
// Only your editor or `tsc` will catch it.
```

**Fix:** Use VS Code with the TypeScript extension. Run `tsc --noEmit` in CI.

### Mistake 2: Using Node.js-Only APIs

```typescript
// ❌ Some Node.js APIs may not be fully supported
import { createServer } from "node:http"
// This works in Bun but using Bun.serve() is idiomatic

// ✅ Use Bun's native APIs when available
Bun.serve({ fetch: (req) => new Response("Hello") })
```

### Mistake 3: Blocking the Event Loop

```typescript
// ❌ CPU-intensive synchronous work blocks everything
function fibonacci(n: number): number {
  if (n <= 1) return n
  return fibonacci(n - 1) + fibonacci(n - 2)
}
fibonacci(45)  // blocks for seconds — no other request can be served!

// ✅ For CPU-heavy work, consider:
// 1. Breaking it into chunks
// 2. Using Workers (Bun supports Web Workers)
// 3. Offloading to a queue/background job
```

### Mistake 4: Not Handling Async Properly

```typescript
// ❌ Forgetting await — gets a Promise object, not the value
const file = Bun.file("data.json")
const data = file.json()  // This is a Promise<any>, not the data!
console.log(data)  // Promise { <pending> }

// ✅ Always await async operations
const data2 = await file.json()  // Now it's the actual data
console.log(data2)  // { name: "Alice" }
```

---

## M. 🧠 Comprehension Check

1. What **three roles** does Bun serve beyond being a runtime?
2. Why is `bun install` faster than `npm install`?
3. What does `--hot` do vs `--watch`? When would you use each?
4. Bun runs TypeScript natively — but what does it NOT do with TypeScript?
5. What is the `fetch` convention? Write a minimal Bun HTTP server from memory.
6. Why should you never commit `.env` files?

---

## N. Exercises

### Exercise 1 — File-Based Data Store

Create a script that:
1. Reads a JSON file (`users.json`) using `Bun.file()`
2. Adds a new user to the array
3. Writes the updated array back using `Bun.write()`
4. Prints the total number of users

Start with this `users.json`:
```json
[
  { "id": 1, "name": "Alice", "email": "alice@example.com" }
]
```

### Exercise 2 — Environment Validation

Create a script that:
1. Reads `PORT`, `DATABASE_URL`, and `API_KEY` from environment
2. Validates that all three exist
3. Prints a clear error for each missing variable
4. Exits with code 1 if any are missing

### Exercise 3 — Raw HTTP Server

Build a Bun HTTP server (no Hono) that handles:
- `GET /` → returns `{ "status": "ok" }`
- `GET /time` → returns `{ "time": "<current ISO timestamp>" }`
- `POST /echo` → returns whatever JSON body was sent
- Everything else → returns 404

---

## O. Glossary

| Term | Definition |
|------|------------|
| **Bun.serve()** | Starts an HTTP server using the fetch convention |
| **Hard link** | A filesystem link that points to the same data on disk — no copying needed |
| **Hot module reloading** | Replacing changed modules without restarting the process |
| **JavaScriptCore (JSC)** | The JavaScript engine used by Safari and Bun (alternative to V8) |
| **Lockfile** | Records exact dependency versions for deterministic installs |
| **Transpile** | Convert TypeScript to JavaScript by stripping types (no type checking) |
| **Web Standard API** | Browser-compatible APIs (Request, Response, fetch, URL, Headers) |

---

## Summary

You now understand:
- ✅ What Bun is and the problems it solves
- ✅ Bun vs Node.js — advantages and honest tradeoffs
- ✅ Bun's four roles: runtime, package manager, bundler, test runner
- ✅ Native TypeScript execution (transpile, not type-check)
- ✅ Core Bun APIs: file I/O, env vars, hashing, password hashing
- ✅ The `fetch` convention — `(Request) → Response`
- ✅ The `export default` pattern for HTTP servers
- ✅ Bun's test runner with `bun:test`

---

## Navigation

← Previous: [Module 00 — Backend & HTTP Fundamentals](./module-00.md)
→ Next: [Module 02 — Hono Fundamentals](./module-02.md)

---

> **Mentor Note**: Run every code example yourself. Change things. Break things.
> The fastest way to learn a runtime is to *use* it, not read about it.
> Next up: Hono — the framework that makes Bun's HTTP server a joy to use.
