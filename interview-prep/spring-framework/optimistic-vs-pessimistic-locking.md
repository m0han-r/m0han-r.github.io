← [Back to Index](../README.md)

# Optimistic vs Pessimistic Locking in JPA & Hibernate

In concurrent enterprise applications, multiple users or background jobs frequently attempt to read and mutate the same database records simultaneously. Without appropriate locking mechanisms, systems suffer from **Lost Updates**, race conditions, and data inconsistency.

This guide provides deep technical insights into **Optimistic** and **Pessimistic** locking strategies using JPA, Hibernate, and Spring Data JPA.

---

## 1. The Concurrency Problem: Lost Updates

Consider an e-commerce inventory with 1 item remaining. Two users, Alice and Bob, simultaneously attempt to purchase the item:

```
Time    Alice's Transaction (Tx A)            Bob's Transaction (Tx B)
 0      BEGIN;                                BEGIN;
 1      SELECT stock FROM item WHERE id = 1;  
        --> stock = 1                         
 2                                            SELECT stock FROM item WHERE id = 1;
                                              --> stock = 1
 3      stock = stock - 1 (1 - 1 = 0)
 4      UPDATE item SET stock = 0;            stock = stock - 1 (1 - 1 = 0)
 5      COMMIT;                               
 6                                            UPDATE item SET stock = 0;
 7                                            COMMIT;
```

**Result**: 2 purchases succeeded, but stock only decremented by 1! Tx B overwrote Tx A's change without being aware of it. This is a classic **Lost Update**.

To solve this, database architectures use two concurrency control models:
- **Optimistic Locking**: Assume conflicts are rare. Detect collisions at write time and abort/retry.
- **Pessimistic Locking**: Assume conflicts are frequent. Block concurrent access using database row locks.

---

## 2. Optimistic Locking

Optimistic Locking operates at the **application/ORM level** (not the database lock level). Transactions do not lock rows when reading. Instead, each record tracks a version number or timestamp. When updating, the ORM checks if the record's version has changed since it was read.

### 2.1 The JPA `@Version` Annotation

JPA provides the `@Version` annotation to automatically manage version tracking on entities.

```java
package com.example.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private Integer stock;

    private BigDecimal price;

    /**
     * Optimistic locking version field.
     * Automatically incremented by Hibernate on every UPDATE.
     * Supported types: int, Integer, short, Short, long, Long, Timestamp, Instant.
     */
    @Version
    private Long version;

    public Product() {}

    public Product(String name, Integer stock, BigDecimal price) {
        this.name = name;
        this.stock = stock;
        this.price = price;
    }

    // Getters and Setters...
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Integer getStock() { return stock; }
    public void setStock(Integer stock) { this.stock = stock; }
    public Long getVersion() { return version; }
}
```

### 2.2 How Hibernate Executes the SQL Under the Hood

When Hibernate flushes updates for an entity with `@Version`, it appends the version to the `WHERE` clause and increments the version in the `SET` clause:

```sql
UPDATE products 
SET stock = 4, version = version + 1 
WHERE id = 101 AND version = 3;
```

- **If the row was modified by another transaction**: The database finds 0 matching rows (`updated rows = 0`) because `version` in the database is already `4`.
- Hibernate detects that row count is `0` and immediately throws:
  - `org.hibernate.StaleObjectStateException` (Hibernate-specific)
  - `jakarta.persistence.OptimisticLockException` (JPA standard)
  - Spring translates these into `org.springframework.dao.OptimisticLockingFailureException`.

### 2.3 Handling Collisions: Retry Strategy

Optimistic locking requires a retry mechanism when an `OptimisticLockingFailureException` occurs.

#### Option A: Spring Retry with `@Retryable`

```java
package com.example.service;

import com.example.entity.Product;
import com.example.repository.ProductRepository;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class InventoryService {

    private final ProductRepository productRepository;

    public InventoryService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    /**
     * Retries up to 3 times with exponential backoff if OptimisticLockingFailureException occurs.
     */
    @Retryable(
        retryFor = { OptimisticLockingFailureException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2.0)
    )
    @Transactional
    public void deductStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new IllegalArgumentException("Product not found: " + productId));

        if (product.getStock() < quantity) {
            throw new IllegalStateException("Insufficient stock for product: " + productId);
        }

        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
    }

    /**
     * Fallback recovery method executed when all retry attempts fail.
     */
    @Recover
    public void recover(OptimisticLockingFailureException e, Long productId, int quantity) {
        throw new IllegalStateException("High concurrency conflict: unable to complete purchase for product " + productId, e);
    }
}
```

---

## 3. Pessimistic Locking

Pessimistic Locking relies on **database-level row locks**. When a transaction reads a record, it locks the corresponding database row using SQL syntax like `SELECT ... FOR UPDATE`. Any other concurrent transaction attempting to read (with lock) or modify the row is **blocked** until the first transaction commits or rolls back.

```
Time    Transaction 1 (Tx1)                    Transaction 2 (Tx2)
 0      BEGIN;                                 BEGIN;
 1      SELECT * FROM products WHERE id = 1    
        FOR UPDATE; (Lock acquired)
 2                                             SELECT * FROM products WHERE id = 1 FOR UPDATE;
                                               --> BLOCKED! Waiting for lock...
 3      UPDATE products SET stock = stock - 1;
 4      COMMIT; (Lock released)
 5                                             --> Lock acquired by Tx2!
                                               Reads updated stock value!
 6                                             UPDATE products SET stock = stock - 1;
 7                                             COMMIT;
```

### 3.1 JPA Pessimistic Lock Modes

JPA defines several lock modes in `jakarta.persistence.LockModeType`:

| LockModeType | Underlying SQL (PostgreSQL/MySQL) | Behavior |
| :--- | :--- | :--- |
| **`PESSIMISTIC_WRITE`** | `SELECT ... FOR UPDATE` | **Exclusive Lock**. Prevents other transactions from reading (with lock), updating, or deleting. Ideal for mutating data. |
| **`PESSIMISTIC_READ`** | `SELECT ... FOR SHARE` / `LOCK IN SHARE MODE` | **Shared Lock**. Allows other transactions to read, but blocks any transaction from modifying or deleting the row. |
| **`PESSIMISTIC_FORCE_INCREMENT`** | `SELECT ... FOR UPDATE` + `@Version` bump | Acquires exclusive row lock AND increments entity's `@Version` field even if no fields changed. Useful when updating child collections. |

### 3.2 Spring Data JPA Code Example with Pessimistic Locking

```java
package com.example.repository;

import com.example.entity.Product;
import jakarta.persistence.LockModeType;
import jakarta.persistence.QueryHint;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.jpa.repository.QueryHints;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    /**
     * Acquires an exclusive database lock (SELECT ... FOR UPDATE).
     * Includes a timeout hint to prevent threads from hanging indefinitely.
     */
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({
        @QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000") // 3-second timeout
    })
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);

    /**
     * Shared lock (SELECT ... FOR SHARE).
     */
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForRead(@Param("id") Long id);
}
```

### 3.3 Pessimistic Lock Timeout Options
If a thread tries to acquire a locked row, it will wait. If the holding transaction takes too long, it may cause connection pool starvation. You can configure timeouts via query hints:
- **Value in milliseconds** (e.g. `"3000"` for 3 seconds): Throws `PessimisticLockException` or `LockTimeoutException` on timeout.
- **`0` (NOWAIT)**: Fails immediately (`SELECT ... FOR UPDATE NOWAIT`) if the row is already locked.
- **`-2` (SKIP LOCKED)**: Skips already-locked rows (`SELECT ... FOR UPDATE SKIP LOCKED`). Extremely useful for job queues or task processing tables where multiple workers consume work items in parallel!

---

## 4. Optimistic vs Pessimistic Locking: Comprehensive Comparison

| Dimension | Optimistic Locking | Pessimistic Locking |
| :--- | :--- | :--- |
| **Lock Level** | Application / ORM level (Software check). | Database level (RDBMS row/page locks). |
| **Lock Duration** | Zero lock time during reading; brief lock at `UPDATE` commit. | Held from the moment query executes until transaction `COMMIT`/`ROLLBACK`. |
| **Deadlock Risk** | **Virtually None** at the DB row lock level. | **High** if multiple transactions acquire locks in different orders. |
| **Throughput / Scalability** | **High** under low to moderate contention. Does not hold DB connections. | **Low to Medium** under high concurrency; blocked threads exhaust connection pools. |
| **Collision Handling** | Fails at transaction commit; requires rollback & application retry. | Prevents collisions by serializing access at the database. |
| **Conflict Frequency** | Best suited for **Low to Moderate** conflict scenarios. | Best suited for **High Contention** scenarios. |
| **Multi-Request Workflows** | Works across disconnected HTTP requests (via version token or ETag). | **Impossible** across multiple HTTP requests (cannot hold DB connection open). |

---

## 5. When to Choose Which? (Decision Framework)

```
                            How frequent are write conflicts?
                                     │
                   ┌─────────────────┴─────────────────┐
                   │                                   │
              Low / Rare                         Frequent / High
                   │                                   │
                   ▼                                   ▼
        [Optimistic Locking]                 [Pessimistic Locking]
       - Read-heavy systems                 - Flash-sale inventory
       - Stateless REST APIs                - Seat / ticket booking
       - Long user think-time               - Bank account debiting
       - Microservices architecture         - High contention on small # of rows
```

### Use Optimistic Locking When:
1. **High Read-to-Write Ratio**: 95% of requests only read; collisions are rare.
2. **Stateless Web / REST APIs**: A user fetches an order, views it on a web form for 2 minutes, and submits updates. Optimistic versioning tracks the version across the HTTP request lifecycle (e.g., using `ETag` and `If-Match` headers).
3. **High Scalability Requirements**: Holding database row locks degrades database throughput and limits horizontal scaling.

### Use Pessimistic Locking When:
1. **High Contention (Hot Spots)**: E.g., Black Friday flash sale with 10,000 users attempting to buy 10 iPhone units. Under optimistic locking, 9,990 transactions would fail and retry, causing a catastrophic **Retry Storm** and wasting CPU/database cycles.
2. **Cost of Retry is Unacceptable**: Complex business workflows where rolling back and restarting consumes significant resources or calls external non-idempotent third-party APIs.

---

## 6. Common Interview Questions & Answers

### Q1: Why is `Long` or `Integer` preferred over `java.sql.Timestamp` for `@Version`?
**Answer**:
1. **Precision Limits**: Timestamps depend on system clock resolution (milliseconds or microseconds). In high-frequency transactions, two concurrent updates could occur within the same millisecond, receiving the identical timestamp and resulting in an undetected lost update.
2. **Clock Skew**: In clustered/distributed environments where multiple application servers point to the same database, JVM system clocks can drift out of sync.
3. Numeric increments (`version + 1`) are deterministic, monotonic, and immune to time drift.

### Q2: What is a "Retry Storm" in Optimistic Locking?
**Answer**:
A retry storm occurs under high contention when multiple concurrent threads fail optimistic lock checks simultaneously. If all failed threads immediately retry at the same time, they collide again, fail again, and continue looping.
**Prevention**: Use exponential backoff with random jitter (e.g., Spring Retry with `@Backoff(random = true)`).

### Q3: How do you prevent deadlocks when using Pessimistic Locking?
**Answer**:
Deadlocks occur when Tx1 locks Row A and requests Row B, while Tx2 locks Row B and requests Row A.
**Best Practices to Prevent Deadlocks**:
1. **Consistent Lock Ordering**: Always acquire locks in the same predictable sequence (e.g., sort primary keys in ascending order before locking: `items.sort(Comparator.comparing(Item::getId))`).
2. **Configure Lock Timeouts**: Always set `jakarta.persistence.lock.timeout` or use `NOWAIT` so transactions fail fast rather than deadlocking indefinitely.
3. Keep transactions as short as possible.

### Q4: What is `SELECT ... FOR UPDATE SKIP LOCKED` and what is its primary use case?
**Answer**:
`SKIP LOCKED` instructs the database engine to skip any rows currently locked by other transactions and return only the available unlocked rows that match the query predicate.
**Primary Use Case**: Implementing high-throughput, concurrent **distributed task/message queues** stored in relational database tables. Multiple worker nodes can run `SELECT * FROM task_queue WHERE status = 'PENDING' LIMIT 1 FOR UPDATE SKIP LOCKED` simultaneously without blocking each other or picking the same task.

### Q5: How do you implement Optimistic Locking across stateless HTTP requests?
**Answer**:
1. Send the entity's `version` field to the client as an HTTP `ETag` header or JSON property.
2. When the client submits an update (`PUT`/`PATCH`), it passes the version back in the `If-Match` request header.
3. The server loads the entity, verifies that the client version matches the database version, applies changes, and saves. If mismatched, the server returns HTTP `412 Precondition Failed` or `409 Conflict`.

---

## 7. Gotchas & Best Practices

1. **Do Not Manually Mutate `@Version`**: Never call `entity.setVersion(...)` in business logic. The version field must be managed strictly by Hibernate.
2. **Avoid Holding Pessimistic Locks During External I/O**: Never perform slow HTTP calls, RPCs, or file processing while holding a `PESSIMISTIC_WRITE` lock. This blocks other database transactions and exhausts the HikariCP connection pool.
3. **Bulk Updates Bypass `@Version`**: JPQL bulk update queries (`@Modifying @Query("UPDATE Product p SET p.stock = ...")`) bypass the Hibernate entity lifecycle and do **NOT** increment `@Version` automatically. You must manually increment the version in the JPQL query (`SET p.stock = ..., p.version = p.version + 1`).
4. **Use `@Version` Even With Pessimistic Locking (Hybrid Approach)**: Having `@Version` on entities even when occasionally using `PESSIMISTIC_WRITE` provides safety against queries that forget to declare pessimistic locks.
