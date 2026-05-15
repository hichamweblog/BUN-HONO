# Module 17 — Realtime APIs

> **Course:** Hono + Bun Production Backend Engineering
> **Level:** Advanced
> **Duration:** ~2–3 hours

---

## Navigation

← Previous: [Module 16 — Deployment](./module-16.md)
→ Next: [Module 18 — Capstone Project](./module-18.md)

---

## What You'll Learn

- When to use realtime vs request-response
- Server-Sent Events (SSE)
- WebSockets with Hono + Bun
- Streaming responses
- Connection management and cleanup
- Realtime architecture patterns

---

## A. Mental Model — Mail vs Phone vs Walkie-Talkie

```
Request-Response (HTTP):
  📬 Mail — you send a letter, wait for a reply
  • Client initiates every interaction
  • One question, one answer
  • Use for: CRUD, forms, data fetches

Server-Sent Events (SSE):
  📻 Radio — server broadcasts, client listens
  • Server pushes updates to client
  • One-way: server → client
  • Use for: live feeds, notifications, dashboards

WebSockets:
  📞 Phone call — both sides talk freely
  • Full-duplex: both client and server send anytime
  • Persistent connection
  • Use for: chat, gaming, collaborative editing
```

---

## B. Server-Sent Events (SSE)

SSE is the simplest realtime protocol. The server pushes text events over a long-lived HTTP connection.

```typescript
import { Hono } from "hono"
import { streamSSE } from "hono/streaming"

const app = new Hono()

// SSE endpoint — client connects and receives events
app.get("/api/events", (c) => {
  return streamSSE(c, async (stream) => {
    let id = 0

    while (true) {
      // Send an event every 2 seconds
      await stream.writeSSE({
        id: String(id++),
        event: "update",
        data: JSON.stringify({
          timestamp: new Date().toISOString(),
          message: `Event #${id}`,
        }),
      })

      // Wait 2 seconds
      await stream.sleep(2000)
    }
  })
})
```

**Client-side (browser):**

```javascript
const eventSource = new EventSource("http://localhost:3000/api/events")

eventSource.addEventListener("update", (event) => {
  const data = JSON.parse(event.data)
  console.log("Received:", data)
})

eventSource.onerror = () => {
  console.log("Connection lost, reconnecting...")
  // EventSource auto-reconnects by default
}
```

### When to Use SSE

```
✅ Use SSE when:
• Server pushes data to client (one-way)
• Live dashboards, stock tickers, notification feeds
• You want auto-reconnection (built into EventSource API)
• You need simple setup (just HTTP — works through proxies)

❌ Don't use SSE when:
• Client needs to send data back (use WebSockets)
• Binary data (SSE is text-only)
• High-frequency bidirectional communication
```

---

## C. WebSockets

For full-duplex communication, Hono supports WebSockets via Bun's native API:

```typescript
import { Hono } from "hono"
import { createBunWebSocket } from "hono/bun"

const { upgradeWebSocket, websocket } = createBunWebSocket()

const app = new Hono()

// Connected clients
const clients = new Set<WebSocket>()

app.get(
  "/ws",
  upgradeWebSocket((c) => ({
    onOpen(_event, ws) {
      console.log("Client connected")
      clients.add(ws.raw as WebSocket)
    },

    onMessage(event, ws) {
      const message = event.data.toString()
      console.log("Received:", message)

      // Broadcast to all connected clients
      for (const client of clients) {
        if (client !== ws.raw && client.readyState === WebSocket.OPEN) {
          client.send(JSON.stringify({
            type: "message",
            data: message,
            timestamp: new Date().toISOString(),
          }))
        }
      }
    },

    onClose(_event, ws) {
      console.log("Client disconnected")
      clients.delete(ws.raw as WebSocket)
    },

    onError(event, ws) {
      console.error("WebSocket error:", event)
      clients.delete(ws.raw as WebSocket)
    },
  }))
)

// Export with websocket handler for Bun
export default {
  port: 3000,
  fetch: app.fetch,
  websocket,
}
```

**Client-side:**

```javascript
const ws = new WebSocket("ws://localhost:3000/ws")

ws.onopen = () => {
  console.log("Connected")
  ws.send("Hello from client!")
}

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)
  console.log("Received:", data)
}

ws.onclose = () => {
  console.log("Disconnected")
  // Implement reconnection logic
}
```

---

## D. Streaming Responses

For large or progressive data, use Hono's streaming helpers:

```typescript
import { stream, streamText } from "hono/streaming"

// Stream text progressively
app.get("/api/stream", (c) => {
  return streamText(c, async (stream) => {
    for (let i = 0; i < 10; i++) {
      await stream.write(`Chunk ${i + 1}\n`)
      await stream.sleep(500)
    }
  })
})

// Stream binary data
app.get("/api/download", (c) => {
  return stream(c, async (stream) => {
    const file = Bun.file("./large-file.csv")
    const reader = file.stream().getReader()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      await stream.write(value)
    }
  })
})
```

---

## E. Connection Management

### Heartbeats (Keep Connections Alive)

```typescript
// Send periodic pings to detect dead connections
onOpen(_event, ws) {
  const interval = setInterval(() => {
    if (ws.raw.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: "ping" }))
    } else {
      clearInterval(interval)
    }
  }, 30_000)  // every 30 seconds
}
```

### Cleanup on Disconnect

```typescript
onClose(_event, ws) {
  // Remove from all rooms/channels
  clients.delete(ws.raw as WebSocket)
  // Clean up any associated resources
  // Remove from chat rooms, unsubscribe from topics, etc.
}
```

---

## F. Architecture Patterns

### Channel/Room Pattern (Chat)

```typescript
const rooms = new Map<string, Set<WebSocket>>()

function joinRoom(roomId: string, ws: WebSocket) {
  if (!rooms.has(roomId)) rooms.set(roomId, new Set())
  rooms.get(roomId)!.add(ws)
}

function broadcastToRoom(roomId: string, message: string, sender?: WebSocket) {
  const room = rooms.get(roomId)
  if (!room) return

  for (const client of room) {
    if (client !== sender && client.readyState === WebSocket.OPEN) {
      client.send(message)
    }
  }
}
```

### Pub/Sub Pattern (Notifications)

```typescript
const subscriptions = new Map<string, Set<WebSocket>>()

function subscribe(topic: string, ws: WebSocket) {
  if (!subscriptions.has(topic)) subscriptions.set(topic, new Set())
  subscriptions.get(topic)!.add(ws)
}

function publish(topic: string, data: unknown) {
  const subs = subscriptions.get(topic)
  if (!subs) return

  const message = JSON.stringify({ topic, data })
  for (const client of subs) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(message)
    }
  }
}
```

---

## G. When to Use What

| Pattern | Protocol | Direction | Use Case |
|---------|----------|-----------|----------|
| Request-Response | HTTP | Client → Server → Client | CRUD, forms, data fetches |
| Polling | HTTP | Client → Server (repeated) | Simple updates, fallback |
| SSE | HTTP (long-lived) | Server → Client | Notifications, live feeds |
| WebSocket | WS | Bidirectional | Chat, gaming, collaboration |
| Streaming | HTTP (chunked) | Server → Client | Large downloads, AI responses |

---

## H. Exercises

### Exercise 1 — Live Notifications

Build an SSE endpoint that pushes notifications to connected clients when:
- A new post is created (`POST /api/posts`)
- A comment is added (`POST /api/posts/:id/comments`)

### Exercise 2 — Chat Room

Build a WebSocket-based chat system:
- Users connect and specify a username
- Users can join/leave rooms
- Messages are broadcast to everyone in the same room
- Show "user joined" / "user left" events

---

## Summary

- ✅ SSE — simple server-to-client push (auto-reconnect, HTTP-based)
- ✅ WebSockets — full-duplex bidirectional communication
- ✅ Streaming — progressive data delivery
- ✅ Connection management — heartbeats, cleanup, rooms
- ✅ Choose the right protocol for the right use case

---

## Navigation

← Previous: [Module 16 — Deployment](./module-16.md)
→ Next: [Module 18 — Capstone Project](./module-18.md)
