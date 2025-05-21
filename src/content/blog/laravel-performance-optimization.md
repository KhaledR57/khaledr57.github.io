---
title: "Laravel Performance Optimization: Solving Database Query Issues"
description: "How I tackled performance bottlenecks in a Laravel application by optimizing database queries"
pubDate: "Dec 8 2024 "
heroImage: "/laravel-banner.png"
tags: ["laravel", "performance", "database", "php", "optimization"]
---
I had a problem with my Laravel app running too slow. When I checked the export feature, I found it was making about **500 database calls**! This was too many, so I had to find ways to make it faster. Here's what I did to fix it.

### 1. Eager Loading Instead of Lazy Loading

Laravel automatically uses lazy loading, meaning it fetches related data from the database when you request it. The problem is that this leads to something called the **N+1 Query Problem**, where multiple unnecessary queries are made.

**The solution:**

I switched to using eager loading, which retrieves all the required items along with their related elements in one go using the `with()` method:

```php
$users = User::with('posts', 'comments')->get();
```

This change reduced the number of queries from 500 to around 150, which was a significant improvement.

### 2. Accessing Relations Efficiently ⬅️

Even with eager loading, how you access relations makes a difference. If you use `$model->relation()` (as a method call) instead of `$model->relation` (as a property), it can negate the benefits of eager loading and trigger additional queries.

**The solution:**

I made sure to use relations as properties, not method calls:

```php
// The correct way:
$relation = $model->relation;

// Avoid:
$relation = $model->relation();
```

This small detail helped further optimize performance. If you want more details, check out the difference between method-calls and property-access for model relations.

### 3. Chunking Results

Extracting large amounts of data can consume a lot of memory and cause performance issues or even process crashes.

**The solution:**

I used chunking to process the data in smaller pieces. Specifically, I used `chunkById`, which uses the index more efficiently and avoids issues with "limit" and "offset":

```php
Model::chunkById(1000, function ($records) {
    foreach ($records as $record) {
        // Process each record
    }
});
```

This method may result in more queries, but it is very memory-efficient, especially for background operations. If you want to learn more, check out "Use The Index, No Offset."

## The Result

After implementing these steps, the export process became much faster, and users no longer encountered timeouts. This significantly improved the user experience.

## Key Takeaways

- **Know Your Tools**: Laravel's ORM is powerful, but understanding the details unlocks its true potential.
- **Monitor and Analyze**: Tools like Debugbar are invaluable for identifying performance issues.
- **Think Long-Term**: Techniques like eager loading and chunking are essential when dealing with growing data over time.
- **Stress Testing is Key**: Make sure the system isn't just working under normal conditions but is also prepared to handle unexpected loads. Stress testing is a crucial part of developing any successful system.

## Additional Optimization Tips

Beyond the core issues I addressed, here are some additional Laravel performance optimization techniques worth implementing:

### Database Indexing

Properly indexed database tables can dramatically improve query performance. Always add indexes to columns that are frequently used in `WHERE`, `ORDER BY`, and `JOIN` clauses.

```php
Schema::table('users', function (Blueprint $table) {
    $table->index(['email', 'status']);
});
```

### Query Caching

For data that doesn't change frequently, consider implementing caching:

```php
$users = Cache::remember('users', 3600, function () {
    return User::all();
});
```

### Consider Using Raw Queries for Complex Operations

While Eloquent is convenient, sometimes a raw query can be more efficient for complex operations:

```php
$results = DB::select('SELECT * FROM users WHERE status = ? LIMIT 10', ['active']);
```

By implementing these techniques, you can build Laravel applications that not only work correctly but also perform excellently at scale.