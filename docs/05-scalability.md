# Scalability & High Traffic

## 1. Connection Pooling (Interview Focus)

### What is Connection Pooling?

Instead of opening/closing a DB connection for every request, a pool of reusable connections is maintained. This reduces connection overhead significantly.

```
Without Pooling:
  Request 1 → Open → Use → Close
  Request 2 → Open → Use → Close
  Request 3 → Open → Use → Close
  (3 opens, 3 closes)

With Pooling:
  Request 1 → Get from pool → Use → Return to pool
  Request 2 → Get from pool → Use → Return to pool
  Request 3 → Get from pool → Use → Return to pool
  (0 opens, 0 closes — connections reused!)
```

### Why It Matters

- Opening a DB connection takes ~100-500ms
- MySQL has default `max_connections` (usually 151)
- Under high traffic, opening/closing exhausts connections
- Connection pooling reuses connections across requests

### Laravel Configuration

```php
// config/database.php
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE'),
    'username' => env('DB_USERNAME'),
    'password' => env('DB_PASSWORD'),
    'options' => [
        PDO::ATTR_PERSISTENT => true,  // persistent connections
        PDO::ATTR_EMULATE_PREPARES => false,
    ],
],
```

### PgBouncer (PostgreSQL)

```ini
; pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction  ; release connection after transaction
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
```

### PHP-FPM Connection Management

```ini
; Each worker holds its own connection
; If you have 50 workers, you need 50 DB connections max
pm.max_children = 50

; Reduce with persistent connections
; Each worker reuses the same connection across requests
pm.max_requests = 500  ; recycle worker after 500 requests
```

---

## 2. Caching

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data types | Strings, lists, sets, hashes, sorted sets | Strings only |
| Persistence | Yes (RDB, AOF) | No |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Memory efficiency | Moderate | Very efficient |
| Use case | Complex caching, queues, sessions | Simple key-value caching |

### Cache Strategies

**Cache-Aside (Lazy Loading):**
```php
function getUser(int $id): User
{
    $key = "user:{$id}";

    // 1. Try cache
    $cached = Cache::get($key);
    if ($cached) {
        return $cached;
    }

    // 2. Cache miss — query DB
    $user = User::find($id);

    // 3. Store in cache
    Cache::put($key, $user, 3600);  // 1 hour

    return $user;
}

// Laravel helper
$user = Cache::remember("user:{$id}", 3600, function () use ($id) {
    return User::find($id);
});
```

**Write-Through:**
```php
// Write to cache AND DB simultaneously
$user = User::create($data);
Cache::put("user:{$user->id}", $user, 3600);

// On update
$user->update($data);
Cache::put("user:{$user->id}", $user, 3600);

// On delete
$user->delete();
Cache::forget("user:{$user->id}");
```

**Write-Behind (Write-Back):**
```php
// Write to cache immediately, DB asynchronously
Cache::put("user:{$id}", $data, 3600);

// Queue DB write
UpdateUserDB::dispatch($id, $data);
```

### Cache Invalidation

```php
// Tag-based invalidation
Cache::tags(['posts', 'user:1'])->put($key, $value, 3600);

// When user's posts change, invalidate all their post caches
Cache::tags(['user:1', 'posts'])->flush();

// Cache stampede prevention (lock)
$value = Cache::lock("user:{$id}", 10)->block(5, function () use ($id) {
    return Cache::remember("user:{$id}", 3600, function () use ($id) {
        return User::find($id);
    });
});
```

### HTTP Cache Headers

```php
// Laravel response caching
return response($data)
    ->header('Cache-Control', 'public, max-age=3600')
    ->header('ETag', md5($content))
    ->header('Last-Modified', $lastModified->format('D, d M Y H:i:s'));

// Conditional request (304 Not Modified)
if ($request->header('If-None-Match') === $etag) {
    return response('', 304);
}
```

---

## 3. Queue Systems

### Why Async Processing?

- **Slow operations:** email, file processing, payments
- **User experience:** return immediately, process in background
- **Load leveling:** absorb traffic spikes
- **Retries:** failed jobs can be retried automatically

### Laravel Queue Drivers

```php
// Sync (for testing)
QUEUE_CONNECTION=sync

// Database
QUEUE_CONNECTION=database
php artisan queue:table
php artisan migrate

// Redis
QUEUE_CONNECTION=redis

// SQS (AWS)
QUEUE_CONNECTION=sqs
```

### Job Configuration

```php
class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 30;
    public int $maxExceptions = 2;
    public string $queue = 'payments';

    public function backoff(): array
    {
        return [30, 60, 120]; // seconds between retries
    }

    public function middleware(): array
    {
        return [
            new WithoutOverlapping('payment'),
            new ThrottlesExceptions(10, 60), // 10 exceptions per 60 seconds
        ];
    }

    public function handle(PaymentGateway $gateway): void
    {
        $gateway->charge($this->amount, $this->cardToken);
    }

    public function failed(Throwable $e): void
    {
        // Notify user, log error, etc.
    }
}
```

### Job Prioritization

```php
// Use multiple queues
Bus::batch([
    new HighPriorityTask(),
    new LowPriorityTask(),
])->dispatch();

// Worker priority
// php artisan queue:work --queue=high,default,low
// High queue processed first
```

---

## 4. Horizontal Scaling

### Stateless Application Design

```
Stateless:
  User → Load Balancer → Server 1 ✓
  User → Load Balancer → Server 2 ✓
  (Any server can handle any request)

Stateful (bad):
  User → Server 1 (has session) ✓
  User → Server 2 (no session) ✗
```

**Make it stateless:**
- Store sessions in Redis/DB, not file
- Store files in S3, not local disk
- Use JWT tokens, not server-side sessions
- No local file dependencies

### Load Balancing Algorithms

| Algorithm | Description |
|-----------|-------------|
| Round Robin | Requests distributed evenly |
| Least Connections | Route to server with fewest connections |
| IP Hash | Same client always hits same server |
| Weighted | More powerful servers get more traffic |

### Session Storage

```php
// config/session.php
'driver' => 'redis',  // or 'database', 'memcached'
'connection' => 'default',
'table' => 'sessions',
'lifetime' => 120,
'encrypt' => true,
```

---

## 5. Database Scaling

### Read Replicas

```
Application
    │
    ├── Write ──► Primary DB
    │
    └── Read ───► Replica 1
                ► Replica 2
                ► Replica 3
```

```php
// Laravel
DB::connection('replica1')->table('posts')->get();

// Route specific operations
'connections' => [
    'mysql' => [
        'read' => [
            'host' => ['replica1', 'replica2'],
        ],
        'write' => [
            'host' => 'primary',
        ],
    ],
],
```

### Sharding

Split data across multiple databases based on a shard key.

```php
// Simple sharding example
function getShard(int $userId): string
{
    $shard = $userId % 4;  // 4 shards
    return "database_shard_{$shard}";
}

// Use specific connection
DB::connection(getShard($userId))->table('users')->find($userId);
```

### Query Optimization

```php
// N+1 problem
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name; // N queries!
}

// Fix: eager loading
$posts = Post::with('author')->get(); // 2 queries

// Use select to limit columns
$posts = Post::with('author:id,name')->select('id', 'title', 'author_id')->get();

// Use cursor for large datasets
foreach (Post::cursor() as $post) {
    // Process one at a time, low memory
}

// Chunking
Post::chunk(100, function ($posts) {
    foreach ($posts as $post) {
        // Process in batches
    }
});
```

### Slow Query Log

```sql
-- MySQL
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- queries > 1 second

-- PostgreSQL
log_min_duration_statement = 1000  -- milliseconds
```

---

## 6. De-duplication Strategies

### Request De-duplication

```php
// Use idempotency keys + unique constraint
Schema::create('idempotency_keys', function (Blueprint $table) {
    $table->string('key')->primary();
    $table->json('response');
    $table->integer('status_code');
    $table->timestamp('expires_at');
});

// Atomic check + insert
DB::beginTransaction();
$existing = DB::table('idempotency_keys')
    ->where('key', $key)
    ->where('expires_at', '>', now())
    ->first();

if ($existing) {
    DB::rollBack();
    return response()->json(json_decode($existing->response));
}

// Process request...
DB::table('idempotency_keys')->insert([...]);
DB::commit();
```

### Event De-duplication

```php
// Use unique request IDs
class ProcessEventJob implements ShouldQueue
{
    public function __construct(
        public string $requestId,
        public array $data
    ) {}

    public function handle(): void
    {
        DB::beginTransaction();
        try {
            $exists = DB::table('processed_events')
                ->where('request_id', $this->requestId)
                ->exists();

            if ($exists) {
                DB::rollBack();
                return;
            }

            // Process...
            DB::table('processed_events')->insert([
                'request_id' => $this->requestId,
                'processed_at' => now(),
            ]);

            DB::commit();
        } catch (\Exception $e) {
            DB::rollBack();
            throw $e;
        }
    }
}
```
