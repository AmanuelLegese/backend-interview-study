# PHP Internals & Fundamentals

## 1. PHP-FPM Lifecycle

### What is PHP-FPM?
PHP-FPM (FastCGI Process Manager) is the process manager that handles PHP execution in production. It replaced the older `mod_php` approach and is the standard for running PHP applications.

### How It Works
```
                    ┌─────────────┐
  HTTP Request ───► │  Web Server  │ (Nginx/Apache)
                    └──────┬──────┘
                           │ FastCGI protocol
                    ┌──────▼──────┐
                    │  PHP-FPM     │
                    │  Master      │
                    │  Process     │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼───┐ ┌─────▼─────┐
        │  Worker 1  │ │Worker 2│ │  Worker N  │
        └───────────┘ └───────┘ └───────────┘
```

### Key Concepts

**Master Process:**
- Spawns and manages worker processes
- Handles configuration
- Monitors worker health
- Recycles crashed workers

**Worker Process:**
- Handles one request at a time
- Executes PHP code
- Returns response to web server

### pm Configuration Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `static` | Fixed number of workers | Predictable traffic |
| `dynamic` | Min/max workers, spawns on demand | Most common |
| `ondemand` | Spawns workers only when requests arrive | Low traffic |

```ini
; /etc/php/8.2/fpm/pool.d/www.conf
[www]
pm = dynamic
pm.max_children = 50        ; max workers
pm.start_servers = 5        ; workers at startup
pm.min_spare_servers = 3    ; idle workers kept
pm.max_spare_servers = 10   ; idle workers allowed
pm.max_requests = 500       ; recycle worker after N requests
```

### Request Lifecycle

```
1. INIT
   ├── Register constants
   ├── Pre-load extensions
   └── Read php.ini

2. STARTUP
   ├── Load extensions (OPcache, PDO, etc.)
   └── Register functions

3. PARSE (per request)
   ├── Lexing (tokens)
   ├── Parsing (AST)
   └── Compilation (opcodes)

4. EXECUTE (per request)
   ├── Zend VM executes opcodes
   └── Memory allocated per request

5. SHUTDOWN (per request)
   ├── Flush output buffers
   ├── Close connections
   └── Free request memory

6. TEARDOWN (on shutdown)
   ├── Close extensions
   └── Free global memory
```

### OPcache

OPcache stores precompiled PHP bytecode in shared memory, avoiding recompilation on every request.

```ini
; php.ini
opcache.enable=1
opcache.memory_consumption=128
opcache.max_accelerated_files=10000
opcache.validate_timestamps=0  ; 0 in production, 1 in dev
opcache.revalidate_freq=0
```

---

## 2. Strict vs Loose Comparison

### Loose Comparison (`==`)

PHP performs type juggling before comparing:

```php
0 == "foo"      // true  (string "foo" cast to int 0)
0 == ""         // true  (empty string cast to int 0)
"" == null      // true  (both empty-ish)
"1" == "01"     // true  (string comparison after casting)
false == []     // true  (both empty)
"0" == false    // true  (both falsy)
null == []      // true  (both empty-ish)
```

### Strict Comparison (`===`)

No type juggling — value AND type must match:

```php
0 === "foo"     // false (int vs string)
0 === ""        // false (int vs string)
"" === null     // false (string vs null)
"1" === "01"    // false (different strings)
false === []    // false (bool vs array)
"0" === false   // false (string vs bool)
null === []     // false (null vs array)
```

### Edge Cases to Remember

```php
// These are common interview gotchas:
"0" == false     // true
"0" === false    // false
"1" == true      // true
"1" === true     // false
PHP_INT_MAX + 1 == PHP_FLOAT_MAX  // depends on version

// NULL comparisons
null == 0        // true
null == ""       // true
null == false    // true
null == []       // true
null === 0       // false
null === ""      // false
null === false   // false
null === []      // false
```

### Safe Comparison Functions

```php
// strcmp() returns 0 for equal strings
strcmp("foo", "foo") === 0  // true
// Avoids type juggling issues

// password_verify() for password checking
password_verify($input, $hash)  // correct way
$input === $hash                // WRONG — timing attack risk

// json_decode() returns null on failure
$json = json_decode($input);
if ($json === null) {
    // decode failed
}
```

---

## 3. Pass by Value vs Pass by Reference

### Pass by Value (Default)

```php
function modify($x) {
    $x = 10;
}

$a = 5;
modify($a);
echo $a;  // 5 (unchanged)
```

### Pass by Reference (`&`)

```php
function modify(&$x) {
    $x = 10;
}

$a = 5;
modify($a);
echo $a;  // 10 (changed)
```

### foreach Gotcha with References

```php
// WRONG — creates reference to $value, causes bugs
$arr = [1, 2, 3];
foreach ($arr as &$value) {
    $value *= 2;
}
// $value is now a reference to last element!
unset($value); // FIX: always unset reference after loop

// CORRECT — PHP 7+
$arr = [1, 2, 3];
foreach ($arr as $value) {  // no &
    $value *= 2;  // only modifies local copy
}

// CORRECT — modify array directly
$arr = [1, 2, 3];
foreach ($arr as $key => $value) {
    $arr[$key] = $value * 2;
}
```

### Return by Reference

```php
& function getReference() {
    static $data = [];
    return $data;  // returns reference to static variable
}

$a = &getReference();
$a[] = 'item'; // modifies static $data directly
```

---

## 4. Error Handling

### Exception Hierarchy (PHP 8+)

```
Throwable
├── Error
│   ├── ArithmeticError
│   │   └── DivisionByZeroError
│   ├── AssertionError
│   ├── CompileError
│   │   └── ParseError
│   └── TypeError
│       ├── ArgumentCountError
│       └── NotImplementedError
└── Exception
    ├── ClosedException
    ├── Exception (yes, nested)
    ├── InvalidArgumentException
    ├── LogicException
    │   ├── BadFunctionCallException
    │   ├── BadMethodCallException
    │   ├── DomainException
    │   ├── LengthException
    │   ├── OutOfBoundsException
    │   ├── OutOfRangeException
    │   └── OverflowException
    ├── RangeException
    │   └── OverflowException
    ├── RuntimeException
    │   ├── DirectoryNotFoundException
    │   ├── OutOfMemoryException
    │   └── OverflowException
    └── UnderflowException
```

### Try/Catch/Finally

```php
try {
    $result = $db->query($sql);
} catch (PDOException $e) {
    // Handle specific exception
    Log::error("DB query failed: " . $e->getMessage());
    throw new AppException("Database error", 0, $e);
} finally {
    // Always runs, even if exception thrown or caught
    $db->close();
}
```

### Custom Exceptions

```php
class AppException extends RuntimeException {
    protected $context;

    public function __construct(string $message, int $code = 0, ?Throwable $previous = null, array $context = []) {
        parent::__construct($message, $code, $previous);
        $this->context = $context;
    }

    public function getContext(): array {
        return $this->context;
    }
}

class ValidationException extends AppException {
    protected array $errors;

    public function __construct(array $errors, ?Throwable $previous = null) {
        parent::__construct("Validation failed", 422, $previous, ['errors' => $errors]);
        $this->errors = $errors;
    }

    public function getErrors(): array {
        return $this->errors;
    }
}
```

### Error Handler Setup

```php
// Bootstrap file
set_error_handler(function ($severity, $message, $file, $line) {
    throw new ErrorException($message, 0, $severity, $file, $line);
});

set_exception_handler(function (Throwable $e) {
    http_response_code(500);
    echo json_encode(['error' => $e->getMessage()]);
    exit(1);
});

register_shutdown_function(function () {
    $error = error_get_last();
    if ($error !== null && in_array($error['type'], [E_ERROR, E_PARSE, E_CORE_ERROR, E_COMPILE_ERROR])) {
        // Handle fatal error
    }
});
```

### Laravel Exception Handling

```php
// app/Exceptions/Handler.php
class Handler extends ExceptionHandler {
    protected $dontReport = [
        // Don't log these
    ];

    public function register(): void {
        $this->reportable(function (Throwable $e) {
            // Send to Sentry, Bugsnag, etc.
        });

        $this->renderable(function (ValidationException $e, $request) {
            if ($request->expectsJson()) {
                return response()->json([
                    'message' => 'Validation failed',
                    'errors' => $e->errors(),
                ], 422);
            }
        });
    }
}
```
