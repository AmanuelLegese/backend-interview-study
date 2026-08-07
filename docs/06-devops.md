# DevOps & Environment

## 1. Docker

### Dockerfile for PHP/Laravel

```dockerfile
# Multi-stage build for smaller image
FROM php:8.2-fpm AS base

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git curl zip unzip libpng-dev libonig-dev libxml2-dev \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Node.js (for frontend build)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www

# Production stage
FROM base AS production

COPY . .
RUN composer install --no-dev --optimize-autoloader --no-interaction
RUN npm ci && npm run build

RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache

FROM php:8.2-fpm AS runtime
COPY --from=production /var/www /var/www
WORKDIR /var/www

EXPOSE 9000
CMD ["php-fpm"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: base
    volumes:
      - .:/var/www
    depends_on:
      - database
      - redis

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - .:/var/www
      - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

  database:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: myapp
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  queue:
    build:
      context: .
      dockerfile: Dockerfile
    command: php artisan queue:work
    volumes:
      - .:/var/www
    depends_on:
      - database
      - redis

volumes:
  mysql_data:
```

### Docker Commands

```bash
# Build
docker compose build

# Run
docker compose up -d

# Logs
docker compose logs -f app

# Execute command
docker compose exec app php artisan migrate

# Stop
docker compose down
```

---

## 2. CI/CD (GitHub Actions)

### Laravel CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: testing
          MYSQL_ROOT_PASSWORD: password
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: mbstring, xml, ctype, json, bcmath, pdo, pdo_mysql
          coverage: xdebug

      - name: Install Dependencies
        run: composer install --no-progress --prefer-dist

      - name: Prepare Environment
        run: |
          cp .env.testing .env
          php artisan key:generate

      - name: Run Migrations
        php artisan migrate --force

      - name: Run Tests
        run: php artisan test --coverage

      - name: Run PHPStan
        run: vendor/bin/phpstan analyse

      - name: Run Pint (Laravel Pint)
        run: vendor/bin/pint --test
```

### Deployment Strategy

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www/app
            git pull origin main
            composer install --no-dev
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            sudo systemctl restart php8.2-fpm
```

---

## 3. Monitoring & Logging

### Structured Logging

```php
// Laravel Log channels
// config/logging.php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['daily', 'slack'],
    ],

    'daily' => [
        'driver' => 'daily',
        'path' => storage_path('logs/laravel.log'),
        'level' => env('LOG_LEVEL', 'debug'),
        'days' => 14,
    ],

    'stderr' => [
        'driver' => 'monolog',
        'level' => env('LOG_LEVEL', 'debug'),
        'handler' => Monolog\Handler\StreamHandler::class,
        'formatter' => env('LOG_STDERR_FORMATTER'),
        'with' => [
            'stream' => 'php://stderr',
        ],
    ],
];

// Usage
Log::info('User logged in', ['user_id' => $user->id, 'ip' => $request->ip()]);
Log::error('Payment failed', ['order_id' => $order->id, 'error' => $e->getMessage()]);
Log::warning('High memory usage', ['memory' => memory_get_usage(true)]);
```

### Error Tracking (Sentry)

```php
// config/sentry.php
return [
    'dsn' => env('SENTRY_DSN'),
    'traces_sample_rate' => 1.0,
    'profiles_sample_rate' => 1.0,
];

// Bootstrap
Sentry\init([
    'dsn' => env('SENTRY_DSN'),
    'traces_sample_rate' => 1.0,
]);

// Report exception
try {
    // code
} catch (\Throwable $e) {
    Sentry\captureException($e);
    throw $e;
}
```

### Metrics to Monitor

| Category | Metrics |
|----------|---------|
| **Application** | Request duration, error rate, queue depth |
| **Server** | CPU, memory, disk I/O, network |
| **Database** | Connection count, slow queries, replication lag |
| **PHP-FPM** | Active workers, request queue, memory per worker |

### Health Check Endpoint

```php
// routes/api.php
Route::get('/health', function () {
    $checks = [
        'database' => fn() => DB::select('SELECT 1'),
        'redis' => fn() => Redis::ping(),
        'storage' => fn() => Storage::exists('test'),
    ];

    $healthy = true;
    $results = [];

    foreach ($checks as $name => $check) {
        try {
            $check();
            $results[$name] = 'ok';
        } catch (\Exception $e) {
            $results[$name] = 'error: ' . $e->getMessage();
            $healthy = false;
        }
    }

    return response()->json([
        'status' => $healthy ? 'healthy' : 'unhealthy',
        'checks' => $results,
    ], $healthy ? 200 : 503);
});
```

---

## 4. Environment Configuration

### Laravel .env

```env
APP_NAME=Laravel
APP_ENV=production
APP_KEY=base64:...
APP_DEBUG=false
APP_URL=https://example.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=app
DB_USERNAME=user
DB_PASSWORD=secret

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=...
MAIL_PASSWORD=...
```

### Twelve-Factor App Principles

1. **Codebase** — one codebase in version control
2. **Dependencies** — declare dependencies explicitly
3. **Config** — store config in environment variables
4. **Backing services** — treat databases/queues as attached resources
5. **Build, release, run** — separate build and run stages
6. **Processes** — stateless processes
7. **Port binding** — self-contained via port binding
8. **Concurrency** — scale via process model
9. **Disposability** — fast startup, graceful shutdown
10. **Dev/prod parity** — keep environments similar
11. **Logs** — treat logs as event streams
12. **Admin processes** — run as one-off tasks
