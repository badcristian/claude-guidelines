# Laravel Project Guidelines

> Project-specific conventions for Claude. Place this file in your project root.

---

## Models

### Property Annotations

Always annotate models with `@property` PHPDoc blocks. Group properties by type, with nullable properties at the bottom.

```php
/**
 * @property int $id
 * @property string $name
 * @property string $email
 * @property CompanyStatusEnum $status
 * @property Carbon $created_at
 * @property Carbon $updated_at
 * @property string|null $phone
 * @property string|null $website
 * @property Carbon|null $verified_at
 */
class Company extends Model
```

### Attributes (Accessors & Mutators)

Always use the modern `Illuminate\Database\Eloquent\Casts\Attribute` class. Never use the legacy `getXAttribute()` / `setXAttribute()` methods.

```php
// ✅ Correct
use Illuminate\Database\Eloquent\Casts\Attribute;

protected function fullName(): Attribute
{
    return Attribute::make(
        get: fn () => "{$this->first_name} {$this->last_name}",
    );
}

protected function password(): Attribute
{
    return Attribute::make(
        set: fn (string $value) => bcrypt($value),
    );
}

// ❌ Wrong - legacy style
public function getFullNameAttribute(): string
{
    return "{$this->first_name} {$this->last_name}";
}
```

### Relationships

Always define all relationships explicitly. Use return type hints.

```php
public function company(): BelongsTo
{
    return $this->belongsTo(Company::class);
}

public function orders(): HasMany
{
    return $this->hasMany(Order::class);
}

public function roles(): BelongsToMany
{
    return $this->belongsToMany(Role::class)->withTimestamps();
}
```

### Mass Assignment

Prefer `$guarded` over `$fillable` when most fields are fillable:

```php
// ✅ Simpler when most fields are fillable
protected $guarded = ['id'];

// Use $fillable only when you need explicit allowlist
protected $fillable = ['name', 'email'];
```

### Primary Key Access

Always use `->getKey()` instead of `->id` for better abstraction:

```php
// ✅ Correct
$user->getKey()
$company->getKey()

// ❌ Avoid
$user->id
```

---

## Query Builders

### When to Use Custom Query Builders

- Use custom query builder when model has **more than 2 scopes**
- Only create a scope (in model or builder) when it has **multiple conditions**
- Simple single-condition filters should be inline queries

### Builder Setup

Location: `app/QueryBuilders/`

```php
<?php

namespace App\QueryBuilders;

use App\Models\Company;
use Illuminate\Database\Eloquent\Builder;

/**
 * @mixin Company
 * @template TModelClass of Company
 * @extends Builder<TModelClass>
 */
class CompanyQueryBuilder extends Builder
{
    public function active(): self
    {
        return $this
            ->where('status', CompanyStatusEnum::Active)
            ->whereNotNull('verified_at')
            ->where('is_suspended', false);
    }

    public function withRecentActivity(int $days = 30): self
    {
        return $this
            ->where('last_activity_at', '>=', now()->subDays($days))
            ->orWhereHas('orders', fn ($q) => $q->where('created_at', '>=', now()->subDays($days)));
    }

    public function inRegion(string $region): self
    {
        return $this->where('region', $region);
    }
}
```

### Connecting Builder to Model

```php
<?php

namespace App\Models;

use App\QueryBuilders\CompanyQueryBuilder;
use Illuminate\Database\Eloquent\Model;

class Company extends Model
{
    public function newEloquentBuilder($query): CompanyQueryBuilder
    {
        return new CompanyQueryBuilder($query);
    }
}
```

### Usage

```php
// Now you get IDE autocompletion and type safety
Company::query()->active()->withRecentActivity()->get();
Company::query()->inRegion('EU')->active()->paginate();
```

---

## Enums

### Always Use Enums for Statuses

Never use raw strings for statuses. Always create an enum class.

```php
<?php

namespace App\Enums;

enum CompanyStatusEnum: string
{
    case Pending = 'pending';
    case Active = 'active';
    case Suspended = 'suspended';
    case Closed = 'closed';

    /** Badge color for UI display */
    public function badgeColor(): string
    {
        return match ($this) {
            self::Pending => 'yellow',
            self::Active => 'green',
            self::Suspended => 'red',
            self::Closed => 'gray',
        };
    }

    /** Human-readable label */
    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Pending Review',
            self::Active => 'Active',
            self::Suspended => 'Suspended',
            self::Closed => 'Closed',
        };
    }

    /** Icon name for UI */
    public function icon(): string
    {
        return match ($this) {
            self::Pending => 'clock',
            self::Active => 'check-circle',
            self::Suspended => 'x-circle',
            self::Closed => 'archive',
        };
    }

    /** Get statuses that allow editing */
    public static function editable(): array
    {
        return [self::Pending, self::Active];
    }
}
```

### Naming Convention

Enum classes must always end with `Enum`:

```php
CompanyStatusEnum    // ✅
OrderStatusEnum      // ✅
PaymentTypeEnum      // ✅

CompanyStatus        // ❌
Status               // ❌
```

---

## Services

### Always Use Facades

Create a facade for every service to enable mocking and cleaner syntax:

```php
// app/Services/PaymentService.php
<?php

namespace App\Services;

class PaymentService
{
    public function processPayment(Order $order, float $amount): PaymentResult
    {
        // Implementation
    }
}

// app/Facades/Payment.php
<?php

namespace App\Facades;

use Illuminate\Support\Facades\Facade;

/**
 * @method static PaymentResult processPayment(Order $order, float $amount)
 *
 * @see \App\Services\PaymentService
 */
class Payment extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return \App\Services\PaymentService::class;
    }
}

// Usage
Payment::processPayment($order, 99.99);
```

### Naming Convention

Service classes must end with `Service`:

```php
PaymentService       // ✅
NotificationService  // ✅
ReportService        // ✅

PaymentHandler       // ❌
Payments             // ❌
```

---

## Database & Performance

### Combine Multiple DB Calls

When a method has multiple database calls, try to combine them into a single query, especially for statistics.

```php
// ❌ Multiple queries
$totalOrders = Order::count();
$totalRevenue = Order::sum('amount');
$avgOrderValue = Order::avg('amount');
$pendingOrders = Order::where('status', 'pending')->count();

// ✅ Single query with raw expressions
$stats = Order::query()
    ->selectRaw('COUNT(*) as total_orders')
    ->selectRaw('SUM(amount) as total_revenue')
    ->selectRaw('AVG(amount) as avg_order_value')
    ->selectRaw('SUM(CASE WHEN status = ? THEN 1 ELSE 0 END) as pending_orders', ['pending'])
    ->first();
```

### Eager Loading

Always eager load relations when you know they'll be accessed:

```php
// ❌ N+1 problem
$companies = Company::all();
foreach ($companies as $company) {
    echo $company->owner->name;  // Query per iteration
}

// ✅ Eager loaded
$companies = Company::with(['owner', 'subscriptions'])->get();
```

For Laravel Nova, prefer adding eager loading to the resource:

```php
public static $with = ['owner', 'subscriptions'];
```

### Caching Guidelines

Only cache when there are actual calculations or multiple queries:

```php
// ❌ Don't cache single row lookups
Cache::remember("user_{$id}", 3600, fn () => User::find($id));

// ✅ Cache complex calculations
Cache::remember("dashboard_stats_{$userId}", 3600, function () use ($userId) {
    return [
        'total_revenue' => Order::where('user_id', $userId)->sum('amount'),
        'order_count' => Order::where('user_id', $userId)->count(),
        'avg_order_value' => Order::where('user_id', $userId)->avg('amount'),
        'top_products' => $this->calculateTopProducts($userId),
    ];
});
```

### Defer Non-Critical Operations

Use `defer()` for operations that don't need to complete before the response:

```php
public function store(Request $request): JsonResponse
{
    $order = Order::create($request->validated());

    // These don't need to block the response
    defer(fn () => $this->analytics->trackOrder($order));
    defer(fn () => $this->notifications->notifyAdmin($order));

    return response()->json($order);
}
```

---

## Migrations

### Always Specify Table in `constrained()`

```php
// ✅ Correct - explicit table name
$table->foreignId('company_id')->constrained('companies')->cascadeOnDelete();
$table->foreignId('created_by')->constrained('users')->nullOnDelete();

// ❌ Wrong - implicit (can break with non-standard naming)
$table->foreignId('company_id')->constrained();
```

---

## Requests

### Always Use `->input()` Method

```php
// ✅ Correct
$name = $request->input('name');
$email = $request->input('user.email');  // Nested
$tags = $request->input('tags', []);     // With default

// ❌ Avoid these
$name = $request->name;
$name = $request->get('name');
$name = $request['name'];
```

---

## Comments

### Single-Line DocBlocks

If a comment is only one line, keep it compact:

```php
// ✅ Correct - single line
/** Calculate the total including tax and discounts */
public function calculateTotal(): float

// ❌ Wrong - unnecessary line breaks
/**
 * Calculate the total including tax and discounts
 */
public function calculateTotal(): float
```

### When to Add Comments

Only add method comments when they clarify something non-obvious:

```php
// ✅ Needed - explains business logic
/** Prorated amount is calculated from the 15th of each month */
public function calculateProratedAmount(): float

// ❌ Unnecessary - method name is self-explanatory
/** Get the user's full name */
public function getFullName(): string
```

---

## Code Style

### Method Parameters

When parameters are long, use named arguments on separate lines. Omit parameters that match their default values:

```php
// ✅ Clean with named parameters
$report = $service->generateReport(
    user: $user,
    startDate: $startDate,
    endDate: $endDate,
    format: ReportFormatEnum::PDF,
);

// ❌ Hard to read
$report = $service->generateReport($user, $startDate, $endDate, ReportFormatEnum::PDF, true, null, 'monthly');
```

### String Interpolation

Prefer variable interpolation over concatenation or `sprintf`:

```php
// ✅ Preferred
$message = "User {$user->name} created order #{$order->getKey()}";
$path = "/users/{$userId}/orders";

// ❌ Avoid
$message = 'User ' . $user->name . ' created order #' . $order->getKey();
$message = sprintf('User %s created order #%d', $user->name, $order->getKey());
```

### Single Return Per Method

Prefer a single return statement when it doesn't hurt readability:

```php
// ✅ Preferred - single return
public function calculateDiscount(Order $order): float
{
    $discount = 0;

    if ($order->total > 100) {
        $discount = $order->total * 0.1;
    }

    if ($order->user->isPremium()) {
        $discount += 5;
    }

    return $discount;
}

// ✅ Also acceptable - early returns for guard clauses
public function processOrder(Order $order): bool
{
    if ($order->isPaid()) {
        return false;
    }

    if (! $order->hasStock()) {
        return false;
    }

    // Process order...
    return true;
}
```

### Avoid Unnecessary Methods

Don't create methods that are only used once or that wrap single-line calls:

```php
// ❌ Unnecessary wrapper
private function sendNotification(User $user): void
{
    Notification::send($user);
}

// ✅ Just call it directly where needed
Notification::send($user);

// ❌ Method used only once with simple logic
private function formatDate(Carbon $date): string
{
    return $date->format('Y-m-d');
}

// ✅ Inline with a comment if needed
// Format: YYYY-MM-DD for API compatibility
$formattedDate = $date->format('Y-m-d');
```

For complex logic used once, add a comment instead of creating a method:

```php
// Calculate prorated amount based on remaining days in billing cycle
$daysRemaining = now()->diffInDays($billingCycleEnd);
$dailyRate = $subscription->amount / 30;
$proratedAmount = $dailyRate * $daysRemaining;
```

---

## Logging

### Add Stack Trace for Errors

Always include the stack trace when logging exceptions:

```php
try {
    $this->processPayment($order);
} catch (PaymentException $e) {
    Log::error("Payment failed for order {$order->getKey()}", [
        'order_id' => $order->getKey(),
        'amount' => $order->total,
        'error' => $e->getMessage(),
        'trace' => $e->getTraceAsString(),
    ]);

    throw $e;
}
```

### Dedicated Log Channels for Features

For significant features, create a dedicated daily log channel:

```php
// config/logging.php
'channels' => [
    'payments' => [
        'driver' => 'daily',
        'path' => storage_path('logs/payments.log'),
        'level' => 'debug',
        'days' => 30,
    ],
    'imports' => [
        'driver' => 'daily',
        'path' => storage_path('logs/imports.log'),
        'level' => 'debug',
        'days' => 14,
    ],
],

// Usage
Log::channel('payments')->info("Processing payment for order {$order->getKey()}");
Log::channel('imports')->error('Import failed', ['trace' => $e->getTraceAsString()]);
```

---

## PHP Attributes

Use PHP attributes where supported and where they improve readability:

```php
use Illuminate\Database\Eloquent\Attributes\ObservedBy;
use Illuminate\Database\Eloquent\Attributes\ScopedBy;

#[ObservedBy(CompanyObserver::class)]
#[ScopedBy(ActiveScope::class)]
class Company extends Model
{
}
```

```php
use App\Attributes\RateLimit;

#[RateLimit(maxAttempts: 5, decayMinutes: 1)]
public function store(Request $request): JsonResponse
{
}
```

### Consider Attributes For

- Route definitions (`#[Route]`, `#[Get]`, `#[Post]`)
- Validation rules on DTOs
- Event listeners
- Middleware
- Caching configuration
- API documentation

---

## Laravel Nova

### Naming Convention

Resources must end with `Resource`, Actions with `Action`:

```php
CompanyResource     // ✅
CreateOrderAction   // ✅
SuspendUserAction   // ✅

Company            // ❌ (for Nova resource)
CreateOrder        // ❌ (for Nova action)
```

### Eager Loading in Resources

Add eager loading directly to resources when it makes sense:

```php
class CompanyResource extends Resource
{
    public static $with = ['owner', 'subscription', 'latestOrder'];
}
```

---

## Helper Functions

For small logic repeated too often, add to `app/helpers.php` (or `app/functions.php`):

```php
// app/helpers.php

if (! function_exists('format_money')) {
    function format_money(float $amount, string $currency = 'USD'): string
    {
        return number_format($amount, 2) . ' ' . $currency;
    }
}

if (! function_exists('generate_reference')) {
    function generate_reference(string $prefix = 'REF'): string
    {
        return $prefix . '-' . strtoupper(Str::random(8));
    }
}
```

Register in `composer.json`:

```json
{
    "autoload": {
        "files": ["app/helpers.php"]
    }
}
```

---

## Naming Conventions Summary

| Type | Suffix | Example |
|------|--------|---------|
| Enum | `Enum` | `OrderStatusEnum` |
| Service | `Service` | `PaymentService` |
| Nova Resource | `Resource` | `CompanyResource` |
| Nova Action | `Action` | `SuspendUserAction` |
| Query Builder | `QueryBuilder` | `CompanyQueryBuilder` |
| Job | `Job` | `ProcessOrderJob` |
| Event | `Event` | `OrderCreatedEvent` |
| Listener | `Listener` | `SendOrderConfirmationListener` |
| Observer | `Observer` | `CompanyObserver` |
| Policy | `Policy` | `CompanyPolicy` |
| Request | `Request` | `StoreCompanyRequest` |
| Middleware | `Middleware` | `EnsureUserIsActiveMiddleware` |

---

## Quick Reference

### Do

- ✅ Use `->getKey()` instead of `->id`
- ✅ Use `->input()` for request data
- ✅ Use enums for all statuses
- ✅ Use facades for services
- ✅ Combine DB queries when possible
- ✅ Eager load known relations
- ✅ Use `constrained('table')` explicitly
- ✅ Include stack trace in error logs
- ✅ Use named parameters for clarity
- ✅ Use variable interpolation for strings

### Don't

- ❌ Create single-line wrapper methods
- ❌ Create methods used only once (unless complex)
- ❌ Cache single row lookups
- ❌ Use legacy accessor/mutator methods
- ❌ Use raw strings for statuses
- ❌ Create scopes with single conditions
- ❌ Add obvious comments
