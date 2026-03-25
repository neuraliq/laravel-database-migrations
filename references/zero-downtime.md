# Zero-Downtime Migration Patterns

## Safe Column Operations

### Adding a Column

```php
// SAFE: nullable columns don't lock table on MySQL 8+ / PostgreSQL
Schema::table('users', function (Blueprint $table) {
    $table->string('timezone', 50)->nullable()->after('email');
});
```

### Renaming a Column

```php
// SAFE on MySQL 8+ (uses RENAME COLUMN, instant)
// SAFE on PostgreSQL (always instant)
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('name', 'full_name');
});
```

### Dropping a Column

```php
// Step 1: Deploy code that no longer references the column
// Step 2: Drop column in NEXT deploy (separate migration)
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn('legacy_field');
});
// WARNING: On MySQL < 8.0, DROP COLUMN locks the table. Use pt-online-schema-change.
```

## Dangerous Operations & Safe Alternatives

### Changing Column Type

```php
// DANGEROUS: May lock table and rewrite all rows
$table->text('description')->change(); // from string to text

// SAFER: Create new column, migrate data, drop old
// Migration 1: Add new column
$table->text('description_new')->nullable();

// Migration 2: Copy data (run as batch job, not in migration)
DB::table('products')->lazyById(1000)->each(function ($product) {
    DB::table('products')->where('id', $product->id)
        ->update(['description_new' => $product->description]);
});

// Migration 3: Rename columns (after code is updated)
$table->renameColumn('description', 'description_old');
$table->renameColumn('description_new', 'description');

// Migration 4: Drop old column (next deploy)
$table->dropColumn('description_old');
```

### Adding NOT NULL Constraint

```php
// DANGEROUS: Fails if existing NULLs exist
$table->string('phone')->nullable(false)->change();

// SAFE: Three-step process
// 1. Backfill NULLs
DB::table('users')->whereNull('phone')->update(['phone' => '']);
// 2. Add default
$table->string('phone')->default('')->change();
// 3. Drop nullable (if needed, in next deploy)
```

## Large Table Migrations (MySQL)

```php
// For tables with millions of rows, use pt-online-schema-change
public function up(): void
{
    if (app()->isProduction()) {
        DB::statement("
            pt-online-schema-change --alter 'ADD COLUMN phone VARCHAR(20) DEFAULT NULL'
            --execute D=mydb,t=users
        ");
    } else {
        Schema::table('users', function (Blueprint $table) {
            $table->string('phone', 20)->nullable();
        });
    }
}
```

## PostgreSQL Concurrent Index Creation

```php
// Non-blocking index creation for large tables
public function up(): void
{
    DB::statement('CREATE INDEX CONCURRENTLY idx_orders_email ON orders (email)');
}

// IMPORTANT: Cannot run inside a transaction
// Add to migration class:
public $withinTransaction = false;
```
