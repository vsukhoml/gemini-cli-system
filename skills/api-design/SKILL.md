---
name: api-design
description: Design of the APIs for the back-end services, libraries
---

# API Design Skill


## Before you start

- Identify the core problem and list common algorithms, tools and established approaches.

- Conduct a real web search for recent advancements, state-of-the-art techniques, or existing libraries relevant to the problem.

- Evaluate solutions based on the problem's specific dimensions (e.g., data size, access patterns, latency requirements). If dimensions, scale is not clear - ask user.

- Do a web search for considered solutions to learn their trade-offs, applicability to the use case.


## Planning

- **Domain Knowledge:** Consider regulatory requirements for the domain. Cryptography, FIPS 140-3,  FedRAMP certifications, PCI DSS certifications, GDPR, HIPAA, privacy, requirements for federal, financial and healthcare cloud applications.  Back-end services and Information Security.

- **Optimizations:**  Consider user of load balancing, proxies, gateways, caching, sharding, mapping logical services to containers, VMs, physical machines.

- **Security and Safety:** Ensure that system follows Zero-Trust approach, provides means for authentication and authorization, monitoring, telemetry, auditing (if needed). Consider adding kill-switch to stop operations.

- **Redundancy and failover:** Design systems with ability to introduce redundant components to take over automatically if a primary component fails. 

- **Circuit breaker pattern:** Monitors a service for failures and, if a certain threshold is met, stops sending requests to that service to allow it to recover.

- **Stateless services:** Prefer designing services that do not retain session information between requests. This allows any instance of a service to handle a request, simplifying horizontal scaling and failover. 

- **Backups:** Consider how backups can be implemented.

- **Async Processing:** Asynchronous processing moves resource-intensive tasks, such as sending emails or generating reports, to background workers. This allows the system to handle new requests without waiting for the completion of these tasks. Consider use of queues for the sequence of long operations.

- **YAGNI** - Don't add functionality until deemed necessary.

- **KISS** - Simplicity shall be a design goal.

- **MVP** - Avoid lengthy and (possibly) unnecessary work. Iterate on working versions and respond to feedback.


## RESTful Design Principles

- Design a secure, efficient, and convenient API.

- Adhere to the principles outlined in Google's API Design Guide (google.aip.dev).

- Plan for **WE DO NOT BREAK USERSPACE** as the prime directive. Think carefully about how this API can evolve, and reflect it in the design, but remember YAGNII and KISS.

- **Minimize Versioning:** Design APIs carefully to avoid the need for versioning. If unavoidable, implement it (e.g., `vX` in URLs) but recognize the significant costs for both users and maintainers.

- **Product Value Over API Elegance** API success hinges almost entirely on the underlying product's value is a crucial business-oriented perspective. 

- **Idempotency:** A clear explanation of idempotency keys for safely retrying action-taking requests (POST, PATCH) is provided. 

- **Rate Limiting & Safety:** Emphasizing that APIs can be abused at \\"the speed of code\\" and the necessity of rate limits, kill-switches, and providing client metadata (`X-Limit-Remaining`, `Retry-After`) is vital for API stability and preventing incidents. This is a non-negotiable aspect of public API design.

- Validate that the content type is expected for each request or response.

- If using JWT, make sure that you perform robust signature verification on any JWTs that you receive, and account for edge-cases such as JWTs signed using unexpected algorithms.  Enforce a strict whitelist of permitted hosts for the jku header.  Make sure that you're not vulnerable to path traversal or SQL injection via the kid header parameter. Always set an expiration date for any tokens that you issue. Avoid sending tokens in URL parameters where possible. Include the aud (audience) claim (or similar) to specify the intended recipient of the token. This prevents it from being used on different websites. Enable the issuing server to revoke tokens (on logout, for example).

- Names and method descriptions matter far more than they ever have before.

- Automatic recovery is hugely important. Designing for error recovery rather than failing fast will make the whole system more reliable and less expensive to operate. When errors are unavoidable, your error messages should tell the user how to fix or work around the problem in plain English. However, keep in mind security trade-off and prevent leakage of information about system internals, especially for internal APIs.

- If you can't give the user exactly what they asked for, but you can give them a partial answer or related information, do that.



### 1. Resource Naming
```
✅ Good:
GET    /users          # List users
GET    /users/123      # Get user
POST   /users          # Create user
PUT    /users/123      # Update user
DELETE /users/123      # Delete user

❌ Bad:
GET /getUsers
POST /createUser
GET /user/delete/123
```

### 2. HTTP Status Codes
```typescript
// Success
200 OK           - Successful GET/PUT
201 Created      - Successful POST
204 No Content   - Successful DELETE

// Client Errors
400 Bad Request      - Invalid input
401 Unauthorized     - Auth required
403 Forbidden        - No permission
404 Not Found        - Resource doesn't exist
422 Unprocessable    - Validation failed

// Server Errors
500 Internal Error   - Server bug
503 Service Unavailable - Maintenance
```

### 3. Response Format
```typescript
// Success response
{
  "data": {
    "id": "123",
    "name": "John",
    "email": "john@example.com"
  }
}

// Error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      { "field": "email", "message": "This field is required" }
    ]
  }
}

// List with pagination
{
  "data": [...],
  "meta": {
    "page": 1,
    "perPage": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

### 4. Pagination
```typescript
// Offset-based
GET /posts?page=2&limit=20

// Cursor-based (better for large datasets)
GET /posts?cursor=abc123&limit=20
```

### 5. Filtering & Sorting
```typescript
// Filtering
GET /posts?status=published&author=123

// Sorting
GET /posts?sort=-createdAt,title  // - for DESC
```

## API Versioning
```typescript
// URL versioning (recommended)
GET /api/v1/users

// Header versioning
GET /api/users
Accept: application/vnd.myapp.v1+json
```

## Rate Limiting
```typescript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: { error: 'Too many requests' },
});

app.use('/api/', limiter);
```

## Input Validation
```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(150).optional(),
});

function createUser(req, res) {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({ error: result.error });
  }
  // Process validated data
}
```

### 6. Status Codes

#### Success Codes
- `200 OK` - Successful GET, PUT, PATCH, DELETE
- `201 Created` - Successful POST (resource created)
- `202 Accepted` - Request accepted, processing async
- `204 No Content` - Successful DELETE (no response body)

#### Client Error Codes
- `400 Bad Request` - Invalid request format
- `401 Unauthorized` - Missing or invalid authentication
- `403 Forbidden` - Authenticated but not authorized
- `404 Not Found` - Resource doesn't exist
- `405 Method Not Allowed` - HTTP method not supported
- `409 Conflict` - Resource conflict (duplicate)
- `422 Unprocessable Entity` - Validation errors
- `429 Too Many Requests` - Rate limit exceeded

#### Server Error Codes
- `500 Internal Server Error` - Server error
- `502 Bad Gateway` - Invalid response from upstream
- `503 Service Unavailable` - Temporary unavailable
- `504 Gateway Timeout` - Upstream timeout

### 7. Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      {
        "field": "email",
        "message": "Email must be valid format",
        "code": "INVALID_FORMAT"
      },
      {
        "field": "age",
        "message": "Age must be at least 18",
        "code": "MIN_VALUE"
      }
    ],
    "request_id": "abc123",
    "documentation_url": "https://api.example.com/docs/errors/validation"
  }
}
```

### 8. Authentication & Authorization

#### Authentication Methods
```
Bearer Token:
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

API Key:
X-API-Key: your-api-key-here

Basic Auth:
Authorization: Basic base64(username:password)
```

#### OAuth 2.0 Flow
```
1. GET /oauth/authorize
2. POST /oauth/token
3. Use access_token in requests
4. POST /oauth/refresh (when expired)
```

### 9. Versioning

#### URL Versioning (Recommended)
```
/v1/users
/v2/users
```

#### Header Versioning
```
Accept: application/vnd.api+json; version=1
API-Version: 1
```

### 10. Rate Limiting

#### Response Headers
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
Retry-After: 3600
```

#### Rate Limit Response
```json
HTTP/1.1 429 Too Many Requests

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "retry_after": 3600
  }
}
```

### 11. HATEOAS (Hypermedia)

```json
{
  "data": {
    "id": "123",
    "name": "John Doe",
    "status": "active"
  },
  "links": {
    "self": "/users/123",
    "edit": "/users/123",
    "delete": "/users/123",
    "posts": "/users/123/posts",
    "avatar": "/users/123/avatar"
  }
}
```

### 12. Caching

#### Cache Headers
```
Cache-Control: max-age=3600, public
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Last-Modified: Wed, 15 Jan 2024 10:30:00 GMT
```

#### Conditional Requests
```
GET /users/123
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

Response: 304 Not Modified (if not changed)
```

### 13. Bulk Operations

```
POST /users/bulk
Content-Type: application/json

{
  "operations": [
    { "method": "POST", "path": "/users", "body": {...} },
    { "method": "PATCH", "path": "/users/123", "body": {...} },
    { "method": "DELETE", "path": "/users/456" }
  ]
}
```

### 14. Webhooks

```json
POST /webhooks
{
  "url": "https://example.com/webhook",
  "events": ["user.created", "user.updated"],
  "secret": "webhook_secret_key"
}
```

### 15. API Documentation Format

Provide for each endpoint:
- HTTP Method & Path
- Description
- Authentication required
- Request parameters
- Request body schema
- Response schema
- Status codes
- Example requests/responses
- Rate limits
- Common errors

## Output Format

Provide:

1. **API Overview** - Purpose, base URL, authentication
2. **Resource Models** - Data structures and relationships
3. **Endpoint Specifications** - Complete endpoint documentation
4. **Authentication** - Auth flow and examples
5. **Error Handling** - Error codes and formats
6. **Rate Limiting** - Limits and headers
7. **Versioning Strategy** - How versions are managed
8. **Example Requests** - cURL examples for key operations
9. **OpenAPI/Swagger Spec** - If applicable

Generate complete, production-ready API documentation following REST best practices.
