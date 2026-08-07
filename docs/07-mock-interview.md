# Mock Interview — Explain Out Loud

Practice explaining each topic in 2-3 minutes. Use these as prompts.

---

## Topic 1: PHP-FPM Lifecycle

**Say this:**

> "PHP-FPM is a FastCGI process manager that handles PHP execution. It runs a master process that spawns worker processes. Each worker handles one request at a time.
>
> When a request comes in, the web server (Nginx) sends it to PHP-FPM via FastCGI protocol. A worker picks it up, executes the PHP script, and returns the response.
>
> FPM has three modes: static (fixed workers), dynamic (min/max), and ondemand (spawns only when needed). In production, we use dynamic with settings like max_children=50 to handle traffic spikes.
>
> PHP-FPM also recycles workers after a certain number of requests to prevent memory leaks. The OPcache extension stores precompiled bytecode so we don't recompile the same script on every request."

---

## Topic 2: Laravel Policies vs Gates

**Say this:**

> "Gates and Policies are both authorization mechanisms in Laravel, but they serve different purposes.
>
> Gates are simple closure-based checks defined in AuthServiceProvider. They're best for general actions like 'can-edit-settings' that aren't tied to a specific model.
>
> Policies are class-based and model-specific. When you create a PostPolicy, it has methods like view, create, update, delete that receive the user and the model. Laravel automatically maps these to route model binding.
>
> For example, if I have `@can('update', $post)`, Laravel checks PostPolicy@update. If I have `@can('edit-settings')`, it checks the Gate definition.
>
> Policies are better for CRUD operations on a model, while Gates are for global actions. You can also use Gate::before() to give super admins bypass access."

---

## Topic 3: NULL in UNIQUE Columns

**Say this:**

> "In SQL, NULL is treated as an unknown value. So NULL is not equal to NULL — the comparison returns NULL, not TRUE.
>
> This means UNIQUE constraints allow multiple NULLs. If I have a UNIQUE constraint on the email column, I can insert multiple rows with NULL email. The database considers each NULL as distinct because NULL ≠ NULL.
>
> In MySQL and PostgreSQL, this is the default behavior. PostgreSQL 15 introduced NULLS NOT DISTINCT to change this if needed.
>
> This is important for idempotency and data modeling. If you want to enforce uniqueness including NULLs, you need a workaround like using a default value or a partial unique index."

---

## Topic 4: Idempotency Keys

**Say this:**

> "Idempotency ensures that making the same request multiple times produces the same result as making it once. This is critical for payment APIs and other operations where duplicates could be harmful.
>
> The client generates a unique idempotency key (usually a UUID) and sends it with each request. The server checks if this key exists in its cache or database. If it does, it returns the cached response without processing again. If not, it processes the request and stores the result with that key.
>
> For storage, you can use Redis with SET NX for atomic check-and-store, or a database table with a unique constraint on the key.
>
> POST and PATCH should always use idempotency keys. GET and DELETE are naturally idempotent. PUT is idempotent if the body is the same.
>
> In Laravel, I'd implement this as middleware that checks a cache key before processing and stores the response after."

---

## Topic 5: Connection Pooling

**Say this:**

> "Connection pooling maintains a pool of reusable database connections instead of opening and closing one for every request. Opening a connection takes about 100-500ms, and under high traffic, this overhead becomes significant.
>
> With pooling, a worker gets a connection from the pool, uses it, and returns it. The connection stays alive for the next request.
>
> In Laravel, you can enable persistent connections with PDO::ATTR_PERSISTENT. Each FPM worker holds its own persistent connection, so if you have 50 workers, you need at most 50 DB connections.
>
> For PostgreSQL, you can use PgBouncer as a connection pooler in front of the database, which can handle thousands of client connections with a smaller pool of actual DB connections.
>
> The key settings are max_children (how many workers), max_requests (when to recycle), and the database's max_connections limit."

---

## Topic 6: Upserts

**Say this:**

> "An upsert is an atomic operation that inserts a row if it doesn't exist, or updates it if it does. It's essential for idempotent writes.
>
> In MySQL, I use INSERT INTO ... ON DUPLICATE KEY UPDATE. For example, to increment a counter: INSERT INTO stats (user_id, count) VALUES (1, 1) ON DUPLICATE KEY UPDATE count = count + 1.
>
> In PostgreSQL, it's INSERT INTO ... ON CONFLICT (column) DO UPDATE SET ... .
>
> In Laravel, I use Model::updateOrCreate() for single records, or DB::upsert() for bulk operations. These handle the atomic check-and-write.
>
> Upserts are useful for importing CSV data, counting events, session management, and any scenario where you want to avoid race conditions between check and insert."

---

## Topic 7: API Versioning

**Say this:**

> "API versioning allows you to make changes without breaking existing clients. The most common approach is URI versioning, like /api/v1/users and /api/v2/users.
>
> When deprecating a version, I follow a phased approach. First, both versions work. Then I add deprecation headers (Sunset, Deprecation, Link to successor) and monitor usage. I notify clients via email. After the sunset date, I return 410 Gone.
>
> The alternative is header-based versioning using Accept-Version or a custom header. This keeps URIs clean but is harder to test in a browser.
>
> The key principle is backward compatibility — new versions should add features, not break existing behavior. Breaking changes warrant a new version."

---

## Topic 8: 3NF Normalization

**Say this:**

> "Third Normal Form eliminates transitive dependencies. A table is in 3NF if it's in 2NF and every non-key column depends only on the primary key, not on other non-key columns.
>
> For example, if I have an employees table with department_id and department_name, department_name depends on department_id, not on the employee id. This is a transitive dependency.
>
> To fix this, I create a separate departments table with id and name, and reference it from employees with a foreign key.
>
> However, sometimes I denormalize for performance. If I'm querying department names with every employee query, the JOIN might be expensive. I could store department_name on the employees table as a cache and keep it in sync via events or triggers."

---

## Topic 9: Deadlocks

**Say this:**

> "A deadlock occurs when two or more transactions are waiting for each other to release locks. Transaction A holds a lock on row 1 and wants row 2. Transaction B holds a lock on row 2 and wants row 1. Both wait forever.
>
> The database detects this and rolls back one transaction to break the cycle.
>
> To prevent deadlocks: always lock rows in the same order across all transactions. Keep transactions as short as possible. Use proper indexing to reduce lock scope. Use SELECT ... FOR UPDATE SKIP LOCKED when processing queues to skip already-locked rows.
>
> In Laravel, I wrap operations in DB::transaction() and avoid holding locks longer than necessary."

---

## Topic 10: Consumer-Driven Contracts

**Say this:**

> "Consumer-driven contract testing ensures that services can communicate without breaking each other. The consumer defines what it expects from the provider, and the provider verifies it meets those expectations.
>
> For example, if my frontend expects GET /api/posts to return { data: [{ id: 1, title: '...' }] }, that becomes a contract. When the backend team changes the response structure, the contract test fails before deployment.
>
> Pact is a popular tool for this. The consumer writes a test that describes its expectations. This generates a contract file. The provider runs a verification test against this contract.
>
> This enables independent deployment — the backend can change as long as it satisfies the contract. It's especially useful in microservices architectures."

---

## Practice Tips

1. **Time yourself** — aim for 2-3 minutes per topic
2. **Speak out loud** — don't just think it, say it
3. **Use concrete examples** — abstract explanations are forgettable
4. **Structure your answer** — What → How → When to use
5. **Mention trade-offs** — shows depth of understanding
