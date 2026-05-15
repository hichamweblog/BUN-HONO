# Module 18 — Capstone Project

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~8–12 hours
> **Project:** Full SaaS Backend

---

## Navigation

← Previous: [Module 17 — Realtime APIs](./module-17.md)
→ Return to: [README / Course Roadmap](../README.md)

---

## 🎯 The Challenge

Build a **production-grade Task Management SaaS backend** from scratch, applying EVERYTHING from Modules 00–17.

This is not a guided tutorial. This is your **engineering challenge**.

You will design, architect, implement, test, document, and deploy a complete backend system.

---

## A. Project Requirements

### The Product

A multi-tenant task management API where:
- Organizations can sign up and invite members
- Members create projects and tasks within their organization
- Tasks have statuses, priorities, assignees, due dates, and comments
- Admins manage organization settings and members
- Real-time updates when tasks change

### Functional Requirements

```
Authentication:
  POST   /auth/signup          → create account + organization
  POST   /auth/login           → get JWT tokens
  POST   /auth/refresh         → refresh access token

Organizations:
  GET    /api/v1/orgs/:orgId              → get org details
  PATCH  /api/v1/orgs/:orgId              → update org (admin)
  POST   /api/v1/orgs/:orgId/invite       → invite member (admin)
  GET    /api/v1/orgs/:orgId/members      → list members

Projects:
  GET    /api/v1/projects                 → list projects (org-scoped)
  POST   /api/v1/projects                 → create project
  GET    /api/v1/projects/:id             → get project
  PATCH  /api/v1/projects/:id             → update project
  DELETE /api/v1/projects/:id             → archive project (admin/owner)

Tasks:
  GET    /api/v1/projects/:id/tasks       → list tasks (filterable, paginated)
  POST   /api/v1/projects/:id/tasks       → create task
  GET    /api/v1/tasks/:id                → get task with comments
  PATCH  /api/v1/tasks/:id                → update task
  DELETE /api/v1/tasks/:id                → delete task
  POST   /api/v1/tasks/:id/comments       → add comment

Realtime:
  GET    /api/v1/events                   → SSE stream of org activity

Health:
  GET    /health                          → system health
```

### Non-Functional Requirements

```
Architecture:
  ✅ Three-layer architecture (routes → services → repositories)
  ✅ Modular route files
  ✅ Validated environment config
  ✅ Consistent error handling with custom error classes

Security:
  ✅ JWT authentication with refresh tokens
  ✅ RBAC (owner, admin, member roles)
  ✅ Resource ownership enforcement
  ✅ CORS, rate limiting, secure headers
  ✅ Input validation on every endpoint

Data:
  ✅ PostgreSQL with Drizzle ORM
  ✅ Proper schema with relations and indexes
  ✅ Migrations
  ✅ Transactions where needed

Quality:
  ✅ At least 15 tests (unit + integration)
  ✅ OpenAPI documentation with Swagger UI
  ✅ Consistent response envelope

Deployment:
  ✅ Dockerfile (multi-stage)
  ✅ docker-compose.yml (app + Postgres)
  ✅ Health check endpoint
  ✅ Graceful shutdown
```

---

## B. Database Schema Design

Design your schema before writing code. Here's a starting point — modify and extend it:

```
organizations
  id, name, slug, created_at, updated_at

users
  id, name, email, password_hash, created_at, updated_at

org_members (join table)
  id, org_id → organizations, user_id → users,
  role (owner | admin | member), joined_at

projects
  id, name, description, org_id → organizations,
  created_by → users, status (active | archived),
  created_at, updated_at

tasks
  id, title, description, status (todo | in_progress | done),
  priority (low | medium | high | urgent),
  project_id → projects, assignee_id → users (nullable),
  created_by → users, due_date (nullable),
  created_at, updated_at

comments
  id, content, task_id → tasks, author_id → users,
  created_at
```

### Schema Questions to Answer

1. What indexes do you need?
2. What happens when a user is deleted — cascade or soft delete?
3. How do you enforce that a task's assignee belongs to the same org?
4. What columns should have UNIQUE constraints?

---

## C. Project Structure

```
capstone/
├── src/
│   ├── index.ts
│   ├── app.ts
│   ├── config/
│   │   └── env.ts
│   ├── db/
│   │   ├── index.ts
│   │   ├── schema.ts
│   │   └── migrations/
│   ├── routes/
│   │   ├── auth.routes.ts
│   │   ├── org.routes.ts
│   │   ├── project.routes.ts
│   │   ├── task.routes.ts
│   │   └── health.routes.ts
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── org.service.ts
│   │   ├── project.service.ts
│   │   └── task.service.ts
│   ├── repositories/
│   │   ├── user.repository.ts
│   │   ├── org.repository.ts
│   │   ├── project.repository.ts
│   │   └── task.repository.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── guards.ts
│   │   ├── org-context.ts
│   │   └── error-handler.ts
│   ├── schemas/
│   │   ├── auth.schema.ts
│   │   ├── org.schema.ts
│   │   ├── project.schema.ts
│   │   ├── task.schema.ts
│   │   └── common.schema.ts
│   ├── errors/
│   │   └── app-error.ts
│   ├── lib/
│   │   ├── jwt.ts
│   │   └── response.ts
│   └── types/
│       └── index.ts
├── tests/
│   ├── auth.test.ts
│   ├── tasks.test.ts
│   └── helpers.ts
├── Dockerfile
├── docker-compose.yml
├── drizzle.config.ts
├── package.json
├── tsconfig.json
├── .env.example
└── .gitignore
```

---

## D. Implementation Guide

### Phase 1 — Foundation (2–3 hours)

1. Initialize project with `bun init`
2. Set up folder structure
3. Create database schema and run migrations
4. Set up environment validation
5. Create global error handler
6. Implement health check endpoint
7. Verify with `docker compose up`

### Phase 2 — Auth (2–3 hours)

1. Implement signup (create user + organization)
2. Implement login with JWT
3. Create auth middleware
4. Create role-based guards
5. Write auth tests

### Phase 3 — Core CRUD (3–4 hours)

1. Organization routes (read, update, invite)
2. Project routes (full CRUD)
3. Task routes (full CRUD with filtering and pagination)
4. Comment routes
5. Ensure proper authorization on every endpoint

### Phase 4 — Polish (2–3 hours)

1. Add OpenAPI documentation
2. Add SSE event stream
3. Add CORS, rate limiting, secure headers
4. Write integration tests (at least 15)
5. Create Dockerfile and docker-compose.yml
6. Test deployment with `docker compose up`

---

## E. Evaluation Criteria

Score yourself honestly:

| Criteria | Points | Your Score |
|----------|--------|------------|
| All endpoints work correctly | /20 | |
| Proper HTTP methods and status codes | /10 | |
| Input validation on every endpoint | /10 | |
| Consistent error responses | /10 | |
| JWT authentication works | /10 | |
| RBAC authorization works | /10 | |
| Database schema is well-designed | /5 | |
| Three-layer architecture followed | /5 | |
| At least 15 passing tests | /5 | |
| OpenAPI docs available | /5 | |
| Docker deployment works | /5 | |
| Code is clean and organized | /5 | |
| **Total** | **/100** | |

```
90-100: Production-ready engineer — you can build real systems
70-89:  Solid foundations — review weak areas, try again
50-69:  Re-study relevant modules, then retry
<50:    Go back to Module 05 and work forward again
```

---

## F. Stretch Goals

If you finish early and want more challenge:

- **Search**: Full-text search on task titles and descriptions
- **File Attachments**: Upload files to tasks using `Bun.file()`
- **Activity Log**: Record all changes (who changed what, when)
- **Email Notifications**: Send emails when assigned to a task
- **API Versioning**: Implement v1 and v2 simultaneously
- **GraphQL**: Add a GraphQL endpoint alongside REST
- **Webhooks**: Allow organizations to register webhook URLs
- **Multi-tenancy**: Complete data isolation between organizations

---

## G. After Completing This Project

You now have:

1. **A portfolio piece** — a production-grade backend you can show to employers
2. **Practical experience** — you've solved real engineering problems
3. **An architecture template** — reuse this structure for future projects
4. **Debugging skills** — you've found and fixed real bugs
5. **Production habits** — validation, error handling, testing, deployment

### What to Learn Next

```
Immediate next steps:
  → Deploy to a real cloud provider (Railway, Fly.io, AWS)
  → Add monitoring (Grafana, Datadog)
  → Learn Redis for caching and rate limiting at scale
  → Learn message queues (BullMQ, RabbitMQ)

Advanced topics:
  → Microservices architecture
  → Event-driven architecture
  → GraphQL with Hono
  → gRPC
  → Kubernetes
```

---

## 🎉 Congratulations

You've completed the **Hono + Bun Production Backend Engineering Course**.

You started by learning what HTTP is. You're ending by building a production-grade SaaS backend with authentication, authorization, database integration, testing, documentation, and deployment.

That's not a tutorial completion — that's engineering growth.

```
Module 00: "What is HTTP?"
Module 18: "I built a production SaaS backend."
```

> **Final Mentor Note**: The best engineers aren't the ones who memorize APIs.
> They're the ones who understand systems, make good tradeoffs, and can debug
> anything. You've built that foundation. Now go build something real.

---

## Navigation

← Previous: [Module 17 — Realtime APIs](./module-17.md)
→ Return to: [README / Course Roadmap](../README.md)
