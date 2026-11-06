# 📚 Java Backend Interview Preparation

> Comprehensive interview-ready notes for **Senior Java Developers (8+ years)** covering Data Structures, Java Internals, Spring Boot, PostgreSQL, Design Patterns, Microservices, and System Design — with code examples, diagrams, and Q&A.

---

## 🗂️ All Topics at a Glance

| # | Topic | Category |
|---|-------|----------|
| 1 | [Cyclic Linked List](data-structures/cyclic-linked-list.md) | Data Structures |
| 2 | [TreeMap](data-structures/treemap.md) | Data Structures |
| 3 | [HashMap & Red-Black Tree](data-structures/hashmap-and-red-black-tree.md) | Data Structures |
| 4 | [Tree Traversals](data-structures/tree-traversals.md) | Data Structures |
| 5 | [LRU Cache](data-structures/lru-cache.md) | Data Structures |
| 6 | [Java Features (All LTS: 8→25)](java-internals/java-features.md) | Java Internals |
| 7 | [Functional Interface](java-internals/functional-interface.md) | Java Internals |
| 8 | [Synchronized vs Semaphore](java-internals/synchronized-vs-semaphore.md) | Java Internals |
| 9 | [Heap Space Architecture](java-internals/heap-space-architecture.md) | Java Internals |
| 10 | [Advanced Concurrency](java-internals/concurrency-advanced.md) | Java Internals |
| 11 | [Caching Strategies](java-internals/caching-strategies.md) | Java Internals |
| 12 | [Spring Boot Internals](spring-framework/spring-boot-internals.md) | Spring Framework |
| 13 | [Spring Security & JWT](spring-framework/spring-security.md) | Spring Framework |
| 14 | [REST API Best Practices](spring-framework/rest-api-best-practices.md) | Spring Framework |
| 15 | [JPA Isolation Levels](spring-framework/jpa-isolation-levels.md) | Spring Framework |
| 16 | [AOP (Aspect-Oriented Programming)](spring-framework/aop.md) | Spring Framework |
| 17 | [Optimistic vs Pessimistic Locking](spring-framework/optimistic-vs-pessimistic-locking.md) | Spring Framework |
| 18 | [Microservices Patterns](spring-framework/microservices-patterns.md) | Spring Framework |
| 19 | [PostgreSQL Internals & Performance](database/postgres-internals-and-performance.md) | Database |
| 20 | [Advanced SQL (PostgreSQL)](database/advanced-sql.md) | Database |
| 21 | [Design Patterns — All 23 GoF](design-patterns/design-patterns-overview.md) | Design Patterns |
| 22 | [SOLID Principles](design-patterns/solid-principles.md) | Design Patterns |
| 23 | [Circuit Breaker & Fallback](design-patterns/circuit-breaker-and-fallback.md) | Design Patterns |
| 24 | [Saga Pattern](design-patterns/saga-pattern.md) | Design Patterns |
| 25 | [Kafka vs Pub/Sub](messaging/kafka-vs-pubsub.md) | Messaging |

---

## 📁 Data Structures & Algorithms

Core data structure concepts frequently asked in Java backend interviews.

- 🔗 [**Cyclic Linked List**](data-structures/cyclic-linked-list.md) — Circular linked list types, Floyd's cycle detection algorithm, Java implementation
- 🌳 [**TreeMap**](data-structures/treemap.md) — Red-Black Tree internals, NavigableMap API, comparison with HashMap/LinkedHashMap
- 🗄️ [**HashMap & Red-Black Tree**](data-structures/hashmap-and-red-black-tree.md) — Hashing, bucket array, collision handling, Java 8 treeification, rehashing
- 🌲 [**Tree Traversals**](data-structures/tree-traversals.md) — Inorder, Preorder, Postorder, Level-order, Morris traversal with Java code
- 📦 [**LRU Cache**](data-structures/lru-cache.md) — Doubly Linked List + HashMap approach, LinkedHashMap approach, LeetCode #146

---

## ☕ Java Internals

Deep dive into JVM, concurrency, and Java language features across all LTS versions.

- 🚀 [**Java Features — All LTS & Major Releases**](java-internals/java-features.md) — Java 8 → 25: Lambdas, Streams, Records, Sealed Classes, Virtual Threads, Pattern Matching, and more with code examples for every feature
- 🎯 [**Functional Interface**](java-internals/functional-interface.md) — @FunctionalInterface, built-in interfaces, SAM, composition, method references
- 🔒 [**Synchronized vs Semaphore**](java-internals/synchronized-vs-semaphore.md) — Intrinsic locks, lock escalation, Semaphore permits, ReentrantLock comparison
- 🧠 [**Heap Space Architecture**](java-internals/heap-space-architecture.md) — Young/Old Gen, Metaspace, TLAB, GC flow, OOM types, diagnostic tools
- ⚡ [**Advanced Concurrency**](java-internals/concurrency-advanced.md) — CompletableFuture, Virtual Threads, ForkJoinPool, Executor Framework, Locks (ReentrantLock, StampedLock), Atomic classes, ConcurrentHashMap, ThreadLocal, deadlock detection
- 💾 [**Caching Strategies**](java-internals/caching-strategies.md) — Cache-Aside, Read/Write-Through, Redis (data structures, eviction, Cluster), Caffeine, Spring Cache, distributed caching challenges, multi-level caching

---

## 🍃 Spring Framework & Spring Boot

Spring Boot internals, security, microservices, and enterprise patterns for senior developers.

- ⚙️ [**Spring Boot Internals**](spring-framework/spring-boot-internals.md) — Auto-configuration, @Conditional, Starters, Embedded servers, Actuator & Micrometer, Profiles, Bean lifecycle, Custom auto-configuration, Spring Boot 3.x / GraalVM AOT
- 🛡️ [**Spring Security & JWT**](spring-framework/spring-security.md) — Filter chain architecture, Authentication/Authorization pipeline, complete JWT implementation, OAuth2/OIDC, CORS, CSRF, Password encoding, Spring Security 6.x migration
- 🌐 [**REST API Best Practices**](spring-framework/rest-api-best-practices.md) — Richardson Maturity Model, API versioning, Pagination (offset vs cursor), Error handling (RFC 7807), Validation, HATEOAS, Rate limiting, Idempotency, DTOs, SpringDoc/Swagger
- 🔐 [**JPA Isolation Levels**](spring-framework/jpa-isolation-levels.md) — READ_UNCOMMITTED to SERIALIZABLE, dirty/phantom reads, @Transactional, propagation types
- 🎭 [**AOP (Aspect-Oriented Programming)**](spring-framework/aop.md) — Aspects, Advice types, Pointcuts, Spring AOP vs AspectJ, practical examples
- 🔄 [**Optimistic vs Pessimistic Locking**](spring-framework/optimistic-vs-pessimistic-locking.md) — JPA @Version, SELECT FOR UPDATE, retry strategies, Spring Data JPA examples
- 🏗️ [**Microservices Patterns**](spring-framework/microservices-patterns.md) — API Gateway, Service Discovery, Config Server, OpenFeign, Distributed Tracing, CQRS, Event Sourcing, Outbox Pattern, Strangler Fig, 12-Factor App

---

## 🗄️ Database & SQL (PostgreSQL)

PostgreSQL internals, performance tuning, and advanced SQL for backend developers.

- 🐘 [**PostgreSQL Internals & Performance**](database/postgres-internals-and-performance.md) — MVCC (xmin/xmax), WAL, VACUUM & autovacuum, Indexing (B-Tree, GIN, GiST, BRIN, Partial, Covering), EXPLAIN ANALYZE, Query planning, Performance tuning, Partitioning, Replication, N+1 problem
- 📊 [**Advanced SQL**](database/advanced-sql.md) — Window functions, Recursive CTEs, LATERAL joins, JSON/JSONB operations, Full-text search, GROUPING SETS/ROLLUP/CUBE, UPSERT, generate_series, classic interview problems

---

## 🏗️ Design Patterns & Principles

All 23 Gang of Four patterns + architectural patterns with complete Java code examples.

- 📐 [**Design Patterns — Complete Reference**](design-patterns/design-patterns-overview.md) — All 23 GoF patterns (Creational, Structural, Behavioral) with compilable Java code for each: Singleton, Factory, Builder, Adapter, Decorator, Proxy, Strategy, Observer, and more
- 💎 [**SOLID Principles**](design-patterns/solid-principles.md) — SRP, OCP, LSP, ISP, DIP with bad vs good code examples and real-world analogies
- 🔌 [**Circuit Breaker & Fallback**](design-patterns/circuit-breaker-and-fallback.md) — States, sliding windows, Resilience4j, Spring Cloud Circuit Breaker, monitoring
- 🔀 [**Saga Pattern**](design-patterns/saga-pattern.md) — Choreography vs Orchestration, compensating transactions, Outbox + Debezium, vs 2PC

---

## 📨 Messaging & Event Streaming

Distributed messaging systems and event-driven architecture.

- 📡 [**Kafka vs Pub/Sub**](messaging/kafka-vs-pubsub.md) — Architecture deep-dive, delivery guarantees, ordering, retention, Java producer/consumer code examples

---

## 🚀 How to Use

1. **Quick Review**: Use the [Topics at a Glance](#️-all-topics-at-a-glance) table to jump to any topic
2. **Deep Dive**: Each page has concept explanation → code examples → interview Q&A → gotchas
3. **Before Interview**: Focus on the **Interview Questions** section at the bottom of each page
4. **Senior Focus**: Topics marked with Spring Boot, PostgreSQL, and Microservices are especially relevant for 8+ year roles

---

## 📋 Interview Prep Checklist — Senior Java Developer

| Area | Must-Know Topics |
|------|-----------------|
| **Java Core** | [Java Features](java-internals/java-features.md), [Concurrency](java-internals/concurrency-advanced.md), [Functional Interface](java-internals/functional-interface.md), [Heap Architecture](java-internals/heap-space-architecture.md) |
| **Spring Boot** | [Boot Internals](spring-framework/spring-boot-internals.md), [Security](spring-framework/spring-security.md), [REST APIs](spring-framework/rest-api-best-practices.md), [AOP](spring-framework/aop.md) |
| **Database** | [PostgreSQL](database/postgres-internals-and-performance.md), [Advanced SQL](database/advanced-sql.md), [JPA/Locking](spring-framework/optimistic-vs-pessimistic-locking.md), [Isolation Levels](spring-framework/jpa-isolation-levels.md) |
| **Architecture** | [Microservices](spring-framework/microservices-patterns.md), [Design Patterns](design-patterns/design-patterns-overview.md), [SOLID](design-patterns/solid-principles.md), [Caching](java-internals/caching-strategies.md) |
| **Resilience** | [Circuit Breaker](design-patterns/circuit-breaker-and-fallback.md), [Saga](design-patterns/saga-pattern.md), [Kafka](messaging/kafka-vs-pubsub.md) |
| **DSA** | [HashMap](data-structures/hashmap-and-red-black-tree.md), [LRU Cache](data-structures/lru-cache.md), [Tree Traversals](data-structures/tree-traversals.md) |

---

> 💡 *Each topic includes a `← Back to Index` link at the top for easy navigation back to this page.*

[← Back to Main Site](../index.html)
