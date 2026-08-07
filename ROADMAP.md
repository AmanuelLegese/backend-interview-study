# Study Roadmap — PHP Backend Developer

Based on interview feedback, here's a complete roadmap to prepare for your next attempt.

---

## 1. PHP Internals & Fundamentals

### 1.1 PHP Lifecycle & FPM
- [ ] How PHP-FPM works (master/worker process model)
- [ ] `php-fpm.conf` key settings (`pm`, `pm.max_children`, `pm.start_servers`)
- [ ] Request lifecycle: init → parse → execute → shutdown
- [ ] OPcache and how it speeds up execution
- [ ] PHP 8.x new features (enums, fibers, readonly properties, named args)

### 1.2 Strict vs Loose Comparison
- [ ] `===` vs `==` with type juggling rules
- [ ] `0 == "foo"` edge cases
- [ ] `strcmp()` and `=== 0` for safe comparison

### 1.3 Pass by Value vs Pass by Reference
- [ ] `&` operator in function params
- [ ] `foreach` with `&$value` gotchas
- [ ] `unset()` behavior with references

### 1.4 Error Handling
- [ ] Exceptions vs Errors
- [ ] `try/catch/finally`
- [ ] Custom exception classes
- [ ] `set_error_handler()` vs `set_exception_handler()`

---

## 2. Laravel Framework

### 2.1 Policies vs Gates
- [ ] Gate: simple callback-based authorization (`Gate::define`)
- [ ] Policy: model-specific, class-based (`php artisan make:policy`)
- [ ] When to use which
- [ ] `@can` / `@cannot` Blade directives
- [ ] `Gate::before()` and `Gate::after()`
- [ ] Policy methods: `viewAny`, `view`, `create`, `update`, `delete`

### 2.2 Middleware
- [ ] How middleware works in request lifecycle
- [ ] Creating custom middleware
- [ ] Middleware groups and `Route::middleware()`
- [ ] `handle()` method signature

### 2.3 Controllers & Routing
- [ ] Resource controllers
- [ ] Route model binding
- [ ] Form Requests for validation

### 2.4 Eloquent ORM
- [ ] Relationships: one-to-one, one-to-many, many-to-many, polymorphic
- [ ] Eager loading (`with()`) vs lazy loading
- [ ] Query scopes (local and global)
- [ ] Accessors and mutators
- [ ] `whereHas()`, `whereColumn()`

### 2.5 Queues & Jobs
- [ ] Sync, database, Redis, SQS drivers
- [ ] `dispatch()`, `dispatchSync()`, `Bus::batch()`
- [ ] Job chaining and middleware
- [ ] Failed jobs handling

### 2.6 Events & Listeners
- [ ] Event/listener registration
- [ ] Event broadcasting (WebSockets)
- [ ] Queued event listeners

---

## 3. Database Design & SQL

### 3.1 Normalization
- [ ] 1NF: atomic values, no repeating groups
- [ ] 2NF: no partial dependencies
- [ ] 3NF: no transitive dependencies
- [ ] When to denormalize for performance

### 3.2 NULL Behavior
- [ ] `NULL = NULL` is `NULL` (not true!)
- [ ] `IS NULL` vs `= NULL`
- [ ] `NULL` in UNIQUE columns (allows duplicates)
- [ ] `NULL` in indexes
- [ ] `COALESCE()`, `NULLIF()`, `IFNULL()`

### 3.3 Constraints
- [ ] `PRIMARY KEY` vs `UNIQUE` vs `INDEX`
- [ ] Composite unique constraints
- [ ] `ON DELETE CASCADE`, `SET NULL`, `RESTRICT`
- [ ] Foreign key constraints

### 3.4 Upserts
- [ ] MySQL: `INSERT INTO ... ON DUPLICATE KEY UPDATE`
- [ ] PostgreSQL: `INSERT INTO ... ON CONFLICT ... DO UPDATE`
- [ ] Laravel: `Model::updateOrCreate()` and `DB::upsert()`
- [ ] Use case: idempotent writes

### 3.5 Transactions & Locking
- [ ] `BEGIN`, `COMMIT`, `ROLLBACK`
- [ ] Pessimistic locking: `SELECT ... FOR UPDATE`
- [ ] Optimistic locking: version column
- [ ] Deadlocks: causes and prevention
- [ ] Isolation levels: READ COMMITTED, REPEATABLE READ, SERIALIZABLE

### 3.6 Indexing
- [ ] B-tree vs hash indexes
- [ ] When to add an index
- [ ] Composite indexes and column order
- [ ] `EXPLAIN` / `EXPLAIN ANALYZE`

---

## 4. API Design & Architecture

### 4.1 RESTful Design
- [ ] Proper HTTP methods and status codes
- [ ] Resource naming conventions
- [ ] Pagination (offset vs cursor)
- [ ] Filtering and sorting

### 4.2 API Versioning
- [ ] URI versioning (`/v1/users`)
- [ ] Header versioning
- [ ] Deprecation strategy (phased rollout)
- [ ] Sunset header

### 4.3 Idempotency
- [ ] What idempotency means and why it matters
- [ ] Idempotency keys: client generates UUID, server stores and checks
- [ ] Storage: Redis `SET NX` or DB unique constraint
- [ ] Retry-safe endpoints (POST, PUT, DELETE)

### 4.4 Rate Limiting
- [ ] Token bucket vs sliding window
- [ ] `429 Too Many Requests` response
- [ ] Per-user, per-IP, per-API-key limiting
- [ ] Laravel `ThrottleRequests` middleware

### 4.5 Error Handling
- [ ] RFC 7807 Problem Details
- [ ] Consistent error response format
- [ ] Logging vs exposing errors to client

### 4.6 Authentication
- [ ] JWT (access + refresh tokens)
- [ ] OAuth 2.0 flows (Authorization Code, Client Credentials)
- [ ] API tokens (Laravel Sanctum / Passport)
- [ ] Token expiry and revocation

### 4.7 Consumer-Driven Contracts
- [ ] What contract testing is
- [ ] Pact framework basics
- [ ] Provider vs consumer tests

---

## 5. Scalability & High Traffic

### 5.1 Connection Pooling
- [ ] What connection pooling is and why it matters
- [ ] `sql-mode` and persistent connections
- [ ] Laravel `DB::purge()` and connection reuse
- [ ] PgBouncer for PostgreSQL

### 5.2 Caching
- [ ] Redis vs Memcached
- [ ] Cache strategies: write-through, write-behind, cache-aside
- [ ] Cache invalidation patterns
- [ ] HTTP cache headers (ETag, Cache-Control)

### 5.3 Queue Systems
- [ ] Why async processing matters at scale
- [ ] Redis queue, SQS, RabbitMQ
- [ ] Job prioritization
- [ ] Delayed jobs and retries

### 5.4 Horizontal Scaling
- [ ] Stateless application design
- [ ] Load balancing (round-robin, least connections)
- [ ] Session storage (Redis, DB, cookie)
- [ ] Database read replicas

### 5.5 Database Protection
- [ ] Read/write splitting
- [ ] Connection limits and pooling
- [ ] Query optimization (N+1, slow query log)
- [ ] Sharding basics

---

## 6. DevOps & Environment

### 6.1 Docker
- [ ] Dockerfile best practices for PHP
- [ ] Docker Compose for local dev
- [ ] Multi-stage builds

### 6.2 CI/CD
- [ ] GitHub Actions / GitLab CI basics
- [ ] Automated testing in pipeline
- [ ] Deployment strategies (blue-green, rolling)

### 6.3 Monitoring
- [ ] Application logging (structured logs)
- [ ] Error tracking (Sentry, Bugsnag)
- [ ] Metrics (CPU, memory, request time)

---

## 7. Coding Challenge Prep

### 7.1 Problem-Solving Process
- [ ] Read requirements twice
- [ ] Clarify questions before coding
- [ ] Pseudocode / plan first
- [ ] Talk through your thought process

### 7.2 Edge Cases
- [ ] Empty inputs
- [ ] Null values
- [ ] Large inputs (performance)
- [ ] Concurrent access
- [ ] Invalid data types

### 7.3 Testing Your Solution
- [ ] Manual test cases before submitting
- [ ] Check all requirements against your code
- [ ] Time complexity analysis

---

## 8. Mock Interview Topics

Practice explaining these out loud (2-3 min each):

| Topic | Key Points to Cover |
|---|---|
| PHP-FPM lifecycle | master/worker, `pm` settings, request flow |
| Policies vs Gates | class-based vs callback, model-specific |
| NULL in UNIQUE | `NULL != NULL`, allows multiple NULLs |
| Idempotency keys | UUID, storage, retry safety |
| Connection pooling | reuse connections, reduce overhead |
| Upserts | `ON DUPLICATE KEY`, `updateOrCreate()` |
| API versioning | URI vs header, deprecation strategy |
| 3NF normalization | transitive dependencies, when to denormalize |
| Deadlocks | circular wait, prevention, detection |
| Consumer-driven contracts | Pact, provider/consumer roles |

---

## Weekly Plan

| Week | Focus Area |
|---|---|
| 1 | PHP Internals + Laravel Policies/Gates |
| 2 | Database Design (Normalization, NULL, Constraints, Upserts) |
| 3 | API Architecture (Idempotency, Versioning, Rate Limiting) |
| 4 | Scalability (Caching, Queues, Connection Pooling) |
| 5 | Coding Challenges + Mock Interviews |
| 6 | Review + Weak Areas + Practice Explaining Concepts |
