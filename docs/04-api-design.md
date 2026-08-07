# API Design & Architecture

## 1. RESTful Design

### HTTP Methods & Status Codes

| Method | Purpose | Status Codes |
|--------|---------|--------------|
| GET | Read resource | 200 OK, 404 Not Found |
| POST | Create resource | 201 Created, 422 Unprocessable |
| PUT | Replace resource | 200 OK, 204 No Content |
| PATCH | Update resource partially | 200 OK, 204 No Content |
| DELETE | Delete resource | 204 No Content, 404 Not Found |

### Resource Naming

```
GET    /api/v1/users              # list users
POST   /api/v1/users              # create user
GET    /api/v1/users/123          # get user 123
PUT    /api/v1/users/123          # replace user 123
PATCH  /api/v1/users/123          # update user 123
DELETE /api/v1/users/123          # delete user 123

# Nested resources
GET    /api/v1/users/123/posts    # user's posts
POST   /api/v1/users/123/posts    # create post for user

# Actions (non-CRUD)
POST   /api/v1/posts/123/publish  # publish a post
POST   /api/v1/posts/123/archive  # archive a post
```

### Pagination

```json
// Offset-based
GET /api/v1/posts?page=3&per_page=20

{
  "data": [...],
  "meta": {
    "current_page": 3,
    "per_page": 20,
    "total": 150,
    "last_page": 8
  },
  "links": {
    "first": "/api/v1/posts?page=1&per_page=20",
    "prev": "/api/v1/posts?page=2&per_page=20",
    "next": "/api/v1/posts?page=4&per_page=20",
    "last": "/api/v1/posts?page=8&per_page=20"
  }
}

// Cursor-based (better for real-time)
GET /api/v1/posts?cursor=eyJpZCI6MTIzfQ&per_page=20

{
  "data": [...],
  "meta": {
    "per_page": 20,
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_more": true
  }
}
```

### Filtering & Sorting

```
# Filtering
GET /api/v1/posts?status=published&author_id=5
GET /api/v1/posts?created_after=2024-01-01

# Sorting
GET /api/v1/posts?sort=-created_at,title
# - = descending, no prefix = ascending

# Field selection
GET /api/v1/posts?fields=id,title,created_at
```

### Laravel Example

```php
// routes/api.php
Route::apiResource('posts', PostController::class);
Route::post('posts/{post}/publish', [PostController::class, 'publish']);

// app/Http/Controllers/Api/PostController.php
class PostController extends Controller
{
    public function index(Request $request)
    {
        $posts = Post::query()
            ->when($request->status, fn($q, $s) => $q->where('status', $s))
            ->when($request->author_id, fn($q, $a) => $q->where('author_id', $a))
            ->orderBy($request->get('sort', 'created_at'), $request->get('order', 'desc'))
            ->paginate($request->get('per_page', 20));

        return PostResource::collection($posts);
    }

    public function store(StorePostRequest $request)
    {
        $post = Post::create($request->validated());
        return new PostResource($post);
    }
}
```

---

## 2. API Versioning

### URI Versioning (Most Common)

```php
// routes/api.php
Route::prefix('v1')->group(function () {
    Route::apiResource('posts', V1\PostController::class);
});

Route::prefix('v2')->group(function () {
    Route::apiResource('posts', V2\PostController::class);
});
```

### Header Versioning

```php
// Middleware
class ApiVersionMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $version = $request->header('Accept-Version', 'v1');
        $request->attributes->set('api_version', $version);
        return $next($request);
    }
}
```

### Deprecation Strategy

```php
// 1. Add deprecation header
Route::get('/v1/posts', function () {
    return response()->json($data)
        ->header('Sunset', 'Sat, 01 Jan 2027 00:00:00 GMT')
        ->header('Deprecation', 'true')
        ->header('Link', '</api/v2/posts>; rel="successor-version"');
});

// 2. Phase 1: Both versions work
// 3. Phase 2: Log usage of old version, notify clients
// 4. Phase 3: Return 410 Gone on old version after sunset
```

---

## 3. Idempotency (Interview Focus)

### What is Idempotency?

An operation is idempotent if calling it multiple times produces the same result as calling it once. Critical for retry-safe APIs.

### How Idempotency Keys Work

```
Client                          Server
  │                               │
  │── POST /api/payments ────────►│
  │   Idempotency-Key: abc-123   │
  │                               │
  │                               │ 1. Check if key exists
  │                               │ 2. If no: process, store result
  │                               │ 3. If yes: return cached result
  │                               │
  │◄── 201 Created ──────────────│
  │   { id: 456, status: "ok" }  │
  │                               │
  │── POST /api/payments ────────►│  (retry)
  │   Idempotency-Key: abc-123   │
  │                               │
  │◄── 200 OK ───────────────────│  (same response)
  │   { id: 456, status: "ok" }  │
```

### Server Implementation

```php
// app/Http/Middleware/IdempotencyMiddleware.php
class IdempotencyMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $key = $request->header('Idempotency-Key');

        if (!$key) {
            return $next($request);
        }

        // Check cache/DB for existing key
        $cacheKey = "idempotency:{$key}";
        $cached = Cache::get($cacheKey);

        if ($cached) {
            // Return cached response
            return response()->json($cached['body'], $cached['status']);
        }

        $response = $next($request);

        // Store response for idempotency
        Cache::put($cacheKey, [
            'body' => json_decode($response->getContent(), true),
            'status' => $response->getStatusCode(),
        ], 3600); // expire after 1 hour

        return $response;
    }
}

// Register in Kernel
protected $middlewareGroups = [
    'api' => [
        \App\Http\Middleware\IdempotencyMiddleware::class,
    ],
];
```

### Database Storage Approach

```sql
CREATE TABLE idempotency_keys (
    key VARCHAR(255) PRIMARY KEY,
    response JSON NOT NULL,
    status_code INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP
);
```

```php
// More robust: atomic DB operation
DB::beginTransaction();
try {
    $existing = DB::table('idempotency_keys')
        ->where('key', $key)
        ->where('expires_at', '>', now())
        ->first();

    if ($existing) {
        DB::rollBack();
        return response()->json(json_decode($existing->response), $existing->status_code);
    }

    // Process request...

    DB::table('idempotency_keys')->insert([
        'key' => $key,
        'response' => json_encode($responseData),
        'status_code' => $statusCode,
        'expires_at' => now()->addHour(),
    ]);

    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    throw $e;
}
```

### Client-Side Usage

```javascript
// Generate UUID for each logical operation
const idempotencyKey = crypto.randomUUID();

fetch('/api/payments', {
    method: 'POST',
    headers: {
        'Idempotency-Key': idempotencyKey,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({ amount: 100, currency: 'USD' }),
});

// Safe to retry with SAME key
fetch('/api/payments', {
    method: 'POST',
    headers: {
        'Idempotency-Key': idempotencyKey,  // same key!
    },
    body: JSON.stringify({ amount: 100, currency: 'USD' }),
});
```

### Which Methods Need Idempotency?

| Method | Idempotent by Default? | Needs Key? |
|--------|----------------------|------------|
| GET | Yes | No |
| PUT | Yes (if body same) | Recommended |
| DELETE | Yes | Recommended |
| POST | No | Yes |
| PATCH | No | Yes |

---

## 4. Rate Limiting

### Strategies

**Token Bucket:**
```
Bucket has 10 tokens
Each request consumes 1 token
Tokens refill at 1 per second
If bucket empty → 429 Too Many Requests
```

**Fixed Window:**
```
Max 100 requests per minute
Window: 12:00:00 - 12:00:59
Counter resets at 12:01:00
```

**Sliding Window:**
```
Max 100 requests per 60 seconds
Window slides with each request
More evenly distributed than fixed window
```

### Laravel Implementation

```php
// routes/api.php
Route::middleware('throttle:60,1')->group(function () {
    Route::get('/api/posts', [PostController::class, 'index']);
});

// Custom throttle
Route::middleware('throttle:api')->group(function () {
    // config/throttle.php
});

// Per-user rate limiting
Route::middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::post('/api/payments', [PaymentController::class, 'store']);
});
```

### Rate Limit Response

```json
HTTP/1.1 429 Too Many Requests

{
    "message": "Too many requests, please try again later.",
    "retry_after": 30
}

Retry-After: 30
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704067260
```

---

## 5. Authentication

### JWT (JSON Web Token)

```
Header.Payload.Signature
eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiIuLi4iLCJleHAiOjE3MDQwNjcyNjB9.signature
```

**Flow:**
```
1. User logs in with credentials
2. Server validates, generates JWT
3. Client stores JWT (localStorage/cookie)
4. Client sends JWT in Authorization header
5. Server validates JWT on each request
6. Token expires after X hours
7. Client uses refresh token to get new JWT
```

**Laravel Implementation:**

```php
// Login
public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => 'required|email',
        'password' => 'required',
    ]);

    if (!Auth::attempt($credentials)) {
        return response()->json(['error' => 'Invalid credentials'], 401);
    }

    $user = Auth::user();
    $token = $user->createToken('auth-token')->plainTextToken;

    return response()->json([
        'user' => $user,
        'token' => $token,
    ]);
}

// Protected route
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/api/user', fn(Request $r) => $r->user());
});
```

### OAuth 2.0 Flows

| Flow | Use Case |
|------|----------|
| Authorization Code | Web apps, mobile apps (with PKCE) |
| Client Credentials | Server-to-server (no user) |
| Password Grant | Trusted first-party apps (deprecated) |

### API Tokens (Sanctum)

```php
// Generate token
$token = $user->createToken('mobile-app')->plainTextToken;

// Use in request
Authorization: Bearer {token}

// Revoke
$user->tokens()->delete();
```

---

## 6. Consumer-Driven Contracts

### What is Contract Testing?

Two services agree on a contract (what the API expects/returns). Consumer defines expectations, provider verifies it meets them.

### Pact Framework

```php
// Consumer test (defines contract)
$consumer->expects('a list of posts')
    ->uponReceiving('a request for posts')
    ->with([
        'method' => 'GET',
        'path' => '/api/posts',
        'headers' => ['Accept' => 'application/json'],
    ])
    ->willRespondWith([
        'status' => 200,
        'headers' => ['Content-Type' => 'application/json'],
        'body' => [
            'data' => [
                [
                    'id' => Matchers::type('integer'),
                    'title' => Matchers::like('A post title'),
                    'status' => Matchers::term('published|draft', 'published'),
                ],
            ],
        ],
    ]);

// Provider test (verifies contract)
$provider->given('there are posts')
    ->uponReceiving('a request for posts')
    ->with([
        'method' => 'GET',
        'path' => '/api/posts',
    ])
    ->willRespondWith([
        'status' => 200,
        'body' => /* actual response */,
    ]);
```

### Benefits
- Catch breaking changes before deployment
- Document API behavior
- Enable independent deployment of services

---

## 7. Error Handling

### RFC 7807 Problem Details

```json
// 422 Unprocessable Entity
{
    "type": "https://api.example.com/errors/validation",
    "title": "Validation Error",
    "status": 422,
    "detail": "The request body contains invalid fields",
    "instance": "/api/users/123",
    "errors": {
        "email": ["The email must be a valid email address"],
        "name": ["The name is required"]
    }
}

// 409 Conflict
{
    "type": "https://api.example.com/errors/conflict",
    "title": "Resource Conflict",
    "status": 409,
    "detail": "A user with this email already exists"
}
```

### Laravel Error Response

```php
// app/Exceptions/Handler.php
use Symfony\Component\HttpKernel\Exception\HttpException;

public function register(): void
{
    $this->renderable(function (HttpException $e, $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'type' => 'error',
                'title' => $e->getMessage(),
                'status' => $e->getStatusCode(),
            ], $e->getStatusCode());
        }
    });
}
```
