# API Design Essentials

## API Types

In an interview, you'll typically choose between three main API protocols:

1. REST (Representational State Transfer) - REST uses standard HTTP methods to manipulate resources identified by URLs. This option should be your default choice when designing APIs in System Design interviews. 
2. GraphQL - Unlike REST's fixed endpoints, GraphQL uses a single endpoint with a query language that lets clients specify exactly what data they need. If your interviewer mentions "flexible data fetching" or talks about avoiding over-fetching and under-fetching, they're signaling you to consider GraphQL.
3. RPC (Remote Procedure Calls) - RPC protocols like gRPC use binary serialization and HTTP/2 for efficient communication between services. While REST treats everything as resources, RPC lets you think in terms of actions and procedures. For example - When your user service needs to quickly validate permissions with your auth service, an RPC call like `checkPermission(userId, resource)` is more natural than trying to model this as a REST resource. If the interviewer specifically mentions microservices or internal APIs, consider RPC for those high-performance connections.

![API Types Flowchart](../assets/ApiTypesFlowchart.png)

## REST

### Resource Modeling

The foundation of good REST API design is identifying your resources correctly. Resources are just your core entities. Take Ticketmaster as an example. Your core entities might be events, venues, tickets, and bookings. These naturally map to REST resources:

```
GET /events                    # Get all events
GET /events/{id}               # Get a specific event
GET /venues/{id}               # Get a specific venue
GET /events/{id}/tickets       # Get available tickets for an event
POST /events/{id}/bookings     # Create a new booking for an event
GET /bookings/{id}             # Get a specific booking
```

Importantly, REST resources should represent things in your system, not actions. Instead of thinking about what users can do (like "book" or "purchase"), think about what exists in your system (events, venues, tickets, bookings). Also, resources should always be plural nouns. 

When handling relationships between resources, you have two main approaches. The key difference is whether the relationship is required or optional. Use path parameters (or nested resources) when the value is required. Use query parameters when the filter is optional. 

1. You can nest resources when there is a clear parent-child relationship. For example - `events/{id}/tickets`
2. You can keep resources flat and use query parameters. For example - `events/{id}/tickets?section=VIP`

### HTTP Methods

HTTP provides a set of methods (verbs) that map naturally to common operations

1. `GET` is for retrieving data without changing anything
2. `POST` creates new resources. POST operations are not typically idempotent
3. `PUT` replaces an entire resource with what you send, or create it if it doesn't exist. PUT is idempotent, so sending the same data multiple times results in the same final state.
4. `PATCH` updates part of a resource. Whether a PATCH is idempotent depends on the operation. For example - setting a user's email to a fixed value is idempotent (repeating it leaves the same final state), while appending an entry to the end of a list is not (each call adds another entry). 
5. `DELETE` removes a resource. DELETE operations are usually idempotent even though the response codes differ

### Passing Data to APIs

API endpoints need input to tell the server what to do. This can be which resources to fetch, what data to update in the database, or how to filter results. You have three main options for passing data to your REST API, and each serves a different purpose.

1. Path Parameter - Identify which specific resource you're working with. For example - `events/123`. The event ID is part of the URL structure itself. 
2. Query Parameter - Filter, sort or modify how you retrieve resources. `/events?city=NYC&date=2024-01-01`. These represent optional operations you want performed on the retrieved data. 
3. Request Body - Contains the actual data you're sending to create or update resources.

### Returning Data

An API Response is made up of two parts

1. The status code, which indicates whether the request was successful or not.
2. The response body, which contains the data you're returning to the client (typically JSON)

For status codes, stick to the common ones: 200 for success, 201 for created resources, 400 for bad requests, 401 for authentication required (the client isn't authenticated), 403 for forbidden (authenticated but not allowed), 404 for not found, and 500 for server errors.

## Common API Patterns

Regardless of whether you choose REST, GraphQL, or RPC, there are some patterns that apply across all API types.

1. Pagination - When you're dealing with large datasets, you can't return everything at once. Instead, you need pagination to break large result sets into manageable chunks. There are two main approaches to pagination: offset-based and cursor-based.
- Offset-based pagination - Offset-based pagination is the simplest approach and is used by most websites. You specify how many records to skip and how many to return: `/events?offset=20&limit=10` gets records 21-30. This is intuitive and easy to implement, but it has problems with large datasets. If someone adds a new event while you're paginating through results, you might see duplicates or miss records as the data shifts.
- Cursor-based pagination - Cursor-based pagination solves this by using a pointer to a specific record instead of counting from the beginning. The first request looks like this: `/events?limit=10`. The response to this request includes a pointer to the last record that was retrieved (For example - Something like `next_cursor`). You then use the cursor to skip over records that were previously retrieved to return a new set of records: `/events?cursor=cmd9atj3p000007ky19w1dpy2&limit=10`
2. Versioning - APIs evolve over time, and you need a strategy for handling changes without breaking existing clients. This is particularly important for public APIs where you can't control when clients update their code.
- URL Versioning - The most common approach is URL versioning, where you include the version number in the path: `/v1/events` or `/v2/events`. This is explicit and easy to understand.
- Header Versioning - Puts the version in an HTTP header instead: `Accept-Version: v2` or `API-Version: 2`. This keeps URLs cleaner and follows HTTP standards better, but it's less obvious to developers and harder to test in browsers.

## Security Considerations

Authentication verifies identity - proving the user is who they claim to be. Authorization verifies permissions - checking if that authenticated user is allowed to perform the specific action they're requesting. 

### API Keys

API keys are long, randomly generated strings that act like passwords for applications rather than humans. When a client makes a request, it includes its API key in the Authorization header, and your server looks up that key to identify which application is making the request.

### JSON Web Tokens (JWT)

JWTs encode user information directly into the token itself rather than storing session state on your server. When a user logs in successfully, your server creates a JWT containing their user ID, permissions, and an expiration time, then signs the entire token with a secret key.

Conveniently, when that JWT comes back with future requests, you can verify it's authentic by checking the signature, and you can read the user information directly from the token without any database lookups. The token itself carries all the context you need to authorize the request.

Use API keys for internal service communication and external developer access. Use JWT tokens for user sessions in web and mobile applications. JWT tokens can be stateless (no database lookup required) and can carry user context, making them ideal for user-facing applications.

### Role Based Access Control (RBAC)

RBAC assigns roles to users and permissions to roles. 

```
# Defining roles through RBAC and tagging each user with an appropriate role
Roles:
- customer: can book tickets, view own bookings
- venue_manager: can create events, view sales for their venues
- admin: can access everything

User: john@example.com → Role: customer
User: manager@venue.com → Role: venue_manager

When users use your API, you can use RBAC to perform authentication and authorization
GET /bookings/{id}
1. Is the user authenticated? (valid JWT token)
2. Is the user authorized? (owns this booking OR is admin)
```

### Rate Limiting and Throttling

Rate limiting prevents abuse by restricting how many requests a client can make in a given time period. This protects your system from both malicious attacks and accidental overuse. Common strategies include:

1. Per-user limits: 1000 requests per hour per authenticated user
2. Per-IP limits: 100 requests per hour for unauthenticated requests
3. Endpoint-specific limits: 10 booking attempts per minute to prevent ticket scalping

You typically implement rate limiting at the API gateway level or using middleware in your application. When limits are exceeded, return a `429 Too Many Requests` status code.

## Staff-Level Depth

The sections above cover the mechanics of API design. At a staff-level interview, the interviewer already assumes you know the mechanics. What distinguishes a staff answer is reasoning about *failure, evolution, and cost* — what happens when the network is unreliable, when clients retry, when the API has to change without breaking millions of existing integrations, and what each design choice costs you operationally. This section layers that depth onto the fundamentals above.

### Idempotency Keys (Safe Retries for Non-Idempotent Operations)

The idempotency table above tells you which HTTP methods are *naturally* idempotent. The staff-level follow-up is almost always: "A client calls `POST /events/{id}/bookings`, the network times out, and the client retries. How do you avoid double-booking?"

The answer is an **idempotency key**. The client generates a unique key (typically a UUID) per logical operation and sends it in a header:

```
POST /events/123/bookings
Idempotency-Key: 8f14e45f-ceea-467a-9575-27a3f3c9f2b1
```

The server stores the key alongside the result of the first successful request. On a retry with the same key, the server returns the *original* response instead of creating a second booking. Key design decisions the interviewer will probe:

1. How long do you retain keys? (A TTL — usually 24-48h — balances storage against realistic retry windows.)
2. What happens if a retry arrives *while the first request is still in flight*? (You need a lock or a "request in progress" state to avoid a race, typically returning `409 Conflict` for the concurrent duplicate.)
3. What scope does the key have? (Usually per-endpoint + per-user, so keys can't collide or leak across tenants.)

**The key must be stable across retries.** This is the whole game: the client generates the key *once, before the first attempt*, and reuses the *same* key for every retry of that logical operation. If the client mints a fresh UUID per retry, the mechanism is defeated — the server sees each attempt as a new operation and double-books. The key identifies the *intent* ("book this seat"), not the individual HTTP request.

#### Client-generated vs. content-based keys

There are two ways to produce the key, and they fail on opposite edges:

- **Client-generated (random UUID)** — the client mints a random key per operation and must persist it across retries. Handles *intentionally identical* operations correctly (two separate clicks to buy two identical general-admission tickets get two keys, so both succeed), but relies on the client honoring the "reuse the same key on retry" contract.
- **Content-based (deterministic hash)** — derive the key by hashing the canonicalized request, e.g. `sha256(canonical(payload) + user_id + endpoint)`. Retries dedupe automatically with nothing to persist, but two *legitimately identical* operations collapse into one false duplicate. Canonicalization (stable JSON key order, whitespace, encoding) is mandatory or identical requests hash differently. To support intentional duplicates you must fold in a discriminator (timestamp, sequence number, nonce) — which puts the "contribute something unique per intent" burden right back on the client.

Neither removes the need for a **server-side business uniqueness constraint** (e.g. a unique index on `(event_id, seat_id)`) as the correctness backstop. The idempotency key optimizes the *clean retry path* and returns the original response; the constraint makes double-actions *impossible* regardless of client behavior, at the cost of an uglier `409` the client must interpret. A robust design uses both.

This is the single most common staff-level API question. **Stripe's core REST API** (`api.stripe.com`, on all `POST` requests) is the canonical reference to cite. It's a hybrid worth describing:

1. Client sends a `Idempotency-Key` header (Stripe recommends a V4 UUID) — client-generated and reused across retries.
2. Stripe stores the first request's status code and response body and *replays it verbatim* on any retry, regardless of outcome.
3. It *also* stores a fingerprint (hash) of the request parameters, and rejects a reuse of the same key with a *different* payload — content-based hashing used **defensively to detect misuse**, not to generate the key.
4. A concurrent request with the same key while the first is in flight gets a `409 Conflict`.
5. Keys expire after 24 hours.

### Error Handling as a Design Surface

Status codes tell the client *what category* of thing happened. They are not enough on their own. A well-designed API returns a **structured, machine-readable error body** so clients can program against failures without string-matching:

```json
{
  "error": {
    "code": "SEAT_ALREADY_BOOKED",
    "message": "Seat 14B is no longer available.",
    "request_id": "req_9a8b7c6d",
    "retryable": false
  }
}
```

- Use a stable, documented `code` enum — clients branch on this, never on the human-readable `message`.
- Include a `request_id` so you (and the client) can correlate against your logs when they open a support ticket.
- For transient failures (`429`, `503`), signal *when* to retry with a `Retry-After` header, and design clients to back off exponentially with jitter.
- Distinguish client errors (`4xx`, don't retry — the request is wrong) from server errors (`5xx`, safe to retry).
- For partial failures in batch operations, decide explicitly: all-or-nothing, or per-item status (see Bulk/Async below).

### API Evolution and Backward Compatibility

Versioning (covered above) is what you do when you've *failed* to evolve compatibly. The staff-level instinct is to avoid a new version for as long as possible, because every version you support is a maintenance and testing burden forever.

The techniques that let you evolve without versioning:

1. **Additive-only changes** — adding a new optional field or a new endpoint never breaks an existing client. Removing or renaming a field does.
2. **Tolerant reader / robustness principle** — clients ignore fields they don't recognize, so servers can add fields freely.
3. **Never change the meaning or type of an existing field** — that's a breaking change even if the field name is unchanged.
4. **Deprecation policy** — when you must break, you announce it, expose a `Deprecation` / `Sunset` header, run old and new in parallel through a migration window, and monitor which clients still call the old path.

When you *do* version, prefer a small number of major versions and treat a new major version as an expensive, rare event — not a routine release.

### Concurrency and Consistency Control

Your Ticketmaster example has an obvious hazard: two users try to book the same seat simultaneously. Idempotency keys protect against *retries of the same request*; they do nothing about *two different clients racing*. For that you need concurrency control.

**Optimistic concurrency with ETags** is the REST-native approach. The server returns a version identifier on read:

```
GET /bookings/456
→ 200 OK
  ETag: "v7"
```

The client echoes it back on write with `If-Match`:

```
PUT /bookings/456
If-Match: "v7"
```

If the resource has changed since (someone else wrote `v8`), the server rejects with `412 Precondition Failed` and the client re-reads and retries. This turns a silent lost-update into an explicit, handleable conflict. Also be ready to discuss **read-after-write consistency** expectations — if your write path is asynchronous or replicated, a client that reads immediately after writing may not see its own change, and the API contract should make that explicit.

### Bulk, Batch, and Long-Running (Async) Operations

Chatty APIs that force clients into N round-trips are a scaling problem. Two patterns come up:

1. **Bulk / batch endpoints** — accept many items in one request (e.g. `POST /events/{id}/bookings:batch`). The key design question is *partial failure semantics*: does the whole batch succeed or fail atomically, or does each item get an independent status? Return a per-item result array with individual status codes when atomicity isn't required.
2. **Long-running operations** — when work can't complete within a request timeout (generating a report, processing a large upload), don't block. Return `202 Accepted` with a handle to a status resource:

```
POST /reports
→ 202 Accepted
  Location: /reports/789/status

GET /reports/789/status
→ 200 OK  { "status": "processing" }  ... eventually  { "status": "done", "result_url": "..." }
```

The client polls the status resource, or you push completion via a **webhook** so the client isn't polling at all. Webhooks introduce their own design surface — delivery retries, signing payloads so the receiver can verify authenticity, and idempotency on the receiver's side because you'll deliver at-least-once.

### Protocol Trade-offs (Beyond "When to Pick Each")

The API Types section says *when* to reach for each protocol. A staff answer also weighs what each one *costs*:

1. **GraphQL** — flexible fetching is real, but it moves query-planning cost to the server. Watch for the **N+1 problem** (a nested query fanning out into hundreds of DB calls — mitigated with batching/DataLoader), **caching difficulty** (a single POST endpoint defeats HTTP caching and CDNs), and **query cost / depth limits** to stop a client from issuing a pathologically expensive query. It shines for client-driven, heterogeneous UIs; it's overkill for simple CRUD.
2. **gRPC** — excellent for high-throughput internal service-to-service calls (binary Protobuf, HTTP/2 multiplexing, streaming, strong typed contracts). Costs: **limited browser support** (needs a proxy like gRPC-Web), **weaker human observability** (binary payloads aren't curl-friendly), and you own **schema evolution** discipline via Protobuf field numbering. Rarely the right choice for a public-facing API.
3. **REST** — the safe default precisely because it's cacheable, debuggable, and universally understood. Its cost is over-fetching/under-fetching and chattiness, which is exactly what GraphQL and bulk endpoints exist to address.

### Auth Depth: Token Lifecycle and Authorization Placement

The JWT section covers the happy path. The trade-off the interviewer probes is that JWTs are **hard to revoke** — the whole point is that you *don't* hit a database to validate them, which means a compromised or logged-out token stays valid until it expires. Staff-level mitigations:

1. **Short-lived access tokens + long-lived refresh tokens** — access tokens expire in minutes, so the revocation window is small; refresh tokens (which *are* checked against server state) mint new access tokens and can be revoked centrally.
2. **A revocation/denylist** for the "must kill this session now" case — a small amount of state that reintroduces a lookup, accepted as the cost of revocability.
3. **Key rotation** — sign with rotating keys (via a `kid` header and a published JWKS) so a leaked signing key doesn't compromise you forever. Never accept `alg: none`.

On **authorization placement**: coarse checks (is this token valid, is this user rate-limited) belong at the gateway; fine-grained checks (does *this* user own *this* booking) must happen at the service that owns the data, because only it has the context. Beyond RBAC, be ready to mention **scopes** (OAuth-style, "this token may only read events") and **ABAC** (attribute-based — decisions from user/resource/environment attributes) for cases where roles are too coarse.

### Rate Limiting: Algorithms and Placement

The Security section covers *why* and *what code* to return. The staff-level detail is *how* the limiter counts:

1. **Token bucket** — tokens refill at a steady rate; each request spends one. Allows short bursts up to the bucket size while bounding the sustained rate. Most common general-purpose choice.
2. **Fixed window** — simple counter per time window, but suffers a boundary problem: a client can send a full window's worth of requests at the end of one window and the start of the next, briefly doubling the intended rate.
3. **Sliding window** — smooths the boundary problem at the cost of more state/computation.

In a distributed system the limiter's counters must be *shared* across your fleet (e.g. in Redis), or each node enforces its own limit and the effective limit is N× what you intended. This "where does the counter live" point is what separates a staff answer from a bootcamp answer.

### Caching (HTTP-Native)

Worth a mention because it's cheap leverage and often forgotten. `Cache-Control` directives (`max-age`, `no-store`, `private` vs `public`) and `ETag` / `If-None-Match` conditional requests let clients and CDNs avoid redundant work — a `304 Not Modified` costs almost nothing. This is a first-class reason REST's uniform interface is valuable and a reason to think twice before a single-endpoint GraphQL design for read-heavy public data.

### HATEOAS (Know It, Usually Dismiss It)

REST purists advocate **HATEOAS** — responses embed links to the next available actions so clients discover the API dynamically rather than hardcoding URLs. In practice almost no one implements it fully; it adds complexity that most clients don't exploit. Worth one sentence in an interview to show you know the full REST maturity model (Richardson Maturity Model) and can make a deliberate, justified decision to *not* use it.
