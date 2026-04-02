---
name: rest-api-design
description: REST API design standards and guidelines. Use this skill whenever you are helping design a REST API, creating new REST endpoints in code, naming resources, choosing HTTP methods, structuring API responses, implementing pagination, designing error responses, picking status codes, planning API versioning, or deciding how to model a new route. Invoke proactively when the user mentions endpoints, routes, controllers, API design, HTTP methods, status codes, pagination, request/response format, API versioning, or asks how to structure any part of an HTTP API - even if they don't use the word "REST".
---

# REST API Design Standard

This standard defines how REST APIs should be designed for consistency, predictability, and long-term maintainability.

# Part 1 - Endpoint Design

## HTTP Method

| Method | Safe | Idempotent | Request Body | Response Body | When to use                                                                         |
|--------|------|------------|--------------|---------------|-------------------------------------------------------------------------------------|
| GET    | Yes  | Yes        | No           | Yes           | Retrieves a resource                                                                |
| POST   | No   | No ¹       | Yes          | Yes           | Creates a resource, triggers action                                                 |
| PUT    | No   | Yes        | Yes          | Yes ³         | Full replace of resource (field: absent→nulled, null→nulled, array→replaced)        |
| PATCH  | No   | No ²       | Yes          | Yes ³         | Partially updates a resource (field: absent→untouched, null→nulled, array→replaced) |
| DELETE | No   | Yes        | No           | No            | Removes a resource                                                                  |

- ¹ - POST can be made idempotent by accepting an `Idempotency-Key: <key>` header. The server stores the result keyed by that value and returns the cached response for any repeated request with the same key. Use for operations where duplicate execution would cause harm (e.g., payments, order submission).
- ² - PATCH idempotent when value-based (e.g., `{"name": "Alice"}`), not idempotent when operation-based (e.g., `{"op": "increment", "field": "count"}`). Array replacement may be overridden per endpoint if granular merge is needed.
- ³ - Response body for PUT and PATCH is optional — returning it is the recommended default, it helps avoid an extra GET round-trip

## Path Structure

Path `/{prefix}/{version}/{resource}[:{action}][?{query}]`
- `{prefix}` - always `/api`
- `{version}` - `v1`, `v2`, etc.
- `{resource}` - the entity being operated on (e.g. `/users`, `/orders` can be nested `/orders/{id}/items`)
- `:{action}` - optional - use when the operation doesn't fit standard CRUD semantics (e.g. `:search`, `:repeat`, `:batch-delete`); prefer a plain HTTP method + resource path before reaching for an action suffix.
- `?{query}` - optional - query string for filtering, sorting, pagination, and output control

Nested paths express ownership, not identity. Use the full path (`/orders/{orderId}/items/{itemId}`) when the child resource only exists in the context of its parent.

### Versioning

1. New Endpoint → start with `/api/v1/`
2. Non-breaking changes → no version bump needed:
   - Adding new optional request fields or query parameters
   - Adding new fields to a response
   - Relaxing validation
3. Breaking changes → requires a new version (affected endpoint only)
   - Removing or renaming fields
   - Changing a field type
   - Tightening validation
   - Changing the URL structure or HTTP method
   - Changing an authentication mechanism
   - Changing authorization requirements (e.g., making a public endpoint require auth, adding a required role or scope)
4. Sunsetting
   - Keep the old version running alongside the new one. Give consumers a minimum **6-month notice** before removing it
   - Use RFC 8594 header `Sunset` in response on every call to the deprecated version, for example `Sunset: Sat, 31 Dec 2099 23:59:59 GMT`
   - Avoid RFC 9745 header `Deprecation` for simplicity
   - Return 410 Gone from the old version after the sunset date has passed

## Naming Conventions
- **Path segment** uses **kebab-case** and **plural nouns**: `/api/v1/orders`, `/api/v1/gift-cards`
- **Path parameter** uses **camelCase**: `/api/v1/orders/{orderId}`, `/api/v1/gift-cards/{cardId}`
- **Action** uses **kebab-case**: `POST /api/v1/orders:search`, `POST /api/v1/orders/{id}:reset-tracker` — the colon distinguishes a verb from a sub-resource path segment
- **Query parameter** uses **camelCase**: `?pageSize=20&sortOrder=desc&createdAfter=2024-01-01`
- **JSON body field** (request and response) uses **camelCase**: `{ "orderId": "123", "createdAt": "..." }`
  - **Be self-descriptive** — spell out words unless the abbreviation is more universally recognized than the full term. `quantity` not `qty`, `message` not `msg`, `employeeId` not `empId`. Acceptable: universally known shorthands (`id`, `ssn`, `iban`, `pin`) or domain-standard abbreviations (`npi` in healthcare).
  - **Be consistent** — once a field name is established in any endpoint or service, use it everywhere. `dateOfBirth` everywhere — not `dob`, `birthDate`, or `birthday` in other endpoints or services.

## Examples
```
GET    /api/v1/orders                           # List orders
GET    /api/v1/orders/{orderId}                 # Get one order by ID
POST   /api/v1/orders                           # Create a new order
POST   /api/v1/orders:search                    # Search orders with complex criteria in the request body
POST   /api/v1/orders/{orderId}:repeat          # Repeat an existing order by ID (creates a new order based on the old one)
PUT    /api/v1/orders/{orderId}                 # Replace an existing order
PATCH  /api/v1/orders/{orderId}                 # Partially update an existing order
DELETE /api/v1/orders/{orderId}                 # Delete an order

GET    /api/v1/orders/{orderId}/items           # List items of an order
POST   /api/v1/orders/{orderId}/items           # Add an item to an order
DELETE /api/v1/orders/{orderId}/items/{itemId}  # Remove an item from an order
```

## Bulk Operations

Use `POST` and colon actions — one endpoint per action:
```
POST /api/v1/orders:batch-create    # Create multiple orders
POST /api/v1/orders:batch-update    # Partially update multiple orders
POST /api/v1/orders:batch-delete    # Delete multiple orders
```

Notes:
- Use `POST` for all batch operations: `DELETE` does not support a request body, and passing arrays of IDs via query params hits URL length limits. Using `POST` uniformly for all batch actions keeps the interface consistent and avoids mixing methods for the same pattern.
- One endpoint per action — avoid a generic multipurpose endpoint (e.g., `POST /api/v1/orders:batch`) that accepts a mixed array of create/update/delete operations.
- Always return `200` with per-item results in `data` — not `201` or `204` — because individual items may succeed or fail independently. Each item must include an identifier (`id` or array `index`), a `status` (`SUCCESS` or `FAILED`), and on failure an `error` with `code` and `message`.

---

# Part 2 - Communication Contract

## HTTP Status Codes

| Code                      | When to use                                                                                                  |
|---------------------------|--------------------------------------------------------------------------------------------------------------|
| 200 OK                    | Successful request, if created → 201, if no body → 204                                                       |
| 201 Created               | New resource created                                                                                         |
| 202 Accepted              | Request accepted but processing deferred; return `Location` header pointing to a status endpoint for polling |
| 204 No Content            | Successful request with no response body                                                                     |
| 400 Bad Request           | Request cannot be parsed (e.g., broken JSON syntax, unsupported `Content-Type`)                              |
| 401 Unauthorized          | Not Authenticated (e.g., no token, expired JWT)                                                              |
| 403 Forbidden             | Authenticated but not Authorized (e.g., user lacks role, permission, or scope)                               |
| 404 Not Found             | Resource doesn't exist (e.g., `GET /orders/1234` - order not found)                                          |
| 409 Conflict              | State conflict (e.g., duplicate unique email, optimistic lock)                                               |
| 410 Gone                  | Resource permanently removed; use after a versioned endpoint's sunset date                                   |
| 422 Unprocessable Entity  | Request is parseable but fails validation (e.g., negative age, missing required field)                       |
| 429 Too Many Requests     | Rate limit exceeded; always include `Retry-After` header                                                     |
| 500 Internal Server Error | Unexpected failure; never expose internal details to the client                                              |
| 502 Bad Gateway           | Upstream returned an invalid or unexpected response                                                          |
| 503 Service Unavailable   | Your service is overloaded or in maintenance; include Retry-After                                            |
| 504 Gateway Timeout       | Upstream service did not respond in time                                                                     |

Search and list endpoints always return `200` with an empty array — never `404` — when no results match.

## Response Structure

Responses may include the following top-level fields:
- `data`: single resource object or array of resources — always present on `2xx` responses with a body
- `meta`: contextual information (e.g., pagination); omit on single-resource responses by default
- `links`: reserved for HATEOAS; do not include it unless explicitly requested
- `error`: structured error info — always present on `4xx`/`5xx` responses

### Success Response

```json
{
  "data": [
    { "id": "1", "name": "Alice" },
    { "id": "2", "name": "Bob" }
  ],
  "meta": {
    "cursor": "abc1",
    "nextCursor": "abc2",
    "limit": 2,
    "hasMore": true
  }
}
```

### Error Response

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request has invalid fields",
    "details": [
      { "field": "order.deliveryInfo.email", "message": "Must be a valid email address" }
    ]
  }
}
```

- `code`: machine-readable constant in `SCREAMING_SNAKE_CASE`, following `{SUBJECT}_{CAUSE}` where cause is typically a past-tense verb or descriptive noun (e.g., `ORDER_NOT_FOUND`, `VALIDATION_FAILED`, `PAYMENT_DECLINED`, `VERSION_MISMATCH`). Be specific to the domain — avoid generic HTTP status names.
- `message`: human-readable explanation
- `details`: optional array for field-level errors (e.g., `VALIDATION_FAILED`). Omit when `code` + `message` are sufficient (e.g., `ORDER_NOT_FOUND`).

### Partial responses

`GET /api/v1/orders?fields[orders]=id,status&fields[items]=id,quantity&include=items`

- `fields[type]=field1,field2` trims columns on an already-fetched resource - if not specified for a type, return all fields by default
- `include` controls which related resources are joined and embedded inside each parent object

Notes:
- Use sparse fieldsets to let clients request only the fields they need, reducing payload size and over-fetching.
- Do not add partial response support by default. Only implement when there is an explicit need for it.
- Return `400` if the client requests a field name or `include` target that does not exist — the client is explicitly depending on that field, so failing fast prevents silent data contract bugs.


## Field Format Standards

| Type          | Format               | Example                               |
|---------------|----------------------|---------------------------------------|
| ID            | UUID v7              | `"id": "019074e0-2a5b-7d1c-9f3e-..."` |
| Date          | ISO 8601 in UTC      | `"2024-03-15"`                        |
| Date and Time | ISO 8601 in UTC      | `"2024-03-15T14:30:00.000Z"`          |
| Enum          | SCREAMING_SNAKE_CASE | `"status": "PENDING_REVIEW"`          |

- Ignore unknown fields during JSON body parsing — do not reject or fail. This allows clients and servers to evolve independently.
- Omit optional fields that have no value rather than including them as null, unless the client needs to distinguish 'not set' from 'explicitly cleared.'

## Query Parameters

- **Filtering**: `?status=active&role=admin`
- **Sorting**: `?sort=createdAt` (default order is `asc`), `?sort=status:desc,createdAt:asc` for multi-field with an explicit direction
- **Pagination**: `?cursor=abc123&limit=20`, `?offset=40&limit=20`, `?page=2&limit=20` (see Pagination)
- **Limiting output size**: `?limit=20` controls how many results are returned
- **Output shaping**: `?fields[orders]=id,status&include=items` (see Partial Responses)

Prefer flat, simple query params. Use `POST :search` when queries involve complex filter types (in, like, range), arrays of values, special characters, or the query string would exceed ~1000 characters. Use `fields[]` bracket notation only when implementing sparse fieldsets (see Partial Responses).

## Pagination

| Style        | Query params (Request)  | Meta fields (Response)                               |
|--------------|-------------------------|------------------------------------------------------|
| Cursor-based | `?cursor=abc1&limit=20` | `cursor` (current), `nextCursor`, `limit`, `hasMore` |
| Offset-based | `?offset=40&limit=20`   | `offset`, `limit`, `total`                           |
| Page-based   | `?page=2&limit=20`      | `page`, `limit`, `total`, `totalPages`               |

```json lines
// Cursor-based meta
{ "meta": { "cursor": "abc1", "nextCursor": "abc2", "limit": 20, "hasMore": true } }
// Offset-based meta
{ "meta": { "offset": 40, "limit": 20, "total": 280 } }
// Page-based meta
{ "meta": { "page": 2, "limit": 20, "total": 280, "totalPages": 14 } }
```

### Choosing a Pagination Style

| Situation                      | Recommended | Reason                                               |
|--------------------------------|-------------|------------------------------------------------------|
| Infinite scroll or feed        | Cursor      | New items shift offsets causing skips/duplicates     |
| Large dataset (> 100k rows)    | Cursor      | Offset scans degrade at depth; keyset stays constant |
| Frequently updated data        | Cursor      | Inserts/deletes shift offsets unpredictably          |
| Need to jump to arbitrary page | Offset      | Cursor-based only supports sequential traversal      |
| Page must stay consistent      | Page        | Users navigate by page number; total count matters   |

**Priority**: Cursor > Offset > Page
- Cursor-based - The only style that stays performant at scale and handles data mutations correctly. Offset-based is hard to retrofit to cursor later.
- Offset-based - More flexible than page-based for jumping around but has performance issues at high offsets and with data mutations.
- Page-based - Suitable when users navigate by page number and the dataset is bounded.

### DB implications

**Cursor-based** uses keyset pagination — `WHERE id > :last_seen_id ORDER BY id LIMIT n`:
- Performance is constant regardless of depth — the query hits an index seek, not a full scan
- Requires a stable, indexed sort column; does not support jumping to an arbitrary position
- Works with most SQL and NoSQL DBs (via range queries on an indexed field)

**Offset-based** and **Page-based** both translate to `LIMIT n OFFSET x` at the database level:
- Performance degrades at high offsets — the DB must scan and discard all preceding rows before returning results


## Authentication

| Mechanism    | Header                                  | Protocol / Token | When to use                                                                                    |
|--------------|-----------------------------------------|------------------|------------------------------------------------------------------------------------------------|
| Bearer token | `Authorization: Bearer <token>`         | OAuth 2.0 + JWT  | When you have an IdP (user-facing or service-to-service with OAuth)                            |
| API key      | `X-API-Key: <key>`                      | Shared secret    | When you don't have an IdP, or the caller can't implement OAuth                                |
| Signature    | `X-Signature: keyId=...,sig=...,ts=...` | HMAC-SHA256      | Inbound webhooks from third parties, or when you need tamper + replay protection on top of TLS |

- **JWT**: verify `iss`, `aud`, `exp`, and required scopes using the IdP's JWKS endpoint — no round-trip to the auth server needed.
- **OAuth 2.0 flows**: use Authorization Code + PKCE for user-facing apps, Client Credentials for service-to-service.
- **Scopes**: define per-operation (e.g., `orders:read`, `orders:write`)
- **Signature**: secret is never transmitted — the client signs method, path, timestamp, and body hash with HMAC-SHA256; server recomputes and compares. Reject requests with a timestamp older than 5 minutes to prevent replay attacks.

## Rate Limiting

| Response Header    | When returned  | Description                          | Example                |
|--------------------|----------------|--------------------------------------|------------------------|
| `RateLimit-Policy` | Every response | Declares quota policies              | `"default";q=100;w=60` |
| `RateLimit`        | Every response | Remaining quota and time until reset | `"default";r=50;t=30`  |
| `Retry-After`      | 429 only       | Seconds to wait before retrying      | `5`                    |

- `q` — quota (max requests), `w` — window in seconds, `r` — remaining requests, `t` — seconds until reset

| Rate limit by | When to use              | Default limit |
|---------------|--------------------------|---------------|
| IP Address    | Public / Unauthenticated | 20/min        |
| User ID       | Authenticated            | 100/min       |
| API Key       | Service-to-service       | 1000/min      |