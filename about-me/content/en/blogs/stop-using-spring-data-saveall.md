+++
title = "Stop Using Spring Data saveAll as a Hammer"
date = 2026-03-16T20:30:00+03:00
draft = false
description = "Why saveAll(...) is often the wrong tool for bulk inserts, and how to switch to insertBatch(...)."
tags = ["spring data", "jdbc", "performance", "postgresql", "java"]
categories = ["backend", "spring"]
+++

`saveAll(...)` is convenient. Super convenient.  
That is exactly why it gets overused.

The problem starts when you push > 1_000+ rows through it in one go.  
`saveAll(...)` is generic, but not always optimal for high-volume inserts: you often pay with extra SQL work, extra
round-trips, and measurable latency.

I ran into this in production-like scenarios and did not want bulk inserts to depend on "maybe it is fast enough".

## Why `saveAll(...)` hurts at scale

- `saveAll(...)` optimizes for API convenience, not maximum insert throughput;
- `saveAll(...)` does not only insert: if an entity already has an `id`, it goes through an update path;
- in the default flow it processes the collection in a regular loop, so for `N` objects you usually end up with roughly
  `N` write operations to the DB;
- on large batches, it often loses to dedicated batch strategies;
- the larger the data volume, the more visible the gap becomes.

For regular CRUD and small sets, this is usually fine.  
But for imports, sync pipelines, or technical jobs with thousands of rows, plain `saveAll(...)` can become a bottleneck
quickly.

## How I solved it

I used the Spring Data Repository Fragment approach:

1. `BulkRepository<T>` with `insertBatch(...)`
2. `BulkRepositoryImpl<T>` backed by `JdbcTemplate.batchUpdate(...)`
3. fragment registration via `META-INF/spring.factories`
4. plugging it into `UserRepository`

A short code-level idea:

```java
public interface UserRepository
        extends ListCrudRepository<User, Long>, BulkRepository<User> {
}
```

So the repository remains familiar, but gets a dedicated bulk method for mass operations.

Inside the implementation:

- SQL is generated from Spring Data mapping metadata;
- insert plans are cached per domain type;
- values are bound with prepared statement placeholders (`?`);
- entities with non-null `id` are skipped with `WARN`.

## Important JPA nuance

If you use Spring Data JPA, batching for `saveAll(...)` can often be enabled with Hibernate settings (for example,
`hibernate.jdbc.batch_size`, `hibernate.order_inserts`).  
But there are important constraints: `IDENTITY` id generation often breaks effective batching, and behavior also depends
on persistence context size, flush/clear strategy, and entity lifecycle details.

In other Spring Data modules, there is usually no single universal switch that makes bulk operations fast. That is why
explicit bulk repository methods are still a practical and predictable approach.

## Where this can go next

The biggest benefit is that this is not a one-off trick for `insertBatch(...)`.

- add `updateAll(...)` with batch updates;
- add `deleteBatch(...)` for mass deletions;
- build your own reusable bulk-operation layer across services.

In other words, you stop forcing all data-heavy workflows through one generic method and start shaping repository APIs
around real load patterns.

## Small benchmark snapshot

My current local run for 5,000 rows:

| Scenario      | Users | Generation (ms) | Insert (ms) | Total (ms) |
|---------------|------:|----------------:|------------:|-----------:|
| `saveAll`     |  5000 |               5 |         232 |        238 |
| `insertBatch` |  5000 |               5 |          58 |         67 |

Run on a remote database. The server is located in my city:

| Scenario      | Users | Generation (ms) | Insert (ms) | Total (ms) |
|---------------|------:|----------------:|------------:|-----------:|
| `saveAll`     | 10000 |              17 |         919 |        936 |
| `insertBatch` | 10000 |              17 |         171 |        188 |

## Repository with full implementation

Full example with benchmark endpoints, Flyway, and PostgreSQL:  
[github Ulllie/spring-data-ext](https://github.com/Ulllie/spring-data-ext)
