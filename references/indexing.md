# Advanced Indexing Strategies

## Partial Indexes (PostgreSQL)

```php
// Only index active users — smaller index, faster queries
DB::statement('CREATE INDEX idx_users_active_email ON users (email) WHERE is_active = true');

// Only index non-null values
DB::statement('CREATE INDEX idx_orders_shipped ON orders (shipped_at) WHERE shipped_at IS NOT NULL');
```

## Expression Indexes

```php
// Index on computed expression (PostgreSQL)
DB::statement('CREATE INDEX idx_users_lower_email ON users (LOWER(email))');

// MySQL generated column + index
Schema::table('users', function (Blueprint $table) {
    $table->string('email_normalized')->virtualAs('LOWER(email)')->index();
});
```

## Covering Indexes

```php
// Include columns to avoid table lookups (PostgreSQL)
DB::statement('CREATE INDEX idx_orders_status_covering ON orders (status) INCLUDE (total, created_at)');

// MySQL equivalent — composite index that covers the SELECT
$table->index(['status', 'total', 'created_at']); // Covers: SELECT total, created_at WHERE status = ?
```

## Index Analysis

```sql
-- MySQL: Check index usage
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'your_db';

-- MySQL: Check duplicate indexes  
SELECT * FROM sys.schema_redundant_indexes WHERE table_schema = 'your_db';

-- PostgreSQL: Index usage stats
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- PostgreSQL: Index size
SELECT pg_size_pretty(pg_relation_size('idx_name'));
```

## When to Use Composite vs Separate Indexes

```
Composite index (a, b):
  ✓ WHERE a = ? AND b = ?     (uses full index)
  ✓ WHERE a = ?               (uses left prefix)
  ✗ WHERE b = ?               (cannot use — wrong order)

Separate indexes (a), (b):
  ✓ WHERE a = ?               (uses index a)
  ✓ WHERE b = ?               (uses index b)
  ~ WHERE a = ? AND b = ?     (index merge — slower than composite)

Rule: If you always query a+b together, use composite.
      If you query a alone AND b alone, use separate indexes.
```
