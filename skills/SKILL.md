---
name: laravel-database-migrations
description: "Laravel database schema design, migrations, indexing strategies, and query performance optimization. Use when creating migrations, designing database schemas, adding indexes, optimizing slow queries, implementing soft deletes, writing seeders/factories, handling zero-downtime schema changes, partitioning large tables, or debugging query performance. Triggers on tasks involving migration creation, column types, foreign keys, composite indexes, query EXPLAIN analysis, database normalization, polymorphic relationships, pivot tables, or any database architecture decision. Use PROACTIVELY whenever database design, migrations, indexes, slow queries, or schema changes are mentioned."
compatible_agents:
  - Claude Code
  - Cursor
  - Windsurf
  - Copilot
tags:
  - laravel
  - database
  - migrations
  - mysql
  - postgresql
  - indexing
  - performance
  - schema
  - php
---

# Laravel Database Migrations & Optimization

Design efficient schemas, write safe migrations, add strategic indexes, and optimize query performance.

## Migration Patterns

### Well-Structured Migration

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('coupon_id')->nullable()->constrained()->nullOnDelete();
            $table->string('number', 20)->unique();
            $table->string('status', 30)->default('pending')->index();
            $table->decimal('subtotal', 12, 2);
            $table->decimal('discount', 12, 2)->default(0);
            $table->decimal('tax', 12, 2)->default(0);
            $table->decimal('total', 12, 2);
            $table->string('currency', 3)->default('USD');
            $table->text('notes')->nullable();
            $table->json('metadata')->nullable();
            $table->timestamp('paid_at')->nullable()->index();
            $table->timestamp('shipped_at')->nullable();
            $table->timestamp('completed_at')->nullable();
            $table->softDeletes();
            $table->timestamps();

            // Composite index for common queries
            $table->index(['user_id', 'status']);
            $table->index(['status', 'created_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

### Pivot Table

```php
Schema::create('order_product', function (Blueprint $table) {
    $table->id();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->foreignId('product_id')->constrained()->cascadeOnDelete();
    $table->integer('quantity')->default(1);
    $table->decimal('unit_price', 12, 2);
    $table->decimal('total', 12, 2);
    $table->timestamps();

    $table->unique(['order_id', 'product_id']);
});
```

### Polymorphic Table

```php
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->morphs('commentable'); // Creates commentable_type + commentable_id + index
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->text('body');
    $table->timestamps();
});
```

## Safe Production Migrations

### Adding a Column (Zero-Downtime)

```php
// Step 1: Add nullable column (safe — no table lock on modern MySQL/Postgres)
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('phone', 20)->nullable()->after('email');
    });
}

// Step 2 (separate migration): Backfill data if needed
public function up(): void
{
    DB::table('users')
        ->whereNull('phone')
        ->update(['phone' => '']);
}

// Step 3 (separate migration): Make non-nullable if required
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('phone', 20)->default('')->change();
    });
}
```

### Renaming a Column (Safe)

```php
// Safe in MySQL 8+ and Postgres — uses RENAME COLUMN (instant)
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->renameColumn('name', 'full_name');
    });
}
```

### Adding Index Without Downtime

```php
// MySQL: Use ALGORITHM=INPLACE for large tables
public function up(): void
{
    Schema::table('orders', function (Blueprint $table) {
        $table->index('email'); // Laravel handles this properly
    });
}

// For very large tables, consider:
DB::statement('CREATE INDEX CONCURRENTLY idx_orders_email ON orders (email)'); // Postgres
DB::statement('ALTER TABLE orders ADD INDEX idx_email (email), ALGORITHM=INPLACE, LOCK=NONE'); // MySQL
```

## Indexing Strategies

### When to Add Indexes

```php
// WHERE clauses — always index columns you filter by
$table->index('status');
$table->index('email');

// Foreign keys — Laravel auto-indexes these with constrained()
$table->foreignId('user_id')->constrained(); // Auto-indexed

// Composite indexes — for multi-column WHERE/ORDER BY
// Order matters: most selective column first
$table->index(['user_id', 'status']);        // WHERE user_id = ? AND status = ?
$table->index(['status', 'created_at']);     // WHERE status = ? ORDER BY created_at

// Unique constraints — also serve as indexes
$table->unique('email');
$table->unique(['user_id', 'slug']);

// Text search (MySQL fulltext)
$table->fullText(['title', 'body']);
```

### When NOT to Index

```
- Columns with low cardinality (boolean, enum with 2-3 values) used alone
- Tables with < 1000 rows (full scan is faster)
- Columns you only SELECT, never WHERE/ORDER/JOIN on
- Write-heavy tables where index maintenance exceeds read benefit
```

## Query Optimization

### EXPLAIN Analysis

```php
// Enable query log
DB::enableQueryLog();

$users = User::where('status', 'active')
    ->with('posts')
    ->orderBy('created_at', 'desc')
    ->paginate(20);

// Check queries
foreach (DB::getQueryLog() as $query) {
    dump($query['query'], $query['time'] . 'ms');
}

// EXPLAIN a query
$explain = DB::select('EXPLAIN ' . User::where('status', 'active')->toSql());
```

### Common Optimizations

```php
// SELECT only needed columns
User::select(['id', 'name', 'email'])->where('active', true)->get();

// Use chunk for large datasets
User::where('active', true)->chunk(1000, function ($users) {
    foreach ($users as $user) { /* process */ }
});

// Use lazy() for memory-efficient iteration
User::where('active', true)->lazy()->each(function ($user) {
    /* process — single row in memory at a time */
});

// Avoid N+1 with eager loading
Post::with(['author:id,name', 'tags:id,name'])->paginate(20);

// Use withCount instead of loading relation to count
User::withCount('posts')->having('posts_count', '>', 5)->get();

// Use subquery selects
User::addSelect([
    'last_order_at' => Order::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->limit(1),
])->get();

// Batch inserts
$chunks = collect($data)->chunk(1000);
foreach ($chunks as $chunk) {
    DB::table('logs')->insert($chunk->toArray());
}

// Upsert (insert or update)
User::upsert(
    $userData,
    ['email'],           // Unique key
    ['name', 'phone'],   // Columns to update on conflict
);
```

## Factories & Seeders

```php
// database/factories/OrderFactory.php
namespace Database\Factories;

use App\Models\User;

class OrderFactory extends Factory
{
    public function definition(): array
    {
        $subtotal = fake()->randomFloat(2, 10, 1000);
        $tax = round($subtotal * 0.14, 2);

        return [
            'user_id' => User::factory(),
            'number' => 'ORD-' . strtoupper(Str::random(8)),
            'status' => fake()->randomElement(['pending', 'paid', 'shipped', 'completed']),
            'subtotal' => $subtotal,
            'tax' => $tax,
            'total' => $subtotal + $tax,
            'currency' => 'USD',
        ];
    }

    public function paid(): static
    {
        return $this->state(fn () => [
            'status' => 'paid',
            'paid_at' => fake()->dateTimeBetween('-30 days'),
        ]);
    }

    public function shipped(): static
    {
        return $this->state(fn () => [
            'status' => 'shipped',
            'paid_at' => fake()->dateTimeBetween('-30 days', '-7 days'),
            'shipped_at' => fake()->dateTimeBetween('-7 days'),
        ]);
    }
}
```

## Key Rules

1. Always add indexes on columns used in WHERE, ORDER BY, and JOIN — explain before and after
2. Always use `constrained()` on foreign keys — adds index + FK constraint in one call
3. Always use `decimal(12, 2)` for money — never `float` (precision loss)
4. Always add composite indexes in selectivity order — most unique column first
5. Always make new columns `nullable()` first in production — then backfill, then constrain
6. Never drop columns in the same deploy as code changes — remove code reference first, then column
7. Always use `softDeletes()` on business-critical tables — accidental deletes are recoverable
8. Always use `chunk()` or `lazy()` for processing > 1000 rows — prevents memory exhaustion
9. Always use `upsert()` for bulk import/sync — single query instead of N select+insert
10. Always test migrations with `migrate:fresh --seed` before deploying — catch errors locally
