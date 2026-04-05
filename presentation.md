# Introduction to Server-Sent Events

**One-Way Real-Time Streaming over HTTP**

`EventSource` | `text/event-stream` | `auto-reconnect` | `HTTP/2` | `streaming`

---

## Table of Contents

1. [Agenda](#slide-02--agenda)
2. [The Problem -- Why Server Push?](#slide-03--the-problem--why-server-push)
3. [What Are Server-Sent Events?](#slide-04--what-are-server-sent-events)
4. [SSE vs WebSockets vs Polling](#slide-05--sse-vs-websockets-vs-polling)
5. [The Event Stream Protocol](#slide-06--the-event-stream-protocol)
6. [EventSource Browser API](#slide-07--eventsource-browser-api)
7. [Custom Event Types](#slide-08--custom-event-types)
8. [Last-Event-ID & Reconnection](#slide-09--last-event-id--reconnection)
9. [Node.js SSE Server](#slide-10--nodejs-sse-server)
10. [SSE with Fetch API](#slide-11--sse-with-fetch-api)
11. [Real-World: Live Notifications](#slide-12--real-world-live-notifications)
12. [Real-World: Live Dashboards](#slide-13--real-world-live-dashboards)
13. [Real-World: AI/LLM Streaming](#slide-14--real-world-aillm-streaming)
14. [SSE over HTTP/2](#slide-15--sse-over-http2)
15. [Authentication & CORS](#slide-16--authentication--cors)
16. [Error Handling & Retry Logic](#slide-17--error-handling--retry-logic)
17. [Scaling SSE](#slide-18--scaling-sse)
18. [Security Considerations](#slide-19--security-considerations)
19. [Summary & Next Steps](#slide-20--summary--next-steps)

---

## Slide 02 -- Agenda

### Foundations
- The problem SSE solves
- What are Server-Sent Events?
- SSE vs WebSockets vs Polling

### Protocol Deep-Dive
- The `text/event-stream` format
- EventSource browser API
- Custom events & reconnection

### Building with SSE
- Node.js server implementation
- Fetch API & ReadableStream
- Live notifications, dashboards, AI streaming

### Production Concerns
- HTTP/2 multiplexing
- Auth, CORS, error handling
- Scaling, security, best practices

**Flow:** Foundations --> Protocol --> Building --> Production

---

## Slide 03 -- The Problem -- Why Server Push?

### Short Polling

Client sends repeated requests at fixed intervals. Wastes bandwidth when nothing has changed. High latency between actual event and client awareness.

```javascript
// Wasteful: polls every 3 seconds
setInterval(async () => {
  const res = await fetch('/api/updates');
  const data = await res.json();
  if (data.length) render(data);
}, 3000);
```

### Long Polling

Server holds request open until data is available, then responds. Client immediately re-opens. Better latency, but connection churn and complex error handling.

```javascript
// Long-poll loop
async function poll() {
  const res = await fetch('/api/updates?wait=30');
  const data = await res.json();
  render(data);
  poll(); // immediately re-connect
}
poll();
```

### Server-Sent Events (Diagram)

With **Short Polling**, the client repeatedly sends `GET /updates` and mostly receives `204 No Content` responses, wasting resources. Occasionally the server returns `200 + data`.

With **Server-Sent Events**, the client sends a single `GET /stream` request and the server pushes `data: event1`, `data: event2`, `data: event3`, etc. over the same connection indefinitely.

---

## Slide 04 -- What Are Server-Sent Events?

### Definition

SSE is an **HTML5 standard** (W3C / WHATWG) that allows a server to push events to a client over a **single, long-lived HTTP connection**. The client opens the connection once; the server writes events as they occur.

### Key Characteristics

- **Unidirectional** -- server to client only
- **Text-based** -- UTF-8 encoded event stream
- **Auto-reconnect** -- browser reconnects if dropped
- **Built-in** -- native `EventSource` API, no library needed
- **HTTP** -- works through proxies, firewalls, CDNs

### Standards & Support

| Spec | Status |
|------|--------|
| WHATWG HTML Living Standard S9.2 | Active |
| W3C Server-Sent Events (2015) | REC |
| Chrome / Edge / Firefox / Safari | Full support |
| IE 11 | No support (polyfill available) |
| Node.js / Deno / Bun | Server-side: trivial to implement |

### Minimal Example

**Client -- 3 lines:**

```javascript
const es = new EventSource('/stream');
es.onmessage = (e) => {
  console.log(e.data);
};
```

**Server response:**

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: Hello, SSE!

data: Another event
```

---

## Slide 05 -- SSE vs WebSockets vs Polling

| Feature | SSE | WebSocket | Short Polling | Long Polling |
|---------|-----|-----------|---------------|--------------|
| Direction | Server --> Client | Bidirectional | Client --> Server | Client --> Server |
| Protocol | HTTP/1.1 or HTTP/2 | `ws://` / `wss://` | HTTP | HTTP |
| Data format | Text (UTF-8) | Text + Binary | Any | Any |
| Auto-reconnect | **Built-in** | Manual | N/A | Manual |
| Event IDs / Resume | **Native** | DIY | DIY | DIY |
| Proxy/firewall friendly | **Yes** | Sometimes blocked | Yes | Yes |
| Browser API | `EventSource` | `WebSocket` | `fetch` | `fetch` |
| Max connections (HTTP/1.1) | 6 per domain | Unlimited | 6 per domain | 6 per domain |
| Complexity | **Low** | Medium | Low | Medium |

### Choose SSE when
- Server-to-client only (notifications, feeds, dashboards)
- You want auto-reconnect and event IDs for free
- HTTP infrastructure is non-negotiable

### Choose WebSocket when
- You need bidirectional messaging (chat, gaming)
- Binary data is required (audio, video chunks)
- Sub-millisecond latency matters

### Choose Polling when
- Updates are infrequent and non-time-critical
- Infrastructure cannot hold connections open
- Simplicity outweighs efficiency

---

## Slide 06 -- The Event Stream Protocol

### MIME Type & Headers

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

The response body is an **indefinite-length text stream**. Each event is separated by a **blank line** (`\n\n`).

### Full Example Stream

```text
: This is a comment (ignored by client)

id: 1
event: user-login
data: {"user":"alice","time":"10:42"}

id: 2
event: metric
data: {"cpu":42,"mem":68}

id: 3
data: Simple message (event = "message")

retry: 5000
```

### Field Reference

| Field | Purpose |
|-------|---------|
| `data:` | Event payload. Multiple `data:` lines are joined with `\n` |
| `event:` | Event type name. Defaults to `"message"` if omitted |
| `id:` | Sets the last event ID. Sent back on reconnect via `Last-Event-ID` header |
| `retry:` | Reconnection delay in ms. Browser uses this for auto-reconnect timing |
| `:` (colon) | Comment line. Used as keep-alive heartbeat |

### Rules

- Lines ending with `\n`, events separated by `\n\n`
- Field names are case-sensitive
- Unknown fields are silently ignored
- Max event size: browser-dependent (typically no hard limit)
- Multi-line data uses repeated `data:` fields

---

## Slide 07 -- EventSource Browser API

### Constructor

```javascript
// Basic usage
const es = new EventSource('/api/stream');

// With credentials (cookies / auth)
const es = new EventSource('/api/stream', {
  withCredentials: true
});
```

### Event Handlers

```javascript
// Default "message" events
es.onmessage = (e) => {
  console.log(e.data);       // string payload
  console.log(e.lastEventId); // event ID
  console.log(e.origin);      // server origin
};

// Connection opened
es.onopen = () => {
  console.log('Connected!');
};

// Error / disconnection
es.onerror = (e) => {
  if (es.readyState === EventSource.CONNECTING) {
    console.log('Reconnecting...');
  }
};
```

### readyState Property

| Constant | Value | Meaning |
|----------|-------|---------|
| `CONNECTING` | 0 | Connecting or reconnecting |
| `OPEN` | 1 | Connection established |
| `CLOSED` | 2 | Connection closed, no retry |

### Lifecycle & Auto-Reconnect

1. `new EventSource(url)` -- state: CONNECTING
2. `onopen` fires on 200 OK -- state: OPEN
3. `onerror` fires on disconnect -- state: CONNECTING (auto-reconnect)
4. Auto-retry after retry delay
5. `es.close()` -- state: CLOSED (permanent)

### Closing the Connection

```javascript
// Permanently close -- no auto-reconnect
es.close();
// readyState === EventSource.CLOSED (2)
```

---

## Slide 08 -- Custom Event Types

### Server Side -- Named Events

```text
event: notification
data: {"type":"info","msg":"Deploy started"}

event: notification
data: {"type":"success","msg":"Deploy complete"}

event: heartbeat
data: ping

event: metric
data: {"cpu":23,"mem":55,"disk":71}
```

The `event:` field sets the event type. Without it, events are dispatched as `"message"`.

### Why Custom Events?

- Separate concerns -- different handlers for different data
- Ignore events you do not care about (no handler = no dispatch)
- Cleaner code than parsing a "type" field inside `data`
- Works with `addEventListener`, not `onmessage`

### Client Side -- addEventListener

```javascript
const es = new EventSource('/api/events');

// Only fires for event: notification
es.addEventListener('notification', (e) => {
  const { type, msg } = JSON.parse(e.data);
  showToast(type, msg);
});

// Only fires for event: metric
es.addEventListener('metric', (e) => {
  const { cpu, mem, disk } = JSON.parse(e.data);
  updateDashboard({ cpu, mem, disk });
});

// Only fires for event: heartbeat
es.addEventListener('heartbeat', (e) => {
  resetIdleTimer();
});

// Catch-all for unnamed events
es.onmessage = (e) => {
  console.log('Unnamed event:', e.data);
};
```

### Important: onmessage vs addEventListener

- `onmessage` only receives events **without** an `event:` field
- Named events require `addEventListener('eventName', ...)`
- You can use both simultaneously

---

## Slide 09 -- Last-Event-ID & Reconnection

### How It Works

1. Client sends `GET /stream`
2. Server sends events with IDs: `id:1 data:A`, `id:2 data:B`, `id:3 data:C`
3. **CONNECTION LOST**
4. Browser waits (retry delay)
5. Client reconnects with `GET /stream` and header `Last-Event-ID: 3`
6. Server resumes: `id:4 data:D`, `id:5 data:E`, ...

### Server-Side Handling

```javascript
app.get('/stream', (req, res) => {
  const lastId = req.headers['last-event-id'];

  // Resume from where client left off
  let cursor = lastId
    ? parseInt(lastId, 10)
    : 0;

  // Send missed events
  const missed = eventLog.filter(
    e => e.id > cursor
  );
  missed.forEach(e => {
    res.write(`id: ${e.id}\n`);
    res.write(`data: ${e.data}\n\n`);
  });

  // Continue streaming new events...
});
```

### Retry Field

```text
retry: 5000

data: After disconnect, browser waits
data: 5 seconds before reconnecting
```

- Default retry is browser-specific (~3s)
- Server can adjust at any time during the stream

---

## Slide 10 -- Node.js SSE Server

### Express Implementation

```javascript
import express from 'express';
const app = express();
const clients = new Set();

app.get('/events', (req, res) => {
  // 1. Set SSE headers
  res.writeHead(200, {
    'Content-Type':  'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection':    'keep-alive',
    'X-Accel-Buffering': 'no', // Nginx
  });

  // 2. Send initial retry interval
  res.write('retry: 5000\n\n');

  // 3. Register client
  clients.add(res);
  console.log(`Client connected (${clients.size})`);

  // 4. Handle disconnect
  req.on('close', () => {
    clients.delete(res);
    console.log(`Client left (${clients.size})`);
  });
});

// Broadcast helper
function broadcast(event, data, id) {
  const msg =
    (id    ? `id: ${id}\n`      : '') +
    (event ? `event: ${event}\n` : '') +
    `data: ${JSON.stringify(data)}\n\n`;

  for (const client of clients) {
    client.write(msg);
  }
}

app.listen(3000);
```

### Keep-Alive Heartbeat

```javascript
// Send comment as heartbeat every 30s
setInterval(() => {
  for (const client of clients) {
    client.write(': heartbeat\n\n');
  }
}, 30_000);
```

Prevents proxies and load balancers from closing idle connections. The `:` prefix makes it a comment, ignored by `EventSource`.

### Critical Headers Explained

| Header | Why? |
|--------|------|
| `Content-Type: text/event-stream` | Required for `EventSource` to accept the response |
| `Cache-Control: no-cache` | Prevents caching of the stream |
| `Connection: keep-alive` | Keeps TCP connection open (HTTP/1.1) |
| `X-Accel-Buffering: no` | Disables Nginx response buffering |

### Usage Example

```javascript
// Trigger from any route or event
app.post('/deploy', (req, res) => {
  broadcast('deploy', {
    status: 'started',
    user: req.user.name,
    time: new Date().toISOString()
  }, nextId());
  res.json({ ok: true });
});
```

---

## Slide 11 -- SSE with Fetch API

The native `EventSource` API lacks support for custom headers, POST requests, and request bodies. The Fetch API with `ReadableStream` provides full control.

### Fetch + ReadableStream

```javascript
async function streamSSE(url, token) {
  const res = await fetch(url, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      'Accept': 'text/event-stream',
    },
    body: JSON.stringify({ channel: 'deploys' }),
  });

  const reader = res.body
    .pipeThrough(new TextDecoderStream())
    .getReader();

  let buffer = '';
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    buffer += value;
    const events = buffer.split('\n\n');
    buffer = events.pop(); // keep incomplete

    for (const raw of events) {
      const event = parseSSE(raw);
      if (event) handleEvent(event);
    }
  }
}
```

### SSE Parser

```javascript
function parseSSE(raw) {
  const event = { data: '', event: 'message',
                  id: null, retry: null };
  for (const line of raw.split('\n')) {
    if (line.startsWith('data: '))
      event.data += line.slice(6) + '\n';
    else if (line.startsWith('event: '))
      event.event = line.slice(7);
    else if (line.startsWith('id: '))
      event.id = line.slice(4);
    else if (line.startsWith('retry: '))
      event.retry = parseInt(line.slice(7));
  }
  event.data = event.data.trimEnd();
  return event.data ? event : null;
}
```

### When to Use Fetch over EventSource

| Need | EventSource | Fetch API |
|------|-------------|-----------|
| Custom headers (Bearer token) | No | **Yes** |
| POST / PUT / PATCH | No (GET only) | **Yes** |
| Request body | No | **Yes** |
| Auto-reconnect | **Built-in** | Manual |
| Last-Event-ID | **Automatic** | Manual |

---

## Slide 12 -- Real-World: Live Notifications

### Server -- Notification Stream

```javascript
const userConnections = new Map(); // userId => Set<res>

app.get('/notifications/stream', auth, (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });

  const userId = req.user.id;
  if (!userConnections.has(userId))
    userConnections.set(userId, new Set());
  userConnections.get(userId).add(res);

  // Send unread count on connect
  const unread = getUnreadCount(userId);
  res.write(`event: unread-count\n`);
  res.write(`data: ${unread}\n\n`);

  req.on('close', () => {
    userConnections.get(userId)?.delete(res);
  });
});

// Called when a new notification is created
function notifyUser(userId, notification) {
  const conns = userConnections.get(userId);
  if (!conns) return;
  const payload =
    `id: ${notification.id}\n` +
    `event: notification\n` +
    `data: ${JSON.stringify(notification)}\n\n`;
  for (const res of conns) res.write(payload);
}
```

### Client -- React Component

```jsx
function NotificationBell() {
  const [count, setCount] = useState(0);
  const [items, setItems] = useState([]);

  useEffect(() => {
    const es = new EventSource(
      '/notifications/stream',
      { withCredentials: true }
    );

    es.addEventListener('unread-count', (e) => {
      setCount(Number(e.data));
    });

    es.addEventListener('notification', (e) => {
      const n = JSON.parse(e.data);
      setItems(prev => [n, ...prev]);
      setCount(prev => prev + 1);
      showToast(n.title, n.body);
    });

    return () => es.close();
  }, []);

  return <Bell count={count} items={items} />;
}
```

### Architecture

```
Browser --SSE--> API Server ---> Database
                           \---> Redis Pub/Sub
```

---

## Slide 13 -- Real-World: Live Dashboards

### Server -- Metric Stream

```javascript
import os from 'node:os';

app.get('/metrics/stream', (req, res) => {
  res.writeHead(200, {
    'Content-Type':  'text/event-stream',
    'Cache-Control': 'no-cache',
  });

  let id = 0;
  const interval = setInterval(() => {
    const cpus = os.cpus();
    const load = os.loadavg();
    const mem  = process.memoryUsage();

    const payload = {
      ts:   Date.now(),
      cpu:  Math.round(load[0] * 100 / cpus.length),
      mem:  Math.round(mem.heapUsed / 1e6),
      rss:  Math.round(mem.rss / 1e6),
      conns: clients.size,
    };

    res.write(`id: ${++id}\n`);
    res.write(`event: metric\n`);
    res.write(`data: ${JSON.stringify(payload)}\n\n`);
  }, 1000);

  req.on('close', () => clearInterval(interval));
});
```

### Client -- Streaming to Chart.js

```javascript
const es = new EventSource('/metrics/stream');
const MAX_POINTS = 60; // 1 minute window

const chart = new Chart(ctx, {
  type: 'line',
  data: {
    labels: [],
    datasets: [{
      label: 'CPU %',
      data: [],
      borderColor: '#f5a623',
    }, {
      label: 'Heap MB',
      data: [],
      borderColor: '#4ecb8d',
    }]
  },
  options: { animation: false }
});

es.addEventListener('metric', (e) => {
  const m = JSON.parse(e.data);
  const t = new Date(m.ts).toLocaleTimeString();

  chart.data.labels.push(t);
  chart.data.datasets[0].data.push(m.cpu);
  chart.data.datasets[1].data.push(m.mem);

  if (chart.data.labels.length > MAX_POINTS) {
    chart.data.labels.shift();
    chart.data.datasets.forEach(d => d.data.shift());
  }
  chart.update();
});
```

### Ideal SSE Dashboard Use Cases

- **Infrastructure monitoring** -- CPU, memory, disk, network
- **CI/CD pipelines** -- build progress, deploy status
- **Business KPIs** -- sales, signups, active users
- **Log tailing** -- live log viewer in the browser

---

## Slide 14 -- Real-World: AI/LLM Streaming

SSE is the **de facto standard** for LLM streaming responses. OpenAI, Anthropic, Google, and others all use `text/event-stream` for token-by-token output.

### Server -- Streaming LLM Proxy

```javascript
app.post('/chat', async (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
  });

  const stream = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: req.body.messages,
    stream: true,
  });

  for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content;
    if (content) {
      res.write(`data: ${JSON.stringify({
        content
      })}\n\n`);
    }
  }

  res.write('data: [DONE]\n\n');
  res.end();
});
```

### Client -- Token-by-Token Rendering

```javascript
async function chat(messages) {
  const res = await fetch('/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages }),
  });

  const reader = res.body
    .pipeThrough(new TextDecoderStream())
    .getReader();

  let buffer = '';
  let output = '';
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buffer += value;

    const parts = buffer.split('\n\n');
    buffer = parts.pop();
    for (const part of parts) {
      const data = part.replace('data: ', '');
      if (data === '[DONE]') return output;
      const { content } = JSON.parse(data);
      output += content;
      renderMarkdown(output); // live update
    }
  }
}
```

### Why SSE for LLMs?

- **Time-to-first-token** -- user sees output immediately
- **Perceived performance** -- feels fast even for 10s+ responses
- **Standard HTTP** -- works through CDNs and API gateways

---

## Slide 15 -- SSE over HTTP/2

### HTTP/1.1 Connection Limit

Browsers enforce a **maximum of 6 connections per domain** under HTTP/1.1. Each SSE stream consumes one connection. With 6 tabs open, the browser is fully blocked.

**HTTP/1.1:** 6 connection slots (SSE #1 through SSE #6), then BLOCKED.

**HTTP/2:** Single TCP connection --> 100+ concurrent streams (multiplexed).

### Workarounds for HTTP/1.1

- Use a **subdomain** for the SSE endpoint: `stream.example.com`
- **Multiplex** multiple event types over a single SSE connection
- Use `SharedWorker` to share one connection across tabs
- Upgrade to **HTTP/2** (recommended)

### HTTP/2 Multiplexing Advantages

- **No connection limit** -- all streams share one TCP connection
- **Header compression** -- HPACK reduces overhead
- **Server push** -- can proactively send resources
- **Priority** -- streams can be prioritized
- **Binary framing** -- more efficient than text-based HTTP/1.1

### Nginx HTTP/2 Config

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    location /events {
        proxy_pass http://backend:3000;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
    }
}
```

### SharedWorker Tab Sharing

```javascript
// shared-worker.js
const ports = new Set();
const es = new EventSource('/stream');
es.onmessage = (e) => {
  for (const port of ports)
    port.postMessage(e.data);
};
onconnect = (e) => ports.add(e.ports[0]);
```

---

## Slide 16 -- Authentication & CORS

### Cookie-Based Auth (EventSource)

```javascript
// Client
const es = new EventSource('/api/stream', {
  withCredentials: true  // sends cookies
});

// Server (Express + CORS)
app.use(cors({
  origin: 'https://app.example.com',
  credentials: true
}));

app.get('/api/stream', (req, res) => {
  // req.cookies available via cookie-parser
  if (!req.session.userId) {
    return res.status(401).end();
  }
  // ... stream events
});
```

### Token via Query String (EventSource)

```javascript
// EventSource can't set custom headers
// Pass token in URL (use with caution)
const token = getAuthToken();
const es = new EventSource(
  `/api/stream?token=${token}`
);

// Server-side validation
app.get('/api/stream', (req, res) => {
  const token = req.query.token;
  const user = verifyJWT(token);
  if (!user) return res.status(401).end();
  // ... stream events
});
```

**Warning:** Tokens in URLs appear in access logs, browser history, and Referer headers. Use short-lived tokens.

### Bearer Token (Fetch API)

```javascript
// Full header control with Fetch
const res = await fetch('/api/stream', {
  headers: {
    'Authorization': `Bearer ${token}`,
    'Accept': 'text/event-stream',
  }
});
// ... process ReadableStream
```

### CORS Headers for SSE

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: Authorization, Last-Event-ID
Access-Control-Expose-Headers: X-Request-Id
```

### Auth Strategy Comparison

| Method | EventSource | Fetch SSE | Security |
|--------|-------------|-----------|----------|
| Cookies | Yes | Yes | Good (HttpOnly, SameSite) |
| Query token | Yes | Yes | Weak (URL exposure) |
| Bearer header | No | Yes | Good |

---

## Slide 17 -- Error Handling & Retry Logic

### EventSource Built-in Retry

```javascript
const es = new EventSource('/stream');

es.onerror = (e) => {
  switch (es.readyState) {
    case EventSource.CONNECTING: // 0
      // Browser is auto-reconnecting
      console.log('Reconnecting...');
      break;
    case EventSource.CLOSED: // 2
      // Server sent HTTP error or closed
      console.log('Connection closed');
      break;
  }
};
```

The browser automatically reconnects when the connection drops. The server controls the delay via the `retry:` field.

### Server-Side: Graceful Shutdown

```javascript
// Send retry: 0 to stop reconnects
function closeAllClients(reason) {
  for (const client of clients) {
    client.write(`event: shutdown\n`);
    client.write(`data: ${reason}\n\n`);
    client.end();
  }
  clients.clear();
}

// Signal permanent close with HTTP 204
app.get('/stream', (req, res) => {
  if (maintenance) {
    return res.status(204).end(); // no reconnect
  }
  // ...
});
```

### Fetch API -- Exponential Backoff

```javascript
async function connectWithBackoff(url, opts = {}) {
  let attempt = 0;
  const maxDelay = 30_000;
  const baseDelay = 1_000;

  while (true) {
    try {
      await streamSSE(url, opts);
      attempt = 0; // reset on success
    } catch (err) {
      attempt++;
      const delay = Math.min(
        baseDelay * 2 ** attempt + Math.random() * 1000,
        maxDelay
      );
      console.log(`Retry #${attempt} in ${delay}ms`);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

### HTTP Status Code Behavior

| Status | EventSource Behavior |
|--------|---------------------|
| **200** | Connection established, events flow |
| **301/307** | Follows redirect, then connects |
| **204** | No reconnect -- permanent close |
| **401/403** | No reconnect -- fires `onerror` |
| **500/502/503** | Auto-reconnect after `retry` delay |

---

## Slide 18 -- Scaling SSE

### Challenge: Connection Limits

- Each SSE client holds an **open TCP connection**
- Node.js default: `ulimit -n` is often **1024**
- Each connection consumes memory (~3-10 KB)
- 10K connections = 30-100 MB RAM just for sockets

### Redis Pub/Sub Fan-Out

```javascript
import Redis from 'ioredis';
const sub = new Redis();
const pub = new Redis();

// Each server instance subscribes
sub.subscribe('events');
sub.on('message', (channel, msg) => {
  // Broadcast to local SSE clients
  for (const client of clients) {
    client.write(`data: ${msg}\n\n`);
  }
});

// Any server can publish
function emit(event) {
  pub.publish('events', JSON.stringify(event));
}
```

### Multi-Server Architecture

```
                 Load Balancer
                /      |      \
          Server A  Server B  Server C
                \      |      /
                 Redis Pub/Sub

All servers receive all events via Redis
```

### Nginx: Disable Buffering

```nginx
location /events {
    proxy_pass http://upstream;
    proxy_buffering off;       # critical!
    proxy_cache off;
    proxy_read_timeout 86400s; # 24h
    proxy_set_header Connection '';
    proxy_http_version 1.1;
}
```

### OS Tuning

```bash
# Raise file descriptor limit
ulimit -n 65536
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
```

---

## Slide 19 -- Security Considerations

### Threat: Connection Exhaustion (DoS)

An attacker opens thousands of SSE connections, exhausting server resources.

```javascript
// Mitigation: per-IP connection limits
const ipConnections = new Map();
const MAX_PER_IP = 10;

app.get('/stream', (req, res) => {
  const ip = req.ip;
  const count = ipConnections.get(ip) || 0;
  if (count >= MAX_PER_IP) {
    return res.status(429)
      .send('Too many connections');
  }
  ipConnections.set(ip, count + 1);
  req.on('close', () => {
    const c = ipConnections.get(ip) - 1;
    if (c <= 0) ipConnections.delete(ip);
    else ipConnections.set(ip, c);
  });
  // ... stream
});
```

### Threat: Memory Leaks

- Forgetting to remove clients from `Set` on disconnect
- Growing event buffers without bounds
- Intervals/timers not cleared on close

```javascript
// Always clean up
req.on('close', () => {
  clients.delete(res);
  clearInterval(timer);
  clearTimeout(timeout);
});
```

### Threat: Data Injection

If user-controlled data flows into SSE events, newlines can inject fields.

```javascript
// DANGEROUS: unsanitized user input
res.write(`data: ${userInput}\n\n`);
// If userInput = "hi\nevent: admin\ndata: pwned"
// attacker injects a fake event!

// SAFE: serialize as JSON
res.write(`data: ${JSON.stringify({
  message: userInput
})}\n\n`);
```

### Rate Limiting Events

```javascript
// Throttle: max 10 events/second/client
function throttledWrite(client, msg) {
  client._count = (client._count || 0) + 1;
  if (client._count > 10) return; // drop
  client.write(msg);
}
// Reset counters every second
setInterval(() => {
  for (const c of clients) c._count = 0;
}, 1000);
```

### Security Checklist

- Always use HTTPS in production
- Validate `Origin` header / set CORS properly
- JSON-encode all data payloads
- Set per-IP and per-user connection limits
- Authenticate before streaming
- Monitor connection counts and memory usage
- Implement idle timeouts (e.g., 2h max)

---

## Slide 20 -- Summary & Next Steps

### What We Covered

- SSE protocol format and fields
- EventSource API and lifecycle
- Custom events, IDs, reconnection
- Node.js server implementation
- Fetch API for advanced use cases
- Real-world patterns: notifications, dashboards, AI streaming
- Production: HTTP/2, auth, scaling, security

### Key Takeaways

- SSE is the **simplest** server push technology
- **Auto-reconnect + Last-Event-ID** = resilient by default
- Works over **standard HTTP** -- no special protocol
- Perfect for **unidirectional** real-time data
- HTTP/2 removes the 6-connection limit
- Use Fetch API when you need custom headers or POST
- Always **sanitize data** and limit connections

### Next Steps

- Build a live notification system
- Add SSE to an existing REST API
- Explore **WebTransport** for bidirectional streaming
- Try **NATS** or **Kafka** for event sourcing
- Benchmark: SSE vs WebSocket for your workload
- Read the [WHATWG SSE spec](https://html.spec.whatwg.org/multipage/server-sent-events.html)

### Essential Resources

| Resource | URL |
|----------|-----|
| WHATWG Living Standard | `html.spec.whatwg.org` S9.2 |
| MDN EventSource | `developer.mozilla.org/en-US/docs/Web/API/EventSource` |
| Can I Use | `caniuse.com/eventsource` |
| SSE Polyfill (IE11) | `github.com/Yaffle/EventSource` |

### When NOT to Use SSE

- **Bidirectional** communication needed --> use WebSocket
- **Binary data** streaming --> use WebSocket or WebTransport
- **Peer-to-peer** --> use WebRTC
- **Offline-first** --> use service workers + sync
