# 📡 Introduction to Server-Sent Events

An interactive Reveal.js presentation covering Server-Sent Events (SSE) — the HTML5 standard for real-time, one-way server-to-client streaming over plain HTTP. Covers the protocol, browser APIs, Node.js implementation, real-world patterns (notifications, dashboards, AI/LLM streaming), and production concerns including scaling, security, and HTTP/2.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Introduction_to_Server_Sent_Events/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Introduction to Server-Sent Events |
| 02 | Agenda | Four-part roadmap: Foundations, Protocol, Building, Production |
| 03 | The Problem | Why server push matters; polling vs long-polling limitations |
| 04 | What Are Server-Sent Events? | HTML5 spec, one-way stream, key characteristics, browser support |
| 05 | SSE vs WebSockets vs Polling | Feature comparison table and decision guide |
| 06 | The Event Stream Protocol | text/event-stream MIME type, data/event/id/retry fields |
| 07 | EventSource Browser API | Constructor, event handlers, readyState, auto-reconnect lifecycle |
| 08 | Custom Event Types | Named events, addEventListener vs onmessage, dispatching |
| 09 | Last-Event-ID & Reconnection | Automatic reconnect, resuming from last ID, retry field |
| 10 | Node.js SSE Server | Express implementation, headers, broadcast, heartbeat |
| 11 | SSE with Fetch API | ReadableStream, TextDecoder, manual parsing, when to use |
| 12 | Real-World: Live Notifications | Per-user notification streaming with React client |
| 13 | Real-World: Live Dashboards | Streaming metrics to Chart.js in real time |
| 14 | Real-World: AI/LLM Streaming | Token-by-token output, ChatGPT-style streaming |
| 15 | SSE over HTTP/2 | Multiplexing, no 6-connection limit, SharedWorker sharing |
| 16 | Authentication & CORS | Cookie auth, query tokens, Bearer headers, CORS config |
| 17 | Error Handling & Retry Logic | Retry field, exponential backoff, HTTP status behavior |
| 18 | Scaling SSE | Connection limits, Redis Pub/Sub fan-out, Nginx config, OS tuning |
| 19 | Security | Connection exhaustion, memory leaks, data injection, rate limiting |
| 20 | Summary & Next Steps | Key takeaways, resources, and when not to use SSE |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## Markdown Version

A comprehensive markdown reference of all slide content is available in [`presentation.md`](presentation.md).

## References

- [WHATWG HTML Living Standard — Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [MDN Web Docs — EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
- [MDN Web Docs — Using Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [Can I Use — EventSource](https://caniuse.com/eventsource)
- [EventSource Polyfill (Yaffle)](https://github.com/Yaffle/EventSource)
- [Node.js Streams Documentation](https://nodejs.org/api/stream.html)
- [HTTP/2 Specification (RFC 7540)](https://datatracker.ietf.org/doc/html/rfc7540)

## License

Educational use. Code examples provided as-is.
