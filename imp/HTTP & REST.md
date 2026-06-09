# HTTP & REST API - Interview Notes

---

## 1. HTTP Methods

HTTP methods define the **type of operation** performed on a resource in a RESTful API.

| Method | Purpose | Idempotent | Has Body |
|--------|---------|:----------:|:--------:|
| **GET** | Retrieve data from server | Yes | No |
| **POST** | Create a new resource | No | Yes |
| **PUT** | Replace an existing resource (or create if not exists) | Yes | Yes |
| **PATCH** | Partially update an existing resource | No | Yes |
| **DELETE** | Delete a resource | Yes | No (usually) |
| **OPTIONS** | Get supported HTTP methods for a resource (CORS preflight) | Yes | No |
| **HEAD** | Same as GET but without response body (headers only) | Yes | No |

### Example

```http
GET /api/users/123 HTTP/1.1
Host: myapp.com

---

POST /api/users HTTP/1.1
Content-Type: application/json

{ "name": "John", "email": "john@example.com" }
```

---

## 2. GET vs POST

| Feature | GET | POST |
|---------|-----|------|
| Purpose | Fetch data | Create a new resource |
| Idempotent | Yes | No |
| Request body | Not used (data in URL params) | Uses request body |
| Caching | Can be cached | Not cached by default |
| Bookmarkable | Yes | No |
| Data limit | URL length limit (~2048 chars) | No limit |
| Security | Data visible in URL | Data in body (more secure) |

### Example

```http
# GET - fetching a user
GET /api/users?id=123

# POST - creating a user
POST /api/users
Body: { "name": "John", "email": "john@example.com" }
```

---

## 3. PUT vs PATCH

| Feature | PUT | PATCH |
|---------|-----|-------|
| Purpose | **Replace entire resource** | **Partially update resource** |
| Idempotent | Yes | No (can be, but not guaranteed) |
| Request body | Full object required | Only fields to be updated |
| If field missing | Sets it to null/default | Leaves it unchanged |

### Example

```http
# PUT - replaces the ENTIRE user object
PUT /api/users/123
Body: { "name": "John", "email": "john@new.com", "age": 30, "city": "NYC" }

# PATCH - updates ONLY the email field, rest stays the same
PATCH /api/users/123
Body: { "email": "john@new.com" }
```

> **Interview Tip:** If you PUT without a field, it gets wiped. PATCH only touches what you send.

---

## 4. Stateless Behaviour

- Each request from client to server must contain **all information** necessary to understand and fulfill that request.
- Server does **not store any client state** between requests.
- No sessions on the server side — every request is independent.

### Why Stateless?
- **Scalability** — any server instance can handle any request (easy load balancing)
- **Reliability** — no state to lose if server crashes
- **Cacheability** — responses can be cached independently

### How is state maintained then?
- **Tokens (JWT)** — client sends token in every request header
- **Cookies** — browser automatically sends with each request
- **Query params** — state passed in URL

```http
# Stateless auth - token in every request
GET /api/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

---

## 5. Idempotency

A method is **idempotent** if calling it **multiple times** produces the **same result** as calling it once.

| Method | Idempotent? | Why? |
|--------|:-----------:|------|
| GET | Yes | Fetching data multiple times doesn't change it |
| PUT | Yes | Updating with same data gives same result |
| DELETE | Yes | Deleting same resource multiple times = same end state |
| POST | **No** | Creating same resource multiple times = multiple records |
| PATCH | **No** | Depends on implementation (could increment a counter) |

### Why does it matter?
- Network failures → client retries → **idempotent methods are safe to retry**
- POST is dangerous to retry (might create duplicates)
- Solution for POST: use **idempotency keys** (unique request ID sent by client)

```http
# Idempotency key to prevent duplicate payments
POST /api/payments
Idempotency-Key: abc-123-unique
Body: { "amount": 100, "to": "user_456" }
```

---

## 6. How to Secure a REST API

| Technique | What it does |
|-----------|-------------|
| **JWT** | Token-based authentication (stateless, no session) |
| **HTTPS** | Encrypts communication (prevents data sniffing) |
| **Rate Limiting** | Limits requests per user/IP (prevents abuse/DDoS) |
| **API Keys** | Identifies the calling application |
| **OAuth 2.0** | Standard authorization framework (delegated access) |
| **Input Validation** | Prevents SQL injection, XSS |
| **CORS** | Controls which domains can access the API |

### Example: JWT Flow

```
1. Client sends login credentials → POST /auth/login
2. Server verifies → returns JWT token
3. Client stores token (localStorage / cookie)
4. Every subsequent request includes: Authorization: Bearer <token>
5. Server validates token on each request (no DB lookup needed)
```

---

## 7. What is CORS?

**Cross-Origin Resource Sharing** — allows web apps to access resources from **different domains**.

- By default, browsers **block cross-origin requests** for security (Same-Origin Policy).
- CORS headers like `Access-Control-Allow-Origin` allow controlled cross-domain access.

![][image1]

### How it works

```
1. Browser makes request from http://frontend.com to http://api.backend.com
2. Browser sends preflight OPTIONS request (for non-simple requests)
3. Server responds with CORS headers:
   Access-Control-Allow-Origin: http://frontend.com
   Access-Control-Allow-Methods: GET, POST, PUT
   Access-Control-Allow-Headers: Authorization, Content-Type
4. If allowed, browser proceeds with actual request
```

### Spring Boot CORS Config

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://frontend.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*");
    }
}
```

---

## 8. REST vs SOAP

| Feature | REST | SOAP |
|---------|------|------|
| Protocol | HTTP only | HTTP, TCP, SMTP |
| Data format | JSON/XML (usually JSON) | XML only |
| Lightweight | Yes | No (heavy XML overhead) |
| Performance | Fast | Slow (XML parsing) |
| Security | HTTPS + OAuth/JWT | WS-Security (built-in) |
| Contract | No formal contract | WSDL (strict contract) |
| Statefulness | Stateless | Can be stateful |
| Use case | Web/Mobile APIs, Microservices | Enterprise, Banking, Legacy |

> **Interview Tip:** REST won because it's simpler, lighter, and JSON is easier to work with than XML. SOAP is still used in banking/insurance for strict contracts and built-in security.

---

## 9. HTTP Status Codes

### Success (2xx)

| Code | Meaning | Used With |
|------|---------|-----------|
| **200** | OK — Request succeeded | GET, PUT, PATCH, DELETE |
| **201** | Created — Resource created | POST |
| **204** | No Content — Success but no body returned | DELETE |

### Client Error (4xx)

| Code | Meaning | When |
|------|---------|------|
| **400** | Bad Request — Invalid syntax/validation failed | Malformed JSON, missing fields |
| **401** | Unauthorized — Authentication required | No token / invalid token |
| **403** | Forbidden — Authenticated but no permission | User doesn't have role |
| **404** | Not Found — Resource doesn't exist | Wrong URL / deleted resource |
| **405** | Method Not Allowed | POST on a GET-only endpoint |
| **409** | Conflict — Resource already exists | Duplicate email on signup |
| **422** | Unprocessable Entity — Validation error | Valid JSON but bad data |
| **429** | Too Many Requests — Rate limited | API abuse |

### Server Error (5xx)

| Code | Meaning | When |
|------|---------|------|
| **500** | Internal Server Error — Unhandled exception | Bug in code |
| **502** | Bad Gateway — Upstream server failed | Proxy/LB can't reach backend |
| **503** | Service Unavailable — Server overloaded/down | Maintenance, scaling |
| **504** | Gateway Timeout — Upstream took too long | Slow downstream service |

---

## 10. REST API Versioning

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URI Path | `/api/v1/users` | Simple, visible | Pollutes URL |
| Query Param | `/api/users?version=2` | Easy to add | Easy to miss |
| Header | `X-API-Version: 2` | Clean URL | Not visible in browser |
| Content-Type | `Accept: application/vnd.myapp.v2+json` | RESTful | Complex |

> **Most common in practice:** URI Path versioning (`/api/v1/...`)

---

## 11. Caching in REST API

Caching improves performance by avoiding redundant server calls.

### Cache-Control Headers

```http
# Server response - cache for 1 hour
Cache-Control: max-age=3600

# No caching at all
Cache-Control: no-store

# Cache but revalidate every time
Cache-Control: no-cache

# ETag - server sends hash, client sends back for comparison
ETag: "abc123"
If-None-Match: "abc123"  → Server returns 304 Not Modified
```

### Caching Strategies

| Strategy | How | Best For |
|----------|-----|----------|
| **Browser cache** | `Cache-Control` headers | Static assets, GET responses |
| **CDN** | Edge servers cache responses | Global APIs, static content |
| **Application cache** | Redis / in-memory | Frequently accessed data |
| **Reverse proxy** | Nginx/Varnish caches responses | High-traffic endpoints |

---

## 12. WebSockets vs REST API

| Feature | REST API | WebSockets |
|---------|----------|------------|
| Communication | Request-Response (client initiates) | **Bi-directional** (both can send) |
| Connection | New connection per request | **Persistent connection** |
| Protocol | HTTP | WS (WebSocket protocol) |
| Real-time | No (polling needed) | Yes (instant push) |
| Overhead | Higher (headers on each request) | Lower (single handshake) |
| Use case | CRUD operations | Chat, live feeds, gaming, notifications |
| Scalability | Easy (stateless) | Harder (stateful connections) |

### When to use WebSockets?
- **Chat applications** — real-time messaging
- **Live notifications** — push without polling
- **Stock tickers** — real-time price updates
- **Multiplayer games** — instant state sync
- **Collaborative editing** — Google Docs-style

### When to stick with REST?
- Standard CRUD operations
- Data doesn't change frequently
- Client doesn't need instant updates

---

## 13. OPTIONS Method (CORS Preflight)

The OPTIONS method is used by browsers as a **preflight request** before making cross-origin requests.

![][image2]

```http
# Browser automatically sends this before a cross-origin POST/PUT/DELETE
OPTIONS /api/users HTTP/1.1
Origin: http://frontend.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization

# Server responds with what's allowed
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://frontend.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

> `Access-Control-Max-Age: 86400` — browser caches preflight for 24h (avoids repeated OPTIONS calls)

---

## 14. Content Negotiation

Client and server agree on the **data format** via headers.

```http
# Client says "I want JSON"
Accept: application/json

# Client says "I'm sending JSON"
Content-Type: application/json

# Server can also return XML if client prefers
Accept: application/xml
```

---

## 15. GraphQL

### What is GraphQL?

- A **query language for APIs** — developed by Facebook (2015).
- Client specifies **exactly what data** it needs — no over-fetching, no under-fetching.
- Single endpoint: `POST /graphql` (unlike REST's many endpoints).
- Uses a **strongly-typed schema** to define available data.

### REST vs GraphQL

| Feature | REST | GraphQL |
|---------|------|---------|
| Endpoints | Multiple (`/users`, `/posts`) | Single (`/graphql`) |
| Data fetching | Fixed response structure | Client chooses fields |
| Over-fetching | Common (get all fields) | Never (get only what you ask) |
| Under-fetching | Common (need multiple calls) | Never (get everything in one query) |
| Versioning | `/v1/`, `/v2/` needed | No versioning (schema evolves) |
| Caching | Easy (HTTP caching) | Complex (custom caching) |
| Learning curve | Low | Medium-High |
| File upload | Simple (multipart) | Complex (needs spec) |

### Simple Example

**Schema Definition:**
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users: [User!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
  createPost(title: String!, content: String!, authorId: ID!): Post!
}
```

**Query (Client asks for specific fields):**
```graphql
# Get user with only name and their post titles - nothing extra!
query {
  user(id: "123") {
    name
    posts {
      title
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "user": {
      "name": "John",
      "posts": [
        { "title": "My First Post" },
        { "title": "GraphQL is awesome" }
      ]
    }
  }
}
```

**Mutation (Create/Update data):**
```graphql
mutation {
  createUser(name: "Jane", email: "jane@example.com") {
    id
    name
  }
}
```

### Pros of GraphQL

| Advantage | Explanation |
|-----------|-------------|
| **No over-fetching** | Client gets exactly the fields it needs |
| **No under-fetching** | Get related data in ONE request (no N+1 REST calls) |
| **Single endpoint** | One URL for everything — simpler routing |
| **Strongly typed** | Schema acts as documentation + validation |
| **Introspection** | Client can discover available queries/types at runtime |
| **Real-time** | Subscriptions for live updates (WebSocket-based) |
| **Frontend freedom** | Frontend devs don't need to wait for backend to add new endpoints |

### Cons of GraphQL

| Disadvantage | Explanation |
|--------------|-------------|
| **Complexity** | More complex server implementation than REST |
| **Caching is hard** | No HTTP caching (everything is POST) — need custom solutions |
| **N+1 problem** | Nested queries can cause N+1 DB queries (use DataLoader to fix) |
| **File uploads** | Not natively supported — needs multipart spec |
| **Rate limiting** | Hard to rate-limit (one query can be cheap or very expensive) |
| **Error handling** | Always returns 200 OK — errors are in the response body |
| **Overkill for simple APIs** | REST is simpler for basic CRUD |
| **Security** | Deeply nested queries can cause DoS (query depth limiting needed) |

### Deep Concepts

#### 1. N+1 Problem & DataLoader

```
# This query causes N+1 DB calls:
query {
  users {          # 1 query to get all users
    posts {        # N queries - one per user to get their posts!
      title
    }
  }
}
```

**Solution: DataLoader** — batches and caches DB calls.
```java
// Instead of N separate queries, DataLoader batches into 1:
// SELECT * FROM posts WHERE author_id IN (1, 2, 3, ..., N)
```

#### 2. Query Depth Limiting (Security)

Malicious clients can send deeply nested queries to overload the server:
```graphql
# Attack: deeply nested query
query {
  user(id: "1") {
    posts {
      author {
        posts {
          author {
            posts { ... } # infinite nesting!
          }
        }
      }
    }
  }
}
```

**Solution:** Set max query depth (e.g., 5 levels) and query complexity limits.

#### 3. Subscriptions (Real-time)

```graphql
# Client subscribes to new messages (WebSocket under the hood)
subscription {
  newMessage(chatId: "room1") {
    id
    text
    sender {
      name
    }
  }
}
```

#### 4. Fragments (Reusable Field Sets)

```graphql
fragment UserFields on User {
  id
  name
  email
}

query {
  user(id: "1") {
    ...UserFields
    posts { title }
  }
}
```

#### 5. Pagination (Cursor-based)

```graphql
query {
  users(first: 10, after: "cursor_abc") {
    edges {
      node { id, name }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

> **Interview Tip:** GraphQL is best for **mobile apps** (save bandwidth), **complex UIs** (dashboard with many data sources), and **microservices** (BFF pattern). REST is better for **simple CRUD**, **public APIs**, and **cacheable resources**.

### When to use GraphQL vs REST?

| Use GraphQL when... | Use REST when... |
|---------------------|-----------------|
| Mobile app needs minimal data | Simple CRUD API |
| Complex UI with many related entities | Public API (easy caching) |
| Multiple clients need different data shapes | File upload heavy |
| Rapid frontend iteration needed | Team prefers simplicity |
| Microservices aggregation (BFF) | Strong HTTP caching needed |
