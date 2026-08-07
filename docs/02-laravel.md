# Laravel Framework

## 1. Policies vs Gates (Interview Focus)

### Gates — Simple Callback-Based Authorization

Gates are closures that determine if a user can perform an action. They're best for general actions that aren't tied to a specific model.

```php
// app/Providers/AuthServiceProvider.php
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::define('edit-settings', function (User $user) {
        return $user->is_admin;
    });

    Gate::define('publish-post', function (User $user, Post $post) {
        return $user->id === $post->author_id;
    });

    // Deny with message
    Gate::deny('delete-post', 'You cannot delete posts.');
}
```

**Using Gates:**
```php
// In controllers
if ($request->user()->can('edit-settings')) {
    // allow
}

// In Blade
@can('edit-settings')
    <a href="/settings">Edit Settings</a>
@endcan

// In Blade (negated)
@cannot('publish-post', $post)
    <p>You cannot publish this post.</p>
@endcannot

// Manual check
Gate::forUser($user)->allows('edit-settings');
```

**Gate Callbacks (before/after):**
```php
// Runs before any gate check
Gate::before(function (User $user, string $ability) {
    if ($user->is_super_admin) {
        return true; // super admin bypasses everything
    }
});

// Runs after gate check (if it returned null)
Gate::after(function (User $user, string $ability, ?bool $result) {
    if ($result === null && $user->is_editor) {
        return true; // editors can do anything not explicitly denied
    }
});
```

### Policies — Model-Specific Authorization

Policies are classes that organize authorization logic around a specific model. They're best when you have CRUD-like operations on a model.

**Generate Policy:**
```bash
php artisan make:policy PostPolicy --model=Post
```

```php
// app/Policies/PostPolicy.php
class PostPolicy
{
    // Define ability methods matching policy methods
    protected $policies = [
        Post::class => PostPolicy::class,
    ];

    public function viewAny(User $user): bool
    {
        return true; // everyone can view posts
    }

    public function view(User $user, Post $post): bool
    {
        return $user->id === $post->author_id
            || $user->is_admin;
    }

    public function create(User $user): bool
    {
        return $user->can('publish-posts');
    }

    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->author_id
            && $post->status !== 'published';
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->author_id
            || $user->is_admin;
    }

    public function restore(User $user, Post $post): bool
    {
        return $user->is_admin;
    }

    public function forceDelete(User $user, Post $post): bool
    {
        return $user->is_super_admin;
    }

    // Custom ability
    public function publish(User $user, Post $post): bool
    {
        return $user->id === $post->author_id
            && $post->status === 'draft';
    }
}
```

**Register in AuthServiceProvider:**
```php
protected $policies = [
    Post::class => PostPolicy::class,
    Comment::class => CommentPolicy::class,
];

public function boot(): void
{
    $this->registerPolicies();
}
```

**Using Policies:**
```php
// In controller
class PostController extends Controller
{
    public function update(Request $request, Post $post)
    {
        $this->authorize('update', $post);
        // or
        $this->authorize($post); // infers 'update' from method name

        $post->update($request->validated());
        return redirect()->route('posts.show', $post);
    }
}

// In Blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@policy('publish', $post)
    <button>Publish</button>
@endpolicy

// Manual check
$request->user()->can('update', $post);
$request->user()->cannot('delete', $post);
```

### Gates vs Policies — When to Use Which

| Aspect | Gates | Policies |
|--------|-------|----------|
| **Scope** | Global actions | Model-specific |
| **Structure** | Closure in AuthServiceProvider | Class with methods |
| **Best for** | `edit-settings`, `send-invite` | CRUD on a model |
| **Model binding** | Manual | Automatic |
| **Testability** | Harder | Easier |
| **Naming** | String ability | Method name |

**Rule of thumb:**
- Actions on a specific model → **Policy**
- Actions not tied to a model (settings, roles) → **Gate**
- Complex logic mixing both → Use **both** (Gate calls Policy)

---

## 2. Middleware

### How Middleware Works

```
Request → [Middleware 1] → [Middleware 2] → [Middleware 3] → Controller
         ← [Middleware 1] ← [Middleware 2] ← [Middleware 3] ← Response
```

### Creating Middleware

```bash
php artisan make:middleware EnsureUserIsAdmin
```

```php
// app/Http/Middleware/EnsureUserIsAdmin.php
class EnsureUserIsAdmin
{
    public function handle(Request $request, Closure $next, string $role = 'admin'): Response
    {
        if (!$request->user() || !$request->user()->hasRole($role)) {
            abort(403, 'Unauthorized.');
        }

        return $next($request);
    }
}
```

### Registering Middleware

```php
// app/Http/Kernel.php
protected $middleware = [
    \App\Http\Middleware\TrustProxies::class,
    \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
    \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
];

protected $middlewareGroups = [
    'web' => [
        \App\Http\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \App\Http\Middleware\VerifyCsrfToken::class,
    ],

    'api' => [
        \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        \Illuminate\Routing\Middleware\ThrottleRequests::class.':api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],
];

protected $middlewareAliases = [
    'auth' => \App\Http\Middleware\Authenticate::class,
    'admin' => \App\Http\Middleware\EnsureUserIsAdmin::class,
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    'signed' => \App\Http\Middleware\ValidateSignature::class,
    'cache' => \Illuminate\Http\Middleware\HandleCors::class,
];
```

### Using Middleware

```php
// Route
Route::get('/admin', [AdminController::class, 'index'])
    ->middleware('auth', 'admin');

// Multiple middleware
Route::post('/posts', [PostController::class, 'store'])
    ->middleware(['auth', 'throttle:60,1']);

// Controller constructor
class PostController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
        $this->middleware('admin')->only(['destroy']);
        $this->middleware('throttle:60,1')->only(['store']);
    }
}
```

### Middleware Parameters

```php
class EnsureRole
{
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        if (!in_array($request->user()->role, $roles)) {
            abort(403);
        }
        return $next($request);
    }
}

// Usage
Route::get('/dashboard', fn() => view('dashboard'))
    ->middleware('role:admin,editor');
```

---

## 3. Eloquent ORM

### Relationships

```php
class User extends Model
{
    // One-to-One
    public function profile(): HasOne
    {
        return $this->hasOne(Profile::class);
    }

    // One-to-Many
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    // Many-to-Many
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class, 'user_roles')
                    ->withPivot('assigned_at')
                    ->withTimestamps();
    }

    // Has-Many-Through
    public function comments(): HasManyThrough
    {
        return $this->hasManyThrough(Comment::class, Post::class);
    }

    // Polymorphic One-to-Many
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }

    // Polymorphic Many-to-Many
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}
```

### Eager Loading (Prevent N+1)

```php
// BAD — N+1 queries
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name; // 1 query per post!
}

// GOOD — Eager load
$posts = Post::with('author')->get(); // 2 queries total
foreach ($posts as $post) {
    echo $post->author->name;
}

// Nested eager loading
$posts = Post::with(['author', 'comments.user', 'tags'])->get();

// Conditional eager loading
$posts = Post::with('author:id,name')->get(); // only load specific columns

// Lazy eager loading (after initial query)
$posts = Post::all();
$posts->load('author', 'comments');
```

### Query Scopes

```php
class Post extends Model
{
    // Local scope
    public function scopePublished(Builder $query): Builder
    {
        return $query->where('status', 'published');
    }

    public function scopeByAuthor(Builder $query, int $authorId): Builder
    {
        return $query->where('author_id', $authorId);
    }

    public function scopeRecent(Builder $query, int $days = 7): Builder
    {
        return $query->where('created_at', '>=', now()->subDays($days));
    }

    // Combining scopes
    public function scopePopular(Builder $query): Builder
    {
        return $query->where('views', '>', 1000)
                     ->where('status', 'published');
    }
}

// Usage
$posts = Post::published()->recent(30)->get();
$posts = Post::byAuthor(1)->popular()->get();
```

### Accessors & Mutators

```php
class User extends Model
{
    // Accessor (get)
    public function getNameAttribute(string $value): string
    {
        return ucfirst($value);
    }

    // Accessor for computed attribute
    public function getFullNameAttribute(): string
    {
        return "{$this->first_name} {$this->last_name}";
    }

    // Mutator (set)
    public function setEmailAttribute(string $value): void
    {
        $this->attributes['email'] = strtolower($value);
    }

    // Date accessor
    public function getCreatedAtAttribute($value)
    {
        return Carbon::parse($value)->format('M d, Y');
    }
}

// Usage
$user->name;  // calls getNameAttribute
$user->full_name;  // computed
$user->email = 'TEST@EXAMPLE.COM';  // stored as 'test@example.com'
```

### Query Builders

```php
// Complex queries
$users = DB::table('users')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->where('orders.status', 'completed')
    ->where('orders.created_at', '>=', now()->subDays(30))
    ->groupBy('users.id')
    ->havingRaw('COUNT(orders.id) > 5')
    ->orderByDesc('total_spent')
    ->limit(10)
    ->get();

// Chunking (memory efficient for large datasets)
User::chunk(100, function ($users) {
    foreach ($users as $user) {
        // process
    }
});

// Cursor (even more memory efficient)
foreach (User::cursor() as $user) {
    // process one at a time
}
```

---

## 4. Queues & Jobs

### Creating Jobs

```bash
php artisan make:job SendWelcomeEmail
```

```php
// app/Jobs/SendWelcomeEmail.php
class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public User $user
    ) {}

    public function handle(Mailer $mailer): void
    {
        $mailer->to($this->user->email)
               ->send(new WelcomeMail($this->user));
    }

    public function failed(Throwable $exception): void
    {
        Log::error("Failed to send welcome email to {$this->user->email}", [
            'error' => $exception->getMessage(),
        ]);
    }

    public function middleware(): array
    {
        return [new WithoutOverlapping('email')];
    }
}
```

### Dispatching Jobs

```php
// Basic dispatch
SendWelcomeEmail::dispatch($user);
dispatch(new SendWelcomeEmail($user));

// Delayed dispatch
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(5));

// On specific queue
SendWelcomeEmail::dispatch($user)->onQueue('emails');

// Job chaining
Bus::chain([
    new ProcessPhoto($photo),
    new CreateThumbnail($photo),
    new NotifyUser($photo),
])->onConnection('sqs')->onQueue('media')->dispatch($photo);

// Batching
Bus::batch([
    new ProcessImport($record1),
    new ProcessImport($record2),
    new ProcessImport($record3),
])->then(function (Batch $batch) {
    // All jobs completed
})->catch(function (Batch $batch, Throwable $e) {
    // First batch job failure
})->finally(function (Batch $batch) {
    // All jobs completed (success or failure)
})->dispatch();
```

### Queue Configuration

```php
// config/queue.php
'connections' => [
    'sync' => [
        'driver' => 'sync', // runs immediately (for testing)
    ],
    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
        'retry_after' => 90,
    ],
    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => 'default',
        'retry_after' => 90,
        'block_for' => null,
    ],
    'sqs' => [
        'driver' => 'sqs',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'prefix' => env('SQS_PREFIX'),
        'queue' => env('SQS_QUEUE'),
        'region' => env('AWS_DEFAULT_REGION'),
    ],
],
```

### Running Workers

```bash
php artisan queue:work redis --queue=high,default,low
php artisan queue:work --tries=3 --timeout=60
php artisan queue:work --max-jobs=1000  # restart after 1000 jobs
php artisan queue:listen  # re-opens connection each job (slower but safer)
```

---

## 5. Events & Listeners

### Creating Events

```bash
php artisan make:event OrderPlaced
php artisan make:listener SendOrderConfirmation
```

```php
// app/Events/OrderPlaced.php
class OrderPlaced
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public Order $order
    ) {}
}

// app/Listeners/SendOrderConfirmation.php
class SendOrderConfirmation
{
    public function __construct(
        private Mailer $mailer
    ) {}

    public function handle(OrderPlaced $event): void
    {
        $this->mailer->to($event->order->user->email)
                     ->send(new OrderConfirmation($event->order));
    }
}
```

### Registering Events

```php
// app/Providers/EventServiceProvider.php
protected $listen = [
    OrderPlaced::class => [
        SendOrderConfirmation::class,
        UpdateInventory::class,
        NotifyAdmin::class,
    ],
];

// Or use Event::listen() in boot()
public function boot(): void
{
    Event::listen(OrderPlaced::class, SendOrderConfirmation::class);
}
```

### Using Events

```php
// Fire event
OrderPlaced::dispatch($order);
event(new OrderPlaced($order));

// Queued listener (implement ShouldQueue)
class SendOrderConfirmation implements ShouldQueue
{
    // Job config here
}
```
