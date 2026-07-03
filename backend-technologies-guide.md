# Backend Technologies — System Design Reference

---

## 1. WebSocket

**Technologies / npm packages:** `ws`, `socket.io`, `uWebSockets.js` (Node); Django Channels, FastAPI WebSockets (Python); Spring WebSocket (Java)

**Pros:**
1. Full-duplex, persistent connection — server can push data without client asking
2. Very low latency after handshake (no repeated HTTP overhead per message)
3. Single TCP connection handles many message types (multiplexed via events)

**Cons:**
1. Stateful connections are hard to scale horizontally — a client is "pinned" to one server instance
2. Requires sticky sessions at the load balancer, or a pub/sub backplane (Redis/Kafka) to sync state across instances
3. Firewalls/proxies sometimes mishandle long-lived connections; needs heartbeat/ping-pong to detect dead clients

**Benefits:**
1. True bidirectional real-time communication
2. Eliminates polling overhead, cutting server load and bandwidth for high-frequency updates

**Use case scenarios:**
- *Explanatory:* Any feature where both client and server need to send unprompted messages to each other in near real time.
- *Expected:* Chat apps, live cursors/collaborative editing, multiplayer games, live trading dashboards, IoT device control.

**Senior system design thought:** WebSocket servers are stateful, so scaling means either sticky sessions (simple but uneven load distribution) or a shared pub/sub layer (Redis Pub/Sub, NATS, Kafka) so any server instance can broadcast to any connected client. Always plan a reconnect/backoff strategy client-side, and a fallback (SSE/long-polling) for restrictive networks.

**Example apps:** Slack, Discord, Figma (multiplayer cursors), Binance (live price feeds)

---

## 2. REST API (HTTP)

**Technologies / npm packages:** Express, Fastify, NestJS (Node); Django REST Framework, FastAPI (Python); Spring Boot (Java)

**Pros:**
1. Simple mental model — resources + verbs (GET/POST/PUT/DELETE)
2. Stateless, so any server instance can handle any request — easy horizontal scaling
3. Leverages standard HTTP caching (ETags, Cache-Control, CDNs)

**Cons:**
1. Over-fetching or under-fetching data (fixed response shape per endpoint)
2. Multiple round trips needed for related/nested resources
3. Not suited to real-time push — client must poll for updates

**Benefits:**
1. Statelessness makes it trivially scalable behind a load balancer
2. Massive tooling, documentation, and client library support (Postman, OpenAPI/Swagger)

**Use case scenarios:**
- *Explanatory:* Default choice for request/response style operations where data doesn't need to change every second.
- *Expected:* CRUD apps, public/partner APIs, mobile app backends, admin panels.

**Senior system design thought:** REST should be your default unless you have a specific reason to deviate (real-time, complex nested queries, internal high-throughput RPC). At scale, invest early in versioning strategy (`/v1/`, header-based), pagination (cursor over offset for large datasets), and idempotency keys for POST/PUT.

**Example apps:** Stripe API, GitHub REST API, Twitter API v1.1

---

## 3. GraphQL

**Technologies / npm packages:** Apollo Server, GraphQL Yoga, Hasura, Pothos

**Pros:**
1. Client specifies exactly the data shape it needs — no over/under-fetching
2. Single endpoint serves many different client needs (web, mobile, partners)
3. Strongly typed schema doubles as self-documenting contract

**Cons:**
1. N+1 query problem is easy to introduce without care (needs DataLoader/batching)
2. Caching is harder than REST since it's not URL-based
3. A single expensive nested query can overload the backend — needs query cost analysis/depth limiting

**Benefits:**
1. Reduces round trips for apps with deeply nested/related data
2. Great for platforms serving multiple frontends with different data needs from the same backend

**Use case scenarios:**
- *Explanatory:* When different clients need different slices of the same underlying data graph.
- *Expected:* Content platforms with web + mobile + partner APIs, dashboards aggregating many resources in one screen load.

**Senior system design thought:** Always pair GraphQL with a DataLoader-style batching layer to avoid N+1 hits on the database, and enforce query complexity/depth limits — an unauthenticated or malicious client can otherwise construct a query that fans out into thousands of DB calls.

**Example apps:** GitHub API v4, Shopify Storefront API, Facebook (origin)

---

## 4. gRPC

**Technologies / npm packages:** `@grpc/grpc-js`, Protocol Buffers (`protobuf`), grpc-web (for browser)

**Pros:**
1. Binary protocol (Protobuf) is much smaller and faster to serialize than JSON
2. Strong typing and code generation from `.proto` schema across languages
3. Native support for streaming (client, server, and bidirectional)

**Cons:**
1. Not natively browser-friendly — needs grpc-web plus a proxy (e.g., Envoy)
2. Binary payloads are hard to inspect/debug without tooling
3. Steeper learning curve than REST/JSON

**Benefits:**
1. Very low latency, ideal for high-volume internal service-to-service calls
2. Contract-first design keeps services in sync as they evolve

**Use case scenarios:**
- *Explanatory:* Internal, high-throughput communication between backend microservices where performance matters more than human-readability.
- *Expected:* Service mesh internals, real-time data pipelines between services, polyglot microservice fleets.

**Senior system design thought:** Common pattern — use gRPC for internal service-to-service traffic, and expose a REST or GraphQL gateway to the outside world that translates. This gets you gRPC's performance internally without forcing external clients to deal with Protobuf/browser limitations.

**Example apps:** Google internal infrastructure (origin), Netflix, Uber's microservices backbone

---

## 5. Message Queues / Event Streaming (Kafka, RabbitMQ, SQS)

**Technologies / npm packages:** `kafkajs`, `amqplib` (RabbitMQ), AWS SDK (SQS/SNS), BullMQ (Redis-based queues)

**Pros:**
1. Decouples producers from consumers — services don't need to know about each other
2. Absorbs traffic bursts (buffer) so downstream services aren't overwhelmed
3. Enables reliable async processing and retries

**Cons:**
1. Adds operational complexity (cluster management, monitoring, partitioning)
2. Eventual consistency — data isn't immediately reflected everywhere
3. Message ordering and exactly-once delivery are genuinely hard problems

**Benefits:**
1. Fault tolerance — a downstream outage doesn't lose data, it just queues up
2. Enables event-driven architecture and horizontal scaling of consumers independently

**Use case scenarios:**
- *Explanatory:* Any workflow where work can happen asynchronously, or where one event needs to fan out to multiple independent consumers.
- *Expected:* Order processing pipelines, email/push notification dispatch, log/metrics aggregation, event sourcing, microservice decoupling.

**Senior system design thought:** Choose Kafka when you need high-throughput event streaming with replay/history (event sourcing, analytics pipelines). Choose RabbitMQ/SQS when you need simpler task-queue semantics with flexible routing. Always design consumers to be idempotent — most queue systems guarantee at-least-once delivery, meaning duplicates *will* happen.

**Example apps:** Uber (trip event pipeline), LinkedIn (Kafka's origin), most e-commerce order systems

---

## 6. Caching (Redis / Memcached)

**Technologies / npm packages:** `ioredis`, `node-redis`, Memcached client libraries

**Pros:**
1. Sub-millisecond reads, drastically reduces load on the primary database
2. Redis supports rich data structures (lists, sets, sorted sets, streams) beyond simple key-value
3. Doubles as session store, rate limiter, and pub/sub broker

**Cons:**
1. Cache invalidation is notoriously hard to get right (stale data risk)
2. Adds another piece of infrastructure to run, monitor, and pay for
3. Memory-bound — large datasets require careful eviction policy planning

**Benefits:**
1. Major latency and cost reduction for read-heavy workloads
2. Enables features like leaderboards (sorted sets), rate limiting, and real-time counters cheaply

**Use case scenarios:**
- *Explanatory:* Any data that's read far more often than it changes, or that needs to be shared/fast across server instances.
- *Expected:* Session storage, API response caching, leaderboards, rate limiting, feed/timeline caching.

**Senior system design thought:** Cache invalidation is famously one of the two hard problems in computer science. Decide deliberately between cache-aside (lazy load + TTL), write-through, or write-behind strategies, and be explicit about your staleness tolerance per data type — don't apply one strategy everywhere.

**Example apps:** Twitter timeline caching, Instagram feed delivery, game leaderboards

---

## 7. Authentication / Authorization (JWT, OAuth2)

**Technologies / npm packages:** `jsonwebtoken`, `passport.js`, `next-auth`, Auth0/Clerk SDKs

**Pros:**
1. JWTs are stateless — no server-side session lookup needed, scales horizontally
2. OAuth2 is an industry standard for third-party/delegated login
3. Works cleanly across mobile, web, and API clients

**Cons:**
1. Revoking a JWT before expiry is hard (no built-in server-side "logout")
2. Tokens carry payload weight — larger than a simple session ID
3. OAuth2 flows (authorization code, PKCE, refresh) add real implementation complexity

**Benefits:**
1. Stateless auth scales without a shared session store
2. Standardized flows for SSO and third-party integrations reduce custom auth code

**Use case scenarios:**
- *Explanatory:* Any system needing to verify identity and control access, especially across multiple services or client types.
- *Expected:* User login systems, API authentication, "Sign in with Google/GitHub," internal service-to-service auth.

**Senior system design thought:** Use short-lived access tokens (minutes) plus longer-lived refresh tokens stored in httpOnly cookies — this balances statelessness with the ability to effectively "revoke" access by invalidating the refresh token. For high-security contexts, maintain a token revocation/blacklist check despite the added statefulness.

**Example apps:** "Sign in with Google" flows, Stripe API key auth, banking app refresh-token sessions

---

## 8. Server-Sent Events (SSE)

**Technologies / npm packages:** Native `EventSource` (browser), Express/Fastify with `res.write()` streaming

**Pros:**
1. Much simpler than WebSocket to implement for one-way server-to-client updates
2. Built-in auto-reconnect in the browser's EventSource API
3. Works over plain HTTP — no special proxy/firewall handling needed

**Cons:**
1. Strictly unidirectional (server → client only)
2. Browsers cap concurrent connections per domain (historically 6 over HTTP/1.1)
3. Text-only — no native binary data support

**Benefits:**
1. Lightweight real-time updates without WebSocket's infra complexity
2. Naturally fits streaming/incremental responses

**Use case scenarios:**
- *Explanatory:* Any one-directional live-update stream where the client never needs to talk back on the same channel.
- *Expected:* Live score/notification feeds, streaming AI/LLM token output, stock ticker updates.

**Senior system design thought:** Prefer SSE over WebSocket whenever the data flow is genuinely one-way — it's simpler to scale (works fine behind standard HTTP load balancers) and to debug. This is exactly why most LLM chat interfaces (including this one) stream responses over SSE rather than WebSocket.

**Example apps:** ChatGPT-style token streaming, live sports score widgets, stock tickers

---

## Quick Decision Cheat Sheet

| Need | Reach for |
|---|---|
| Bidirectional real-time | WebSocket |
| Simple CRUD / public API | REST |
| Flexible queries, multiple client shapes | GraphQL |
| Fast internal service-to-service calls | gRPC |
| Decoupled async processing | Message Queue (Kafka/RabbitMQ) |
| Reduce DB load / fast reads | Redis/Memcached |
| Identity & access control | JWT/OAuth2 |
| One-way live updates / streaming | SSE |
