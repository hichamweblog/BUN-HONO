# 🔥 Hono + Bun: Production Backend Engineering Course

> A practical, production-oriented course for building real-world backend systems
> with **Hono**, **Bun**, **TypeScript**, and modern backend architecture.

---

## 🎯 What You'll Be Able to Do After This Course

- Design and build production-grade REST APIs from scratch
- Structure backend projects using layered architecture
- Implement authentication, authorization, and security best practices
- Write validated, type-safe APIs with Zod and TypeScript
- Work with PostgreSQL databases using Drizzle ORM
- Write meaningful tests with Bun's test runner
- Deploy Dockerized backend services to production
- Debug, profile, and optimize backend performance
- Think like a senior backend engineer

---

## 👤 Who This Course Is For

- JavaScript/TypeScript developers moving into backend
- Frontend developers who want to understand the full stack
- Self-taught developers who want production-quality habits
- Anyone who has done tutorials but never built a real API from scratch

**Not required:** Prior backend experience, database knowledge, or DevOps skills — we start from zero.

---

## 🗺️ How to Use This Course

Each module lives in its own markdown file inside the `modules/` directory.
Work through them **in order** — every module builds on the previous one.

Every module follows this structure:

1. 🧠 **Mental Model** — intuitive analogy before any code
2. 📊 **Visual Diagrams** — architecture and flow visualizations
3. 📖 **Deep Technical Explanation** — the *why*, not just the *how*
4. 💻 **Step-by-Step Code** — production-quality, line-by-line
5. ✅ **Best Practices** — what senior engineers do
6. 🐛 **Common Mistakes & Debugging** — what goes wrong and how to fix it
7. 🏋️ **Exercises & Challenges** — active learning, not passive reading

---

## 📘 Conventions Used in This Course

| Symbol | Meaning |
|--------|---------|
| ✅ | Correct approach / best practice |
| ❌ | Anti-pattern / mistake |
| 🚨 | Production danger — pay close attention |
| 💡 | Key insight or senior-engineer tip |
| 🧠 | Conceptual checkpoint — pause and think |
| ⚠️ | Caveat or nuance you need to know |

---

## 📚 Course Modules

| # | Module | What You'll Learn | Est. Time | File |
|---|--------|-------------------|-----------|------|
| 00 | Backend & HTTP Fundamentals | How the web works, HTTP lifecycle, REST, async I/O | 2–3 hrs | [module-00.md](./modules/module-00.md) |
| 01 | Bun Fundamentals | Bun runtime, tooling, APIs, Bun vs Node.js | 2–3 hrs | [module-01.md](./modules/module-01.md) |
| 02 | Hono Fundamentals | Philosophy, Web Standards, project setup, first app | 2–3 hrs | [module-02.md](./modules/module-02.md) |
| 03 | Routing | Path params, query strings, groups, chaining | 2–3 hrs | [module-03.md](./modules/module-03.md) |
| 04 | Request & Response | Context object, headers, body parsing, helpers | 2 hrs | [module-04.md](./modules/module-04.md) |
| 05 | Middleware | Built-in middleware, custom middleware, execution order | 3–4 hrs | [module-05.md](./modules/module-05.md) |
| 06 | Validation | Zod schemas, request validation, type-safe APIs | 2–3 hrs | [module-06.md](./modules/module-06.md) |
| 07 | Error Handling | Global error handler, HTTPException, typed errors | 2 hrs | [module-07.md](./modules/module-07.md) |
| 08 | Databases | PostgreSQL, SQL, Drizzle ORM, migrations, relations | 4–5 hrs | [module-08.md](./modules/module-08.md) |
| 09 | Authentication | JWT, password hashing, cookies, session security | 3–4 hrs | [module-09.md](./modules/module-09.md) |
| 10 | Authorization | RBAC, permission guards, resource ownership | 2–3 hrs | [module-10.md](./modules/module-10.md) |
| 11 | Testing | Bun test runner, unit/integration tests, mocking | 3–4 hrs | [module-11.md](./modules/module-11.md) |
| 12 | Production Architecture | Layered architecture, DI patterns, folder structure | 3–4 hrs | [module-12.md](./modules/module-12.md) |
| 13 | Security | CORS, CSRF, rate limiting, secure headers, XSS | 2–3 hrs | [module-13.md](./modules/module-13.md) |
| 14 | Performance | Caching, streaming, compression, pagination | 2–3 hrs | [module-14.md](./modules/module-14.md) |
| 15 | OpenAPI | Zod-OpenAPI, auto-generated docs, schema-first design | 2 hrs | [module-15.md](./modules/module-15.md) |
| 16 | Deployment | Docker, env management, VPS, reverse proxy | 3–4 hrs | [module-16.md](./modules/module-16.md) |
| 17 | Realtime APIs | WebSockets, Server-Sent Events, streaming responses | 2–3 hrs | [module-17.md](./modules/module-17.md) |
| 18 | Capstone Project | Full SaaS backend from scratch — everything combined | 8–12 hrs | [module-18.md](./modules/module-18.md) |

**Total estimated time:** ~50–65 hours of deep, hands-on learning.

---

## 🛠️ Prerequisites

- **JavaScript**: Variables, functions, arrays, objects, `async`/`await`, destructuring
- **Terminal/CLI**: Can navigate directories, run commands, read output
- **Code editor**: VS Code recommended (with the following extensions)

---

## ⚙️ Setup Requirements

### 1. Install Bun

```bash
# Install Bun (macOS / Linux / WSL)
curl -fsSL https://bun.sh/install | bash

# Verify installation
bun --version   # should print 1.x.x or higher
```

### 2. Editor Setup (Recommended)

Install [VS Code](https://code.visualstudio.com/) with these extensions:
- **ESLint** — linting
- **Error Lens** — inline error display
- **REST Client** — test HTTP requests from VS Code (alternative to Postman)
- **Thunder Client** — GUI REST client inside VS Code

### 3. HTTP Client (Choose One)

You'll need a way to send HTTP requests to your API:
- **curl** — already installed on most Linux/macOS systems
- **httpie** — `brew install httpie` or `bun install -g httpie` (nicer output)
- **Postman** — GUI tool for exploring APIs (free tier)

### 4. Database (Module 08 Onward)

```bash
# We'll set this up when we get to Module 08
# Options: local PostgreSQL, Docker, or Neon (serverless)
```

---

## 🏗️ Projects You'll Build

| Project | Modules | Skills Practiced |
|---------|---------|-----------------|
| Hello API | 02–03 | Hono setup, routing, first endpoints |
| Notes API | 04–07 | CRUD operations, validation, error handling |
| Auth API | 09–10 | JWT authentication, RBAC authorization |
| Blog Backend | 08, 11 | Database relations, testing, pagination |
| Production REST API | 12–15 | Layered architecture, security, OpenAPI docs |
| SaaS Backend (Capstone) | 18 | Everything combined — a production-grade system |

---

## 📖 Reference Documentation

- [Hono — Official Docs](https://hono.dev/docs/)
- [Bun — Official Docs](https://bun.sh/docs)
- [Drizzle ORM — Official Docs](https://orm.drizzle.team/)
- [Zod — Official Docs](https://zod.dev/)
- [MDN — HTTP Reference](https://developer.mozilla.org/en-US/docs/Web/HTTP)

---

> **Start here →** [Module 00 — Backend & HTTP Fundamentals](./modules/module-00.md)
