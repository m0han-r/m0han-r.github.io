← [Back to Index](../README.md)

# JPA & Database Transaction Isolation Levels & Propagation

In enterprise Java applications using Spring Boot and Spring Data JPA / Hibernate, transaction management is one of the most heavily scrutinized areas in technical interviews. Understanding database isolation levels, concurrency anomalies (read phenomena), and Spring's transaction propagation behaviors is essential for building robust, high-throughput systems.

---

## 1. Concurrency Read Phenomena

When multiple database transactions execute concurrently without adequate isolation, several anomalies (read phenomena) can occur:

```
+-----------------------------------------------------------------------------------------+
|                               Concurrency Anomalies                                     |
+----------------------+------------------------------------------------------------------+
| Dirty Read           | Transaction reads uncommitted data written by another tx.        |
| Non-Repeatable Read  | Same query reads different values for same row within same tx.   |
| Phantom Read         | Re-executing range query finds newly inserted/deleted rows.      |
| Lost Update          | Two txs overwrite each other's changes without concurrency check.|
+----------------------+------------------------------------------------------------------+
```

### 1.1 Dirty Read
A transaction reads data that has been modified by another concurrent transaction but **not yet committed**. If the modifying transaction rolls back, the data read by the first transaction is invalid and never existed permanently.

```
Transaction 1 (Tx1)                    Transaction 2 (Tx2)
-------------------                    -------------------
BEGIN;                                 BEGIN;
UPDATE account SET balance = 500       
  WHERE id = 1; (Old balance: 1000)
                                       SELECT balance FROM account WHERE id = 1;
                                       --> Reads 500 (DIRTY READ!)
ROLLBACK; (balance restored to 1000)
                                       -- Tx2 operates on phantom balance 500!
                                       COMMIT;
```

### 1.2 Non-Repeatable Read (Fuzzy Read)
A transaction reads the same row twice, but between the two reads, another transaction **modifies or deletes** that row and commits. As a result, the two reads return different values for the same record.

```
Transaction 1 (Tx1)                    Transaction 2 (Tx2)
-------------------                    -------------------
BEGIN;
SELECT balance FROM account 
  WHERE id = 1; --> Returns 1000
                                       BEGIN;
                                       UPDATE account SET balance = 750 WHERE id = 1;
                                       COMMIT;
SELECT balance FROM account 
  WHERE id = 1; 
  --> Returns 750 (NON-REPEATABLE READ!)
COMMIT;
```

### 1.3 Phantom Read
A transaction executes a query returning a set of rows satisfying a search condition. Another transaction then **inserts or deletes** rows that satisfy the condition and commits. When the first transaction repeats the query with the exact same predicate, it receives a different set of rows (the "phantoms").

```
Transaction 1 (Tx1)                    Transaction 2 (Tx2)
-------------------                    -------------------
BEGIN;
SELECT COUNT(*) FROM orders 
  WHERE status = 'PENDING'; --> Returns 5
                                       BEGIN;
                                       INSERT INTO orders(status) VALUES ('PENDING');
                                       COMMIT;
SELECT COUNT(*) FROM orders 
  WHERE status = 'PENDING'; 
  --> Returns 6 (PHANTOM READ!)
COMMIT;
```

> **Key Distinction**: Non-Repeatable Read occurs on a **single row** (value modification/deletion), whereas Phantom Read occurs on a **range/set of rows** (new matching records appear or vanish).

### 1.4 Lost Update
Occurs when two transactions read the same record and subsequently update it based on the previously read value. The last commit wins, completely overwriting the modifications made by the earlier transaction without detecting the conflict. (Addressed by Optimistic/Pessimistic locking).

---

## 2. Standard ANSI SQL Isolation Levels

ANSI/ISO SQL-92 defines four standard isolation levels. Each level eliminates specific anomalies at the cost of concurrency and performance:

| Isolation Level | Dirty Reads Allowed? | Non-Repeatable Reads Allowed? | Phantom Reads Allowed? | Typical Mechanism | Default in DBMS |
| :--- | :---: | :---: | :---: | :--- | :--- |
| **READ UNCOMMITTED** | Yes | Yes | Yes | Read without locks; ignores rollback segments. | Rarely used |
| **READ COMMITTED** | **No** | Yes | Yes | Read locks released immediately; MVCC reads latest snapshot. | PostgreSQL, Oracle, SQL Server |
| **REPEATABLE READ** | **No** | **No** | Yes (ANSI) / No (MySQL InnoDB) | Read locks held till commit; MVCC reads snapshot as of Tx start. | MySQL (InnoDB) |
| **SERIALIZABLE** | **No** | **No** | **No** | Range locks (Next-Key Locking), strict 2PL, or SSI (Serializable Snapshot Isolation). | SQLite default, optional in all |

### 2.1 READ UNCOMMITTED (`Isolation.READ_UNCOMMITTED`)
- Lowest isolation level.
- Allows reading uncommitted, in-flight changes.
- **Use Case**: Extremely rare; high-volume telemetry or rough analytics where approximate counts are acceptable and absolute speed is prioritized.

### 2.2 READ COMMITTED (`Isolation.READ_COMMITTED`)
- Prevents Dirty Reads: only committed data can be read.
- Under MVCC (Multi-Version Concurrency Control), a query sees a snapshot of the data taken at the **start of that specific query statement**, not the transaction.
- **Default in**: PostgreSQL, Oracle Database, Microsoft SQL Server.
- **Interview Note**: Between two `SELECT` statements in the same transaction, another transaction can commit changes, causing Non-Repeatable and Phantom reads.

### 2.3 REPEATABLE READ (`Isolation.REPEATABLE_READ`)
- Guarantees that any row read during the transaction remains identical in subsequent reads within that same transaction.
- Under MVCC, a snapshot is created at the **start of the transaction** and reused for all subsequent reads throughout that transaction.
- **Default in**: MySQL (InnoDB engine).
- **Critical Interview Gotcha (MySQL InnoDB)**: MySQL's InnoDB prevents phantom reads even under `REPEATABLE READ` for standard non-locking `SELECT` statements via **MVCC (consistent snapshot read)**, and for locking reads (`SELECT ... FOR UPDATE` / `SELECT ... LOCK IN SHARE MODE`) via **Next-Key Locks** (record lock + gap lock).

### 2.4 SERIALIZABLE (`Isolation.SERIALIZABLE`)
- Complete isolation. Execution of concurrent transactions yields results equivalent to some serial (sequential) execution order.
- Implemented via two-phase locking (2PL) with predicate/range locks or Serializable Snapshot Isolation (SSI).
- Highest data integrity, lowest throughput. High risk of deadlocks and serialization failures (`40001` SQLSTATE).

---

## 3. Spring `@Transactional(isolation = ...)`

Spring provides declarative transaction management via `@Transactional`.

### 3.1 Supported Isolation Enum Values
In `org.springframework.transaction.annotation.Isolation`:
- `Isolation.DEFAULT`: Uses the default isolation level of the underlying database/datastore connection.
- `Isolation.READ_UNCOMMITTED`
- `Isolation.READ_COMMITTED`
- `Isolation.REPEATABLE_READ`
- `Isolation.SERIALIZABLE`

### 3.2 Java Code Example: Configuring Isolation Levels

```java
package com.example.service;

import com.example.entity.Account;
import com.example.repository.AccountRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

@Service
public class BankAccountService {

    private final AccountRepository accountRepository;

    public BankAccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    /**
     * READ_COMMITTED: Standard for everyday balance checks.
     * Prevents reading uncommitted overdrafts or deposits.
     */
    @Transactional(readOnly = true, isolation = Isolation.READ_COMMITTED)
    public BigDecimal getAccountBalance(Long accountId) {
        return accountRepository.findById(accountId)
                .map(Account::getBalance)
                .orElseThrow(() -> new IllegalArgumentException("Account not found: " + accountId));
    }

    /**
     * REPEATABLE_READ: Ensures balance calculations remain stable
     * across multiple reads while generating a composite financial statement.
     */
    @Transactional(readOnly = true, isolation = Isolation.REPEATABLE_READ)
    public FinancialReport generateMonthlyReport(Long accountId) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        BigDecimal initialRead = account.getBalance();

        // Perform complex audits and external validation...
        // Even if another transaction updates account balance now,
        // subsequent reads within this method see the exact same snapshot.
        BigDecimal secondaryAuditCheck = account.getBalance();
        
        return new FinancialReport(account.getId(), initialRead, secondaryAuditCheck);
    }

    /**
     * SERIALIZABLE: Strict order allocation to avoid race conditions
     * where multiple concurrent threads attempt to grab the last voucher/slot.
     */
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId)
                .orElseThrow(() -> new IllegalArgumentException("Source account not found"));
        Account to = accountRepository.findById(toId)
                .orElseThrow(() -> new IllegalArgumentException("Destination account not found"));

        if (from.getBalance().compareTo(amount) < 0) {
            throw new IllegalStateException("Insufficient funds");
        }

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        accountRepository.save(from);
        accountRepository.save(to);
    }
}
```

### 3.3 JPA & Hibernate First-Level Cache vs Isolation Levels
- JPA's `EntityManager` provides a **first-level (L1) cache** (Persistence Context).
- Once an entity is loaded within a transaction, any subsequent `findById()` with the same ID returns the cached entity instance from memory, **without issuing an SQL query to the database**.
- Therefore, JPA naturally simulates `REPEATABLE READ` behavior for individual entities inside a single transaction, regardless of database isolation level!
- However, native SQL queries or JPQL bulk queries may bypass or refresh entities, revealing underlying database isolation nuances.

---

## 4. Spring Transaction Propagation Types

Transaction propagation specifies what happens when a transactional method calls another transactional method. Spring offers 7 propagation types in `org.springframework.transaction.annotation.Propagation`:

```
Method A (@Transactional)
    └── calls ──> Method B (@Transactional(propagation = Propagation.???))
```

### 4.1 Detailed Breakdown of Propagation Levels

| Propagation | Existing Transaction Present? | No Existing Transaction? | Description / Typical Use Case |
| :--- | :--- | :--- | :--- |
| **`REQUIRED`** *(Default)* | Joins existing transaction. | Creates a new transaction. | Standard business workflows where all operations must succeed or fail as a single atomic unit. |
| **`REQUIRES_NEW`** | Suspends existing transaction; executes in a brand-new independent transaction. | Creates a new transaction. | Audit logs, notification history, or billing attempts that must commit regardless of caller's failure. |
| **`NESTED`** | Executes within a nested transaction using **JDBC Savepoints**. | Creates a new transaction (acts like `REQUIRED`). | Partial rollbacks. If child fails, rollback to savepoint while parent continues. (Requires DataSource support). |
| **`MANDATORY`** | Joins existing transaction. | **Throws Exception** (`IllegalTransactionStateException`). | Methods that must never run without an already established transaction context (e.g., internal audit verification). |
| **`SUPPORTS`** | Joins existing transaction. | Runs non-transactionally without creating one. | Read-only helper methods that can benefit from an existing transaction context if present. |
| **`NOT_SUPPORTED`** | Suspends existing transaction; runs non-transactionally. | Runs non-transactionally. | Long-running tasks, file generation, or third-party web service calls that shouldn't hold DB locks. |
| **`NEVER`** | **Throws Exception** (`IllegalTransactionStateException`). | Runs non-transactionally. | Operations that must strictly never execute inside a transactional context. |

### 4.2 Code Example: `REQUIRED` vs `REQUIRES_NEW` vs `NESTED`

#### Practical Scenario: Order Processing with Independent Audit Logging

```java
package com.example.service;

import com.example.entity.Order;
import com.example.entity.AuditLog;
import com.example.repository.OrderRepository;
import com.example.repository.AuditLogRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final AuditLogService auditLogService;
    private final InventoryService inventoryService;

    public OrderService(OrderRepository orderRepository,
                        AuditLogService auditLogService,
                        InventoryService inventoryService) {
        this.orderRepository = orderRepository;
        this.auditLogService = auditLogService;
        this.inventoryService = inventoryService;
    }

    /**
     * Parent transaction (REQUIRED - default).
     * If an unhandled exception occurs here, the order and inventory changes roll back.
     */
    @Transactional(propagation = Propagation.REQUIRED)
    public void placeOrder(Order order) {
        // 1. Audit attempt: MUST persist even if order placement fails later!
        auditLogService.logOrderAttempt(order.getCustomerId(), "ORDER_INITIATED");

        // 2. Persist order
        orderRepository.save(order);

        // 3. Deduct inventory (can throw OutOfStockException)
        inventoryService.deductStock(order.getProductId(), order.getQuantity());

        // 4. Audit success
        auditLogService.logOrderAttempt(order.getCustomerId(), "ORDER_COMPLETED");
    }
}
```

```java
@Service
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;

    public AuditLogService(AuditLogRepository auditLogRepository) {
        this.auditLogRepository = auditLogRepository;
    }

    /**
     * REQUIRES_NEW: Runs in its own independent physical transaction.
     * Even if placeOrder() rolls back due to OutOfStockException,
     * this audit log record is already committed to the database.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderAttempt(Long customerId, String action) {
        AuditLog log = new AuditLog(customerId, action, System.currentTimeMillis());
        auditLogRepository.save(log);
    }
}
```

```java
@Service
public class PaymentProcessingService {

    /**
     * NESTED: Sets a JDBC savepoint before invoking sub-payment method.
     * If the promotional discount calculation fails, only the discount
     * step rolls back to the savepoint, allowing base payment to continue.
     */
    @Transactional(propagation = Propagation.NESTED)
    public void applyPromotionalCredit(Long orderId, String promoCode) {
        // If this throws an exception caught by caller, only this sub-work rolls back!
    }
}
```

---

## 5. Comparison Table: `REQUIRES_NEW` vs `NESTED`

This is one of the most common senior-level Spring interview questions:

| Feature | `Propagation.REQUIRES_NEW` | `Propagation.NESTED` |
| :--- | :--- | :--- |
| **Physical Transactions** | **2 separate physical connections/transactions**. Suspends outer Tx. | **1 single physical transaction** with a JDBC **Savepoint**. |
| **Commit Timing** | Commits immediately when child method finishes, before parent completes. | Commits **only when the outer parent transaction commits**. |
| **Outer Tx Rollback Effect** | Child transaction **remains committed**; outer rollback does not affect it. | Child changes **are rolled back** if the outer parent rolls back! |
| **Child Tx Rollback Effect** | Child rollback is isolated; parent can catch exception and proceed. | Rolls back to savepoint; parent can catch exception and proceed. |
| **Driver / ORM Support** | Supported everywhere (JDBC, JPA, JTA). | Requires JDBC 3.0+ Savepoints; **Not supported by Hibernate JPA directly** without raw DataSource setup. |

---

## 6. Common Interview Questions & Answers

### Q1: What is the default transaction isolation level in MySQL, PostgreSQL, and Oracle?
- **MySQL (InnoDB)**: `REPEATABLE READ`.
- **PostgreSQL**: `READ COMMITTED`.
- **Oracle Database**: `READ COMMITTED`.
- **SQL Server**: `READ COMMITTED`.

### Q2: Does MySQL InnoDB REPEATABLE READ allow phantom reads?
**Answer**: By the ANSI SQL standard, `REPEATABLE READ` allows phantom reads. However, **MySQL InnoDB prevents phantom reads** in practice:
1. For **consistent non-locking reads** (`SELECT ...`): InnoDB uses **MVCC snapshot reads**, where every query inside the transaction reads from the snapshot established at the beginning of the transaction.
2. For **locking reads** (`SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`): InnoDB uses **Next-Key Locking** (a combination of index-row locks and gap locks), preventing other transactions from inserting rows into the gaps between existing keys.

### Q3: Why doesn't `@Transactional` work when one method calls another inside the same class?
**Answer**: Spring uses **AOP Dynamic Proxies** (CGLIB or JDK dynamic proxy). When an external client calls a bean method, the call goes through the proxy, which intercepts the call, opens the transaction, invokes the target, and commits/rollbacks.
When a method calls another method **within the same class (`this.methodB()`)**, the proxy is bypassed—the call is a direct internal Java invocation on the target instance (`this`). Therefore, no transaction advice is executed.
**Solutions**:
1. Move `methodB()` to a separate Spring `@Service` component.
2. Self-inject the bean (`@Autowired private MyService self; self.methodB()`).
3. Use `AopContext.currentProxy()` (requires `exposeProxy = true`).
4. Use AspectJ compile-time or load-time weaving instead of Spring proxy-based AOP.

### Q4: By default, which exceptions trigger a rollback in Spring `@Transactional`?
**Answer**: By default, Spring transactions roll back **only on unchecked exceptions** (`RuntimeException` and `Error`). They do **NOT** roll back on checked exceptions (`Exception` / `IOException` / `SQLException`).
To roll back on all exceptions, configure:
```java
@Transactional(rollbackFor = Exception.class)
```

### Q5: What happens if you specify `isolation = Isolation.REPEATABLE_READ` in Spring when using JPA?
**Answer**: JPA's `EntityManagerFactory` standard does not define isolation per transaction; isolation is typically a connection-level property managed by the JDBC DataSource.
When using Spring's `JpaTransactionManager` with Hibernate:
- Spring inspects whether the underlying `Connection` supports changing isolation levels.
- Spring borrows a connection from the pool, invokes `connection.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ)`, runs the transaction, and resets it before returning it to the pool.
- Note: Some connection pools and drivers require specific configurations to allow dynamic isolation level switching.

---

## 7. Gotchas & Best Practices

1. **Proxy Self-Invocation Trap**: Always remember that `@Transactional` is intercepted by proxies. Avoid private methods (proxies ignore them) and internal self-invocations.
2. **Connection Leak / Pool Starvation with `REQUIRES_NEW`**:
   - `REQUIRES_NEW` opens a **second database connection** from the connection pool while the outer transaction holds onto the first.
   - In high-concurrency environments with small connection pools (e.g., pool size 10, with 10 threads running outer transactions simultaneously), all 10 threads might acquire the first connection and wait indefinitely for a second connection for `REQUIRES_NEW`, leading to a complete **deadlock of the connection pool**!
3. **Keep Transactions Short**: Holding transactions open during slow external API calls or large file operations keeps database locks active, degrades database concurrency, and leads to connection timeouts. Use `Propagation.NOT_SUPPORTED` or perform I/O outside the transactional boundary.
4. **Always Specify `rollbackFor = Exception.class`**: Avoid nasty bugs where business checked exceptions fail to trigger rollbacks, leaving your database in an inconsistent state.
