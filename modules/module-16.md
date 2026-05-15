# Module 16 — Deployment

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~3–4 hours

---

## Navigation

← Previous: [Module 15 — OpenAPI](./module-15.md)
→ Next: [Module 17 — Realtime APIs](./module-17.md)

---

## What You'll Learn

- Docker fundamentals for backend engineers
- Dockerizing a Hono + Bun application
- Environment management (dev, staging, production)
- Production configuration best practices
- VPS deployment with Docker Compose
- Reverse proxy with Caddy/Nginx
- CI/CD pipeline basics
- Health checks and graceful shutdown
- Logging and monitoring fundamentals

---

## A. Mental Model — Shipping Containers

```
Without Docker:
  "It works on my machine" → "It doesn't work on the server"
  Why? Different OS, different Bun version, missing env vars,
       different Node modules, different file paths...

With Docker:
  Your app + runtime + dependencies + config = ONE container
  Same container runs identically everywhere:
    your laptop → CI server → staging → production
```

```
┌──────────────────────────────────────────────┐
│  Docker Container                            │
│  ┌──────────┐  ┌────────┐  ┌─────────────┐ │
│  │ Your App │  │  Bun   │  │ Dependencies│ │
│  │ (code)   │  │(runtime)│  │(node_modules)│ │
│  └──────────┘  └────────┘  └─────────────┘ │
│  ┌──────────────────────────────────────┐   │
│  │ Alpine Linux (minimal OS, ~5MB)      │   │
│  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

---

## B. Dockerfile — Step by Step

```dockerfile
# Dockerfile
# ── Stage 1: Install dependencies ──────────────────
FROM oven/bun:1-alpine AS deps

WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile --production

# ── Stage 2: Build (if needed) ─────────────────────
FROM oven/bun:1-alpine AS builder

WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile
COPY . .

# Type-check (catches errors before deployment)
RUN bunx tsc --noEmit

# ── Stage 3: Production image ──────────────────────
FROM oven/bun:1-alpine AS runtime

WORKDIR /app

# Don't run as root in production
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only production dependencies and source
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/src ./src
COPY --from=builder /app/package.json ./

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start the application
CMD ["bun", "run", "src/index.ts"]
```

**Line-by-line explanation:**

- **Multi-stage build**: Three stages keep the final image small. Build tools and dev dependencies aren't in the production image.
- **`--frozen-lockfile`**: Ensures exact versions from `bun.lockb` — no surprise updates.
- **Non-root user**: Security best practice. If your app is compromised, the attacker has limited permissions.
- **`HEALTHCHECK`**: Docker/orchestrators use this to detect unhealthy containers and restart them.

### Build and Run

```bash
# Build the image
docker build -t my-api .

# Run the container
docker run -d \
  --name my-api \
  -p 3000:3000 \
  -e DATABASE_URL="postgres://..." \
  -e JWT_SECRET="your-secret-here" \
  my-api

# Check logs
docker logs -f my-api
```

---

## C. Docker Compose — Full Stack

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DATABASE_URL=postgres://dev:devpass@db:5432/myapp
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d myapp"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

```bash
# Start everything
docker compose up -d

# View logs
docker compose logs -f api

# Stop everything
docker compose down

# Stop and remove volumes (resets database)
docker compose down -v
```

> 💡 **`depends_on` with `condition: service_healthy`** ensures the API doesn't start until PostgreSQL is ready. Without this, your app tries to connect to a DB that's still booting.

---

## D. Environment Management

```
.env                   ← default values (committed, no secrets)
.env.local             ← local overrides (gitignored)
.env.production        ← production values (gitignored or in CI secrets)
.env.example           ← template for other developers (committed)
```

```bash
# .env.example (committed to git)
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://dev:devpass@localhost:5432/myapp
JWT_SECRET=change-me-in-production-use-openssl-rand-hex-32
CORS_ORIGIN=http://localhost:5173
```

> 🚨 **Never commit real secrets.** Use CI/CD secret management (GitHub Secrets, Railway, Fly.io secrets) for production values.

---

## E. Graceful Shutdown

```typescript
// src/index.ts
import app from "./app"

const server = Bun.serve({
  port: Number(Bun.env.PORT) || 3000,
  fetch: app.fetch,
})

console.log(`Server running on port ${server.port}`)

// Handle shutdown signals (Docker sends SIGTERM)
function gracefulShutdown() {
  console.log("Shutting down gracefully...")

  // Stop accepting new connections
  server.stop()

  // Close database connections
  // await db.end()

  console.log("Server stopped.")
  process.exit(0)
}

process.on("SIGTERM", gracefulShutdown)
process.on("SIGINT", gracefulShutdown)
```

> ✅ **Graceful shutdown** lets in-flight requests finish before the server stops. Without it, Docker kills your process mid-request, and users get broken responses.

---

## F. Reverse Proxy

In production, you typically put a reverse proxy in front of your app:

```
Client → Caddy/Nginx (TLS, compression, static files) → Your Bun App
```

### Caddy Example (auto-HTTPS)

```
# Caddyfile
api.myapp.com {
    reverse_proxy localhost:3000
}
```

Caddy automatically provisions and renews HTTPS certificates via Let's Encrypt. No configuration needed.

### Nginx Example

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## G. Health Check Endpoint

Every production service needs a health check:

```typescript
app.get("/health", async (c) => {
  try {
    // Check database connectivity
    await db.execute(sql`SELECT 1`)

    return c.json({
      status: "healthy",
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      version: Bun.env.APP_VERSION ?? "unknown",
    })
  } catch {
    return c.json({
      status: "unhealthy",
      timestamp: new Date().toISOString(),
    }, 503)
  }
})
```

---

## H. Production Checklist

```
✅ Dockerfile with multi-stage build
✅ Non-root user in container
✅ Health check endpoint (/health)
✅ Graceful shutdown on SIGTERM
✅ Environment variables for all config (no hardcoded values)
✅ .env.example committed, real .env files gitignored
✅ Database connection pooling configured
✅ Reverse proxy for TLS termination
✅ Logging to stdout/stderr (Docker captures these)
✅ Type-checking in CI (bunx tsc --noEmit)
✅ Tests pass before deployment
✅ Error monitoring (Sentry, Grafana, etc.)
```

---

## I. Exercises

### Exercise 1 — Dockerize

Create a Dockerfile and docker-compose.yml for the blog API. Include:
- Multi-stage build
- PostgreSQL service
- Health check
- Environment variables

Verify it starts from scratch with `docker compose up`.

### Exercise 2 — CI Pipeline

Write a GitHub Actions workflow (`.github/workflows/ci.yml`) that:
1. Installs dependencies with `bun install`
2. Runs type-check with `bunx tsc --noEmit`
3. Runs tests with `bun test`
4. Builds the Docker image

---

## Summary

- ✅ Docker — consistent deployments everywhere
- ✅ Multi-stage Dockerfiles — small, secure production images
- ✅ Docker Compose — full stack with one command
- ✅ Environment management — secrets out of code
- ✅ Graceful shutdown — don't kill in-flight requests
- ✅ Reverse proxy — TLS, compression, routing
- ✅ Health checks — let infrastructure know your app is alive

---

## Navigation

← Previous: [Module 15 — OpenAPI](./module-15.md)
→ Next: [Module 17 — Realtime APIs](./module-17.md)
