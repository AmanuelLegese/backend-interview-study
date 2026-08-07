# Additional Study Materials

---

## 1. Coding Challenges

Practice these — you failed the coding challenge in the interview.

### Where to Practice
- [leetcode.com](https://leetcode.com) — Easy/Medium, arrays/strings/hash maps
- [hackerrank.com](https://hackerrank.com) — PHP-specific challenges
- [codewars.com](https://codewars.com) — PHP katas

### PHP Backend Challenges to Solve

```php
// CHALLENGE 1: Upsert Function
// Write a function that inserts a row or updates if it exists
function upsertUser(PDO $db, string $email, string $name): int
{
    $stmt = $db->prepare("
        INSERT INTO users (email, name) VALUES (?, ?)
        ON DUPLICATE KEY UPDATE name = VALUES(name)
    ");
    $stmt->execute([$email, $name]);
    return (int) $db->lastInsertId();
}
```

```php
// CHALLENGE 2: Rate Limiter (Sliding Window)
// Implement a sliding window rate limiter
class RateLimiter
{
    public function __construct(
        private Redis $redis
    ) {}

    public function isAllowed(string $key, int $limit, int $windowSeconds): bool
    {
        $now = microtime(true);
        $windowStart = $now - $windowSeconds;

        $pipe = $this->redis->multi();
        $pipe->zRemRangeByScore($key, 0, $windowStart);
        $pipe->zAdd($key, $now, (string) $now);
        $pipe->zCard($key);
        $pipe->expire($key, $windowSeconds);
        $results = $pipe->exec();

        return $results[2] <= $limit;
    }
}
```

```php
// CHALLENGE 3: Deduplicate Array
// Remove duplicate records based on email, keep latest
function deduplicateUsers(array $users): array
{
    $result = [];
    foreach ($users as $user) {
        $email = $user['email'];
        if (!isset($result[$email]) || $user['updated_at'] > $result[$email]['updated_at']) {
            $result[$email] = $user;
        }
    }
    return array_values($result);
}
```

```php
// CHALLENGE 4: Simple LRU Cache
class LRUCache
{
    private array $cache = [];
    private int $capacity;

    public function __construct(int $capacity)
    {
        $this->capacity = $capacity;
    }

    public function get(int $key): ?int
    {
        if (!isset($this->cache[$key])) {
            return null;
        }
        $value = $this->cache[$key];
        unset($this->cache[$key]);
        $this->cache[$key] = $value; // move to end (most recent)
        return $value;
    }

    public function put(int $key, int $value): void
    {
        if (isset($this->cache[$key])) {
            unset($this->cache[$key]);
        }
        $this->cache[$key] = $value;
        if (count($this->cache) > $this->capacity) {
            $oldest = array_key_first($this->cache);
            unset($this->cache[$oldest]);
        }
    }
}
```

```php
// CHALLENGE 5: JSON API Validator
function validateJsonApi(array $data, array $rules): array
{
    $errors = [];
    foreach ($rules as $field => $rule) {
        $value = $data[$field] ?? null;

        if ($rule['required'] && $value === null) {
            $errors[$field] = "{$field} is required";
            continue;
        }

        if ($value !== null) {
            if (isset($rule['type']) && gettype($value) !== $rule['type']) {
                $errors[$field] = "{$field} must be {$rule['type']}";
            }
            if (isset($rule['min']) && strlen((string) $value) < $rule['min']) {
                $errors[$field] = "{$field} must be at least {$rule['min']}";
            }
            if (isset($rule['max']) && strlen((string) $value) > $rule['max']) {
                $errors[$field] = "{$field} must be at most {$rule['max']}";
            }
        }
    }
    return $errors;
}
```

### Problem-Solving Process
1. Read requirements twice
2. Clarify questions before coding
3. Pseudocode / plan first
4. Talk through your thought process
5. Handle edge cases
6. Test your solution before submitting

---

## 2. Build a Mini Project

Build this to cover everything they criticized you on:

### Requirements
```
REST API — Blog Platform with Laravel

Features:
- User registration + Sanctum auth
- Posts CRUD (create, read, update, delete)
- Post publishing workflow (draft → published)
- Authorization via Policies (only author can edit/delete)
- Idempotency middleware for POST endpoints
- Rate limiting (60 requests/min per user)
- Pagination, filtering, sorting
- Proper error responses (RFC 7807)
- Database constraints (UNIQUE, FOREIGN KEY, CHECK)
- Feature tests
- Docker setup
- GitHub Actions CI
```

### Project Structure
```
blog-api/
├── app/
│   ├── Http/
│   │   ├── Controllers/Api/
│   │   │   ├── AuthController.php
│   │   │   └── PostController.php
│   │   ├── Middleware/
│   │   │   ├── IdempotencyMiddleware.php
│   │   │   └── EnsurePostAuthor.php
│   │   ├── Requests/
│   │   │   ├── StorePostRequest.php
│   │   │   └── UpdatePostRequest.php
│   │   └── Resources/
│   │       └── PostResource.php
│   ├── Models/
│   │   ├── User.php
│   │   └── Post.php
│   └── Policies/
│       └── PostPolicy.php
├── database/
│   ├── migrations/
│   │   ├── create_users_table.php
│   │   ├── create_posts_table.php
│   │   └── create_idempotency_keys_table.php
│   └── seeders/
├── routes/
│   └── api.php
├── tests/
│   └── Feature/
│       ├── PostApiTest.php
│       └── AuthTest.php
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
└── .github/
    └── workflows/
        └── ci.yml
```

### Key Files to Implement

**PostPolicy:**
```php
class PostPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, Post $post): bool
    {
        if ($post->status === 'published') {
            return true;
        }
        return $user->id === $post->author_id;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->author_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->author_id || $user->is_admin;
    }

    public function publish(User $user, Post $post): bool
    {
        return $user->id === $post->author_id && $post->status === 'draft';
    }
}
```

**IdempotencyMiddleware:**
```php
class IdempotencyMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $key = $request->header('Idempotency-Key');

        if (!$key || !in_array($request->method(), ['POST', 'PUT', 'PATCH'])) {
            return $next($request);
        }

        $cached = Cache::get("idempotency:{$key}");
        if ($cached) {
            return response()->json($cached['body'], $cached['status']);
        }

        $response = $next($request);

        Cache::put("idempotency:{$key}", [
            'body' => json_decode($response->getContent(), true),
            'status' => $response->getStatusCode(),
        ], 3600);

        return $response;
    }
}
```

**Database Migration:**
```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('author_id')->constrained('users')->cascadeOnDelete();
    $table->string('title');
    $table->text('content');
    $table->enum('status', ['draft', 'published', 'archived'])->default('draft');
    $table->timestamp('published_at')->nullable();
    $table->timestamps();

    $table->index('status');
    $table->index('author_id');
});
```

**Feature Test:**
```php
class PostApiTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_create_post(): void
    {
        $user = User::factory()->create();
        $response = $this->actingAs($user)
            ->postJson('/api/posts', [
                'title' => 'My Post',
                'content' => 'Content here',
            ]);

        $response->assertStatus(201)
                 ->assertJsonFragment(['title' => 'My Post']);
        $this->assertDatabaseHas('posts', ['title' => 'My Post']);
    }

    public function test_user_cannot_update_other_users_post(): void
    {
        $user = User::factory()->create();
        $otherUser = User::factory()->create();
        $post = Post::factory()->create(['author_id' => $otherUser->id]);

        $response = $this->actingAs($user)
            ->putJson("/api/posts/{$post->id}", ['title' => 'Hacked']);

        $response->assertStatus(403);
    }

    public function test_idempotency_key_returns_same_response(): void
    {
        $user = User::factory()->create();
        $key = Str::uuid();

        $response1 = $this->actingAs($user)
            ->postJson('/api/posts', ['title' => 'Test', 'content' => 'Test'], [
                'Idempotency-Key' => $key,
            ]);

        $response2 = $this->actingAs($user)
            ->postJson('/api/posts', ['title' => 'Test', 'content' => 'Test'], [
                'Idempotency-Key' => $key,
            ]);

        $this->assertEquals(
            $response1->json('data.id'),
            $response2->json('data.id')
        );
    }
}
```

---

## 3. Explain Clearly — FORMAT

Use this format for every answer:

```
1. WHAT    — one sentence definition
2. HOW     — concrete mechanism
3. WHEN    — real-world use case
4. TRADE-OFF — why not always use it
```

### Example: Idempotency Keys

> **What:** Idempotency keys ensure a request is processed only once, even if the client retries.
>
> **How:** The client generates a UUID and sends it as an `Idempotency-Key` header. The server checks its cache or database. If the key exists, it returns the cached response. If not, it processes the request and stores the result with that key.
>
> **When:** Payment APIs, order creation, any operation where duplicates cause real problems (double charges, duplicate orders).
>
> **Trade-off:** Adds complexity and requires storage (Redis/DB). Not needed for GET (read-only) or DELETE (naturally idempotent). Only critical for state-changing operations.

### Example: Connection Pooling

> **What:** Connection pooling maintains a set of reusable database connections instead of opening/closing one per request.
>
> **How:** A pool holds N connections. When a worker needs a DB connection, it borrows one from the pool, uses it, and returns it. The connection stays alive for the next request.
>
> **When:** High-traffic applications where opening connections (100-500ms each) would be too slow. Especially important with PHP-FPM where each worker needs its own connection.
>
> **Trade-off:** Connections consume memory on the DB server. You need to balance pool size against DB `max_connections` limit. Persistent connections in PHP can leak state between requests if not careful.

---

## 4. Common Laravel Interview Questions

Practice answering these cold:

| Question | Answer Key Points |
|----------|-------------------|
| How does middleware work? | Request passes through a pipeline. Each middleware can modify request/response. Defined in Kernel, applied to routes/groups. |
| Difference between `lazy()` and `chunk()`? | `chunk()` loads N records into memory. `lazy()` returns a generator, one record at a time. `lazy()` is more memory-efficient. |
| How to prevent N+1 queries? | Eager loading with `with()`. Use `withCount()` for counts. Use `select()` to limit columns loaded. |
| What does `php artisan queue:work` do? | Starts a worker process that polls the queue. Pulls jobs, processes them, retries on failure. Runs until killed or max jobs reached. |
| How does route model binding work? | Laravel auto-injects model instances based on route parameter name. Implicit: `{post}` → `Post`. Explicit: bind `Post`. Returns 404 if not found. |
| Difference between `artisan serve` and PHP-FPM? | `artisan serve` is a dev server (single process, no optimization). PHP-FPM is production (process manager, OPcache, handles concurrent requests). |
| What is the service container? | IoC container that manages class dependencies. Resolves dependencies automatically. Used for binding, singleton, contextual binding. |
| Explain Eloquent relationships | One-to-one (hasOne), one-to-many (hasMany), many-to-many (belongsToMany), polymorphic (morphMany). Eager loading prevents N+1. |

---

## 5. System Design Questions

### Design a URL Shortener
```
Components:
- API endpoint: POST /shorten (original URL → short URL)
- Redirect: GET /{code} → 301/302 to original URL
- Analytics: track clicks

Database:
- urls table: id, short_code (UNIQUE), original_url, clicks_count, created_at
- Index on short_code for fast lookups

Short code generation:
- Base62 encoding of auto-increment ID
- Or hash (MD5/SHA256) truncated
- Collision check via UNIQUE constraint

Scaling:
- Redis cache for hot URLs
- Read replicas for analytics
- Rate limiting on create endpoint
```

### Design a Rate Limiter
```
Algorithm: Token Bucket
- Bucket has N tokens
- Each request consumes 1 token
- Tokens refill at R per second
- Return 429 when empty

Storage: Redis
- Key: rate:{endpoint}:{user_id}
- Value: {tokens, last_refill}

Implementation:
- Check tokens > 0
- If yes: decrement, allow
- If no: calculate wait time, return 429 with Retry-After header

Distributed:
- Redis atomic operations (DECR, INCRBY)
- Sliding window: sorted set with timestamps
```

### Design a Notification System
```
Channels: Email, SMS, Push, In-app

Components:
- NotificationService → dispatches to channel handlers
- Queue system for async processing
- Retry logic with exponential backoff
- Template system for messages

Database:
- notifications: id, user_id, channel, type, data, status, sent_at
- notification_preferences: user_id, channel, enabled

Flow:
1. App calls NotificationService::send(user, type, data)
2. Service creates notification record
3. Dispatches job to appropriate queue
4. Channel handler sends (Mailgun, Twilio, Firebase)
5. Update status (sent/failed)
6. Retry failed notifications
```

### Design a File Upload System
```
Components:
- Pre-signed URLs for direct S3 upload
- Processing queue (thumbnails, transcoding)
- Metadata storage in DB

Flow:
1. Client requests pre-signed URL from API
2. Client uploads directly to S3 (no server load)
3. S3 event triggers Lambda/queue job
4. Processing: resize images, transcode video
5. Update DB with processed file URLs

Database:
- files: id, user_id, original_name, s3_key, mime_type, size, status

Security:
- Pre-signed URLs expire in 5 minutes
- File size limits
- MIME type validation
- Virus scanning
```

---

## 6. Study Schedule

> **Your schedule:** 8-hour job, study at night
> **Available time:** 2-3 hours weekday nights, 4-6 hours weekends
> **Total:** ~18-20 hours/week

### Daily Routine

```
WEEKDAYS (Mon-Fri)
├── 9:00 PM - 10:00 PM  →  Read docs (theory)
├── 10:00 PM - 11:00 PM →  Practice (coding/explain out loud)
└── 11:00 PM            →  Rest

WEEKENDS (Sat-Sun)
├── Morning: Rest / light review
├── Afternoon (2-4 hrs) →  Deep work (build project, coding challenges)
└── Evening (1-2 hrs)   →  Review + plan next day
```

### WEEK 1 — Foundations

```
Mon night  → PHP Internals (read 01-php-internals.md)
Tue night  → PHP Internals continued (FPM, comparison)
Wed night  → Laravel (read 02-laravel.md — Policies vs Gates)
Thu night  → Laravel continued (middleware, Eloquent)
Fri night  → Coding challenge (1 problem on LeetCode)
Sat        → Database Design (read 03-database-design.md)
Sun        → Database continued (NULL, upserts, transactions)
```

### WEEK 2 — API & Scalability

```
Mon night  → API Design (read 04-api-design.md — REST, versioning)
Tue night  → API Design continued (idempotency keys, rate limiting)
Wed night  → Scalability (read 05-scalability.md — connection pooling)
Thu night  → Scalability continued (caching, queues)
Fri night  → Coding challenge (1 problem)
Sat        → DevOps (read 06-devops.md — Docker, CI/CD)
Sun        → Mock interview practice (read 07-mock-interview.md)
```

### WEEK 3 — Build Project

```
Mon night  → Set up Laravel + Docker + Sanctum auth
Tue night  → Posts CRUD + migrations
Wed night  → Policies + Idempotency middleware
Thu night  → Rate limiting + error responses
Fri night  → Feature tests
Sat        → CI/CD + polish + code review
Sun        → Push to GitHub + document
```

### WEEK 4 — Interview Prep

```
Mon night  → Mock interview topics (explain 3 topics out loud)
Tue night  → Mock interview topics (explain 3 more)
Wed night  → System design practice (1 question)
Thu night  → Weak areas review (re-read feedback files)
Fri night  → Coding challenges (2 problems)
Sat        → Full mock interview (explain all 10 topics)
Sun        → Light review + rest before interview
```

### Progress Tracker

| Week | Mon | Tue | Wed | Thu | Fri | Sat | Sun |
|------|-----|-----|-----|-----|-----|-----|-----|
| 1 | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| 2 | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| 3 | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| 4 | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |

---

## 7. Resources

| Resource | Link | Use For |
|----------|------|---------|
| Laravel Docs | laravel.com/docs | Framework reference |
| PHP The Right Way | phptherightway.com | PHP best practices |
| Refactoring Guru | refactoring.guru | Design patterns |
| LeetCode | leetcode.com | Coding challenges |
| System Design Interview | Alex Xu book | System design |
| Symfony Mailer | symfony.com/doc/current/mailer | Email handling |
| PostgreSQL Wiki | postgresql.org/wiki | SQL deep dives |
