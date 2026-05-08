# Spring Data JPA & Hibernate Guide

## When to use this guide

Read this file when the task is about:

- `@Entity` design and JPA annotations
- Spring Data repositories and query methods
- Hibernate `ddl-auto` behavior
- Fetch plans, projections, and transaction boundaries
- JPA/Hibernate performance tuning and review

For PostgreSQL runtime setup, Docker Compose, Testcontainers image choices, and production database operations, see [DATABASE.md](DATABASE.md).

## Contents

- [Defaults](#defaults)
- [Core Principle](#core-principle)
- [Hibernate DDL Auto Modes](#hibernate-ddl-auto-modes)
- [Observability First](#observability-first)
- [Entity Mapping](#entity-mapping)
- [Repository Patterns](#repository-patterns)
- [Fetch Strategy](#fetch-strategy)
- [Projection Strategy](#projection-strategy)
- [Performance Optimization](#performance-optimization)
- [Bulk Operations](#bulk-operations)
- [Transactions and Concurrency](#transactions-and-concurrency)
- [Review Checklist](#review-checklist)
- [Validation / Checks](#validation--checks)
- [References](#references)

## Defaults

- **Persistence stack:** Spring Data JPA + Hibernate
- **Schema management:** `spring.jpa.hibernate.ddl-auto`
- **Open Session in View:** disabled by default with `spring.jpa.open-in-view=false`
- **Recommended database:** PostgreSQL, configured separately in [DATABASE.md](DATABASE.md)
- **Default posture:** SQL-first, explicit fetch plans, bounded queries, short transactions

## Core Principle

Treat Hibernate as a SQL-generating tool, not an excuse to stop thinking about the database.

- Prefer simple, explicit queries over magical repository behavior.
- Fetch only the rows and columns needed by the use case.
- Use entities for write use cases and aggregate updates.
- Use DTO projections for read-only screens, lists, reports, and API payloads.
- Optimize for round trips, result size, persistence-context size, and transaction duration.

## Hibernate DDL Auto Modes

Hibernate generates the database schema automatically from your `@Entity` classes. No SQL migration files are required in this skill.

| Mode | Behavior | Use when |
|------|----------|----------|
| `update` | Creates/alters tables to match entities. Never drops. | Development |
| `validate` | Only validates schema matches entities. Fails on mismatch. | Production |
| `create` | Drops and recreates schema on startup. | Testing |
| `create-drop` | Like `create`, but also drops on shutdown. | Unit tests |
| `none` | Hibernate does nothing. | Manual schema management |

**Development**

```properties
spring.jpa.hibernate.ddl-auto=update
```

- Hibernate auto-creates tables, adds new columns, and applies safe structural changes
- It is suitable for iterative local development because it does not drop data

**Production**

```properties
spring.jpa.hibernate.ddl-auto=validate
```

- Hibernate checks that the live schema matches the entity model
- Application startup fails fast on schema drift
- Schema changes must be applied before deployment

## Observability First

Before changing mappings or adding caches, make Hibernate visible.

Recommended development settings:

```properties
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.properties.hibernate.log_slow_query=100
```

For older Hibernate versions, slow-query logging may need a different property. Check the Hibernate version in use before applying vendor-specific knobs mechanically.

Use a JDBC proxy such as `p6spy` in tests or local development when you need to inspect:

- SQL statements
- bind values
- execution time
- statement count
- whether batching is actually happening

For high-risk repository methods, add test expectations around:

- statement count
- absence of N+1 query behavior
- pagination behavior
- batching behavior

## Entity Mapping

Entity example:

```java
@Entity
@Table(name = "app_user")
public class AppUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 50, unique = true, nullable = false)
    private String login;

    @Column(length = 191, unique = true, nullable = false)
    private String email;

    @Column(name = "created_at")
    private Instant createdAt;
}
```

Guidelines:

- Keep entities explicit. Do not use Lombok.
- Use `@Table`, `@Column`, and `@Index` deliberately so Hibernate can derive accurate DDL.
- Prefer field-based access and stable, proxy-safe equality semantics.
- Keep bidirectional associations rare and intentional; they make fetch plans harder to reason about.
- Avoid unidirectional `@OneToMany` unless you have verified the SQL shape it creates.
- If a many-to-many join table has attributes, model it as a real entity instead of a direct `@ManyToMany`.

Identifier strategy guidance:

- `GenerationType.IDENTITY` is acceptable for normal CRUD applications and keeps generated projects simple.
- For high-volume insert workloads, `IDENTITY` limits batching. Prefer `SEQUENCE` with pooled allocation where the database supports it.
- Avoid `GenerationType.TABLE`.

Relationship defaults:

- Keep `@OneToMany` and `@ManyToMany` lazy.
- Explicitly set `@ManyToOne(fetch = FetchType.LAZY)`.
- Explicitly set `@OneToOne(fetch = FetchType.LAZY)` only when you understand the ownership model and the generated SQL.
- Never use `FetchType.EAGER` as a convenience shortcut.

## Repository Patterns

Use Spring Data JPA repositories for standard CRUD and simple query derivation:

```java
public interface AppUserRepository extends JpaRepository<AppUser, Long> {

    Optional<AppUser> findByLogin(String login);

    Page<AppUser> findByActiveTrue(Pageable pageable);
}
```

Use `@Query` when the method name becomes unclear or when you need a controlled fetch plan:

```java
@Query("""
    SELECT new com.example.app.user.UserSummary(u.id, u.login, u.email)
    FROM AppUser u
    WHERE u.active = true
    """)
List<UserSummary> findActiveSummaries();
```

Repository guidance:

- Use derived query methods for simple lookups only.
- Avoid long method names that hide query complexity.
- Never expose unbounded `findAll()` in application paths that can grow with production data.
- Do not call `save()` on already managed entities just to be explicit; dirty checking already handles that inside a transaction.
- Avoid routine `saveAndFlush()`; it often reduces batching and forces premature work.

The service layer is optional. For simple CRUD applications, controllers may call repositories directly. Add a service layer when it encapsulates business logic, transactions, orchestration, or reuse.

## Fetch Strategy

Lazy loading should happen inside a transactional method that deliberately fetches exactly what the use case needs.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "customer_id")
private Customer customer;
```

Use query-specific fetching with:

- `JOIN FETCH`
- `@EntityGraph`
- DTO projections
- batch fetching for lazy associations

Avoid relying on Open Session in View to hide lazy-loading mistakes:

```properties
spring.jpa.open-in-view=false
```

## Projection Strategy

Use entities when:

- you need insert, update, or delete behavior
- you need dirty checking or optimistic locking
- you are working with an aggregate inside a command transaction

Use DTO projections when:

- the data is read-only
- you only need a subset of columns
- you are rendering lists, dashboards, tables, exports, or API payloads
- you want to avoid hydrating a large graph

Example:

```java
@Query("""
    SELECT new com.example.app.user.UserSummary(u.id, u.login, u.email)
    FROM AppUser u
    WHERE u.active = true
    ORDER BY u.login
    """)
List<UserSummary> findActiveSummaries();
```

## Performance Optimization

### Avoiding N+1 Queries

The most common JPA performance issue. Use `JOIN FETCH` or `@EntityGraph` when the use case truly needs associations in one round trip:

```java
// BAD: triggers N+1 — one query per order's items
List<Order> orders = orderRepository.findAll();
orders.forEach(o -> o.getItems().size());

// GOOD: single query with JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") String status);

// GOOD: declarative with @EntityGraph
@EntityGraph(attributePaths = {"items"})
List<Order> findByStatus(String status);
```

Additional tools:

- `hibernate.default_batch_fetch_size=25`
- `@BatchSize(size = 25)`

Warning:

- Do not combine collection `JOIN FETCH` with pagination unless you have verified the generated SQL.
- For paginated list endpoints, prefer DTO queries or a two-query approach: page parent IDs first, then fetch details.

### Pagination

Always paginate large result sets:

```java
Page<AppUser> findByActiveTrue(Pageable pageable);
```

Deep offset pagination becomes slow on large tables. For high-volume endpoints, switch to keyset pagination on an indexed cursor column.

### Read-only transactions

Mark query-only service methods as read-only:

```java
@Service
public class UserService {

    @Transactional(readOnly = true)
    public Page<AppUser> listActive(Pageable pageable) {
        return userRepository.findByActiveTrue(pageable);
    }
}
```

Hibernate skips dirty-checking and auto-flush, which reduces unnecessary work on read paths. DTO projections are still the better default for truly read-only list endpoints.

### JDBC fetch size and streaming

Use streaming only when you genuinely need progressive processing of large result sets.

- Keep the transaction open while consuming the stream.
- Always close the stream with try-with-resources.
- Do not return a `Stream` outside the service-layer transaction boundary.

```java
@Transactional(readOnly = true)
public void processPosts() {
    try (Stream<Post> stream = postRepository.streamRecentPosts()) {
        stream.forEach(this::process);
    }
}
```

### Batch Operations

For insert and update-heavy workloads:

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=25
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true
```

Prefer `saveAll()` over repeated `save()` calls in a loop for standard Spring Data use cases.

If you need PostgreSQL to execute true insert batches efficiently, set:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb?reWriteBatchedInserts=true
```

For very large imports, flush and clear periodically:

```java
for (int i = 0; i < items.size(); i++) {
    entityManager.persist(items.get(i));

    if (i > 0 && i % batchSize == 0) {
        entityManager.flush();
        entityManager.clear();
    }
}
```

### Caching

Do not add Hibernate second-level cache before fixing SQL, indexes, fetch plans, and transaction boundaries.

For reference data or read-mostly entities, second-level cache can help:

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country { ... }
```

For application-level caching, prefer Spring's abstraction:

```java
@Cacheable("users")
public AppUser findByLogin(String login) {
    return userRepository.findByLogin(login);
}
```

Measure before enabling caching broadly. It helps read-mostly data, but adds overhead for write-heavy entities and invalidation-heavy flows.

### Query-shape hygiene

- Use bind parameters consistently.
- Keep dynamically generated query shapes under control.
- For large `IN` lists, `hibernate.query.in_clause_parameter_padding=true` can improve plan-cache friendliness.

## Bulk Operations

For large updates or deletes, do not load thousands of entities to change one column.

Prefer bulk JPQL or native SQL:

```java
@Modifying
@Query("""
update Post p
set p.status = :newStatus
where p.status = :oldStatus
""")
int updateStatus(@Param("oldStatus") PostStatus oldStatus,
                 @Param("newStatus") PostStatus newStatus);
```

Important:

- bulk operations bypass the persistence context
- bulk operations bypass entity lifecycle callbacks
- versioning and optimistic-lock behavior require explicit thought
- clear or synchronize the persistence context after running them

## Transactions and Concurrency

Keep transactions short and deliberate.

- Put transaction boundaries in the service layer.
- Avoid remote calls inside transactions.
- Do not keep transactions open while rendering views or serializing large graphs.
- Flush intentionally only when needed.

Use optimistic locking for normal concurrent updates:

```java
@Version
private long version;
```

Use pessimistic locking only for specific cases where conflicts must be prevented at the database level.

## Review Checklist

Flag these during review:

- `FetchType.EAGER`
- Open Session in View enabled as a shortcut
- entity graphs or fetch joins used without checking pagination impact
- entity objects returned directly from REST controllers without intent
- unbounded `findAll`
- long derived query method names
- `save` called on already managed entities
- routine `saveAndFlush`
- `GenerationType.IDENTITY` used in high-volume insert paths
- `GenerationType.TABLE`
- many-to-many mapped as `List`
- `CascadeType.REMOVE` on many-to-many or huge associations
- huge `@OneToMany` collections
- remote calls inside transactions
- bulk operations without clearing persistence context
- second-level cache used to mask bad query design

## Validation / Checks

- Verify that `ddl-auto=update` creates all expected tables on a fresh database
- Add integration tests that save and reload entities
- Inspect generated SQL or Hibernate statistics when changing fetch strategies
- Add query-count assertions for performance-sensitive repository methods when practical
- Keep `spring.jpa.open-in-view=false` so transaction boundaries remain explicit

## References

- [Spring Boot Data Access](https://docs.spring.io/spring-boot/reference/data/sql.html)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Vlad Mihalcea, Hibernate performance tuning tips](https://vladmihalcea.com/hibernate-performance-tuning-tips/)
- [Vlad Mihalcea, best Spring Data JpaRepository](https://vladmihalcea.com/best-spring-data-jparepository/)
- [Thorben Janssen, Hibernate performance tuning](https://thorben-janssen.com/hibernate-performance-tuning/)
- [Database Best Practices](DATABASE.md)
