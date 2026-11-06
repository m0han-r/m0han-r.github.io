← [Back to Index](../README.md)

# LRU Cache (Least Recently Used) — Comprehensive Interview Guide

## 1. Concept Overview

An **LRU (Least Recently Used) Cache** is a fixed-capacity in-memory data store that organizes items in order of their usage. When the cache reaches its maximum capacity, it **evicts the least recently accessed item** to make room for new elements.

### Core Philosophy: Temporal Locality
Programs tend to access data repeatedly within short time intervals. By keeping recently accessed elements close at hand, an LRU cache maximizes cache hits and minimizes expensive database or network calls.

### API Requirements (LeetCode #146)
All operations must execute in strictly **$O(1)$ average time**:
- `get(key)`: Return value associated with `key` if present; otherwise return `-1` (or `null`). Accessing the key marks it as **Most Recently Used (MRU)**.
- `put(key, value)`: Insert or update key-value pair. Marks it as MRU. If capacity is exceeded, evict the **Least Recently Used (LRU)** key.

---

## 2. Anatomy of the $O(1)$ LRU Cache: HashMap + Doubly Linked List

To achieve $O(1)$ for both `get` and `put`, we combine two complementary data structures:

```
                            LRU Cache Architecture
                            
  HashMap (Key -> Node reference)
  +-------+------------+
  | Key 1 | -> [Node 1]|
  | Key 2 | -> [Node 2]|
  | Key 3 | -> [Node 3]|
  +-------+------------+
            |        |        |
            v        v        v
  Doubly Linked List (Ordering by recency)
  
  +--------------+       +----------+       +----------+       +--------------+
  |  Dummy Head  | <---> |  Node 3  | <---> |  Node 1  | <---> |  Dummy Tail  |
  |  (MRU Side)  |       |  (MRU)   |       |  (LRU)   |       |  (LRU Side)  |
  +--------------+       +----------+       +----------+       +--------------+
```

### Why this combination?
| Data Structure Candidate | Lookup Time | Removal of an Element | Append / Move to Front | Verdict |
|---|---|---|---|---|
| **Array / ArrayList** | $O(1)$ by index, $O(n)$ by key | $O(n)$ (shifting elements) | $O(1)$ at end, $O(n)$ at front | ❌ Inefficient |
| **Singly Linked List** | $O(n)$ search | $O(n)$ (cannot find `prev` in $O(1)$) | $O(1)$ | ❌ Cannot delete in $O(1)$ |
| **HashMap alone** | $O(1)$ lookup | $O(1)$ | No inherent access order | ❌ Cannot track recency |
| **HashMap + Doubly Linked List** | **$O(1)$** via Map | **$O(1)$** via `node.prev` & `node.next` | **$O(1)$** insert after head | ✅ **Optimal $O(1)$** |

### Why Dummy Head & Dummy Tail Sentinels?
Using pseudo-head and pseudo-tail nodes prevents tedious `null` pointer checks when:
- Inserting into an empty list.
- Removing the single remaining node.
- Updating boundary elements.

---

## 3. Implementation 1: Custom Doubly Linked List + HashMap (Standard Interview Approach)

```java
import java.util.HashMap;
import java.util.Map;

public class LRUCache {

    // Node representation in Doubly Linked List
    private static class DNode {
        int key;
        int value;
        DNode prev;
        DNode next;

        DNode() {}
        DNode(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private final int capacity;
    private final Map<Integer, DNode> cache;
    private final DNode head;
    private final DNode tail;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>(capacity);

        // Initialize sentinel dummy nodes
        this.head = new DNode();
        this.tail = new DNode();
        head.next = tail;
        tail.prev = head;
    }

    /**
     * O(1) Fetch: Retrieves node, updates its recency to Head.
     */
    public int get(int key) {
        DNode node = cache.get(key);
        if (node == null) {
            return -1;
        }
        moveToHead(node);
        return node.value;
    }

    /**
     * O(1) Store: Inserts or updates node. Evicts LRU if capacity exceeded.
     */
    public void put(int key, int value) {
        DNode node = cache.get(key);

        if (node != null) {
            // Update value and move to MRU head
            node.value = value;
            moveToHead(node);
        } else {
            DNode newNode = new DNode(key, value);
            cache.put(key, newNode);
            addNode(newNode);

            // Evict least recently used if over capacity
            if (cache.size() > capacity) {
                DNode lru = popTail();
                cache.remove(lru.key);
            }
        }
    }

    // ================= Internal DLL Helper Methods =================

    /**
     * Inserts new node right after dummy head (MRU position).
     */
    private void addNode(DNode node) {
        node.prev = head;
        node.next = head.next;

        head.next.prev = node;
        head.next = node;
    }

    /**
     * Unlinks an existing node from DLL.
     */
    private void removeNode(DNode node) {
        DNode prevNode = node.prev;
        DNode nextNode = node.next;

        prevNode.next = nextNode;
        nextNode.prev = prevNode;
    }

    /**
     * Moves an existing node to MRU head position.
     */
    private void moveToHead(DNode node) {
        removeNode(node);
        addNode(node);
    }

    /**
     * Removes and returns the least recently used node (node right before dummy tail).
     */
    private DNode popTail() {
        DNode lru = tail.prev;
        removeNode(lru);
        return lru;
    }
}
```

---

## 4. Implementation 2: Java's Built-in `LinkedHashMap`

In production, writing custom linked list pointers is often unnecessary because Java provides `java.util.LinkedHashMap`.

`LinkedHashMap` contains an overloaded constructor:
```java
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder)
```
- When `accessOrder = true`, iteration order is based on **order of access (least-recently accessed to most-recently accessed)** rather than insertion order.
- Overriding `removeEldestEntry(Map.Entry eldest)` triggers automatic eviction of the eldest entry whenever a new key is added.

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCacheLinkedHashMap<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCacheLinkedHashMap(int capacity) {
        // initialCapacity, loadFactor = 0.75f, accessOrder = true
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // Automatically evict eldest entry when size exceeds capacity
        return size() > capacity;
    }

    public static void main(String[] args) {
        LRUCacheLinkedHashMap<Integer, String> cache = new LRUCacheLinkedHashMap<>(3);
        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        cache.get(1);      // 1 is accessed -> [2, 3, 1]
        cache.put(4, "D"); // Evicts 2    -> [3, 1, 4]

        System.out.println(cache.keySet()); // Output: [3, 1, 4]
    }
}
```

---

## 5. Thread-Safe LRU Cache & Concurrency Bottlenecks

### 5.1 The Hidden Trap: "Every `get()` is a Mutation!"
In standard caches, read operations do not mutate state.
In an **LRU Cache**, every `get(key)` must move the accessed node to the MRU position in the Doubly Linked List. **Thus, every `get()` is a structural write operation.**

### 5.2 Thread-Safety Strategies

```java
// Approach A: Synchronized Wrapper
Map<K, V> syncCache = Collections.synchronizedMap(new LRUCacheLinkedHashMap<>(100));

// Approach B: Explicit ReentrantReadWriteLock (Cannot use ReadLock for get!)
// Because get() mutates the list, both get() and put() require the WRITE lock.
```

### 5.3 High-Throughput Production Solutions (e.g., Caffeine / Guava Cache)
To avoid lock contention where every read locks the entire cache:
1. **Striped Ring Buffers**: Reads record hits into thread-local ring buffers without blocking.
2. **Asynchronous Eviction**: A background maintenance thread reads access events in batches and updates the recency list.
3. **Window TinyLFU (W-TinyLFU)**: Used by **Caffeine**, delivering near-optimal hit ratios and millions of ops/sec.

---

## 6. Comparison of Cache Eviction Policies

| Policy | Name | Eviction Metric | Strengths | Weaknesses |
|---|---|---|---|---|
| **LRU** | Least Recently Used | Timestamp / recency of last access | Adapts quickly to workload shifts | Vulnerable to sequential scan pollution |
| **LFU** | Least Frequently Used | Total access frequency count | Retains popular items long-term | High overhead, stale items with high counts linger |
| **FIFO** | First-In, First-Out | Time item entered cache | Simplest $O(1)$ implementation | Evicts frequently used items simply because they are old |
| **2Q** | Two Queue | Recency + Frequency queue separation | Prevents one-off scan pollution | Moderate complexity |
| **W-TinyLFU** | Window TinyLFU | Frequency sketch + recency window | Near-optimal hit ratio, low memory | Complex implementation (used in Caffeine) |

---

## 7. Real-World Applications of LRU

1. **CPU Caches (L1 / L2 / L3)**: Hardware implementations use pseudo-LRU (tree-PLRU or bit-PLRU) to evict cache lines.
2. **Operating System Page Replacement**: The OS kernel swaps out physical memory pages to disk (swap space) when RAM fills up.
3. **Database Buffer Pools**:
   - **MySQL InnoDB**: Maintains an LRU list for the InnoDB Buffer Pool (with a midpoint insertion strategy to resist table-scan pollution).
   - **PostgreSQL**: Shared buffer pool replacement.
4. **Distributed Caches (Redis & Memcached)**:
   - Redis offers `volatile-lru` and `allkeys-lru` eviction options (using approximated LRU sampling).
5. **Web Browsers**: Cache downloaded images, stylesheets, and scripts in local storage.

---

## 8. Time & Space Complexity Summary

| Method | Time Complexity | Space Complexity | Description |
|---|---|---|---|
| `get(key)` | $O(1)$ average | $O(1)$ | Hash lookup + DLL pointer manipulation |
| `put(key, value)` | $O(1)$ average | $O(1)$ | Hash insert + DLL append + optional tail eviction |
| Eviction | $O(1)$ | $O(1)$ | Unlink `tail.prev` and delete from map |
| **Total Memory** | — | $O(C)$ | Where $C$ is the cache capacity |

---

## 9. Common Interview Questions & Gotchas

### Q1: Why must the Linked List be Doubly Linked instead of Singly Linked?
- **Answer**: To achieve $O(1)$ node removal during `moveToHead` or `popTail`.
- In a singly linked list, unlinking node `X` requires updating its predecessor: `prev.next = X.next`.
- Because a singly linked node does not maintain a reference to its predecessor, finding `prev` requires scanning from the head in **$O(n)$** time, breaking the $O(1)$ contract.

### Q2: Why store BOTH `key` and `value` inside the `DNode`?
- **Answer**: When the cache is full and the least recently used node is evicted via `popTail()`, we must also delete its entry from the `HashMap`:
  ```java
  DNode lru = popTail();
  cache.remove(lru.key); // Requires lru.key!
  ```
- If `DNode` only stored `value`, we would not know which key to remove from the `HashMap` without a linear search over the entire map!

### Q3: What is "Cache Pollution" in LRU and how can it be mitigated?
- **Answer**: A sequential scan over a large table or file (e.g. `SELECT * FROM huge_table;`) will access every item once, filling the LRU cache with entries that will never be requested again while evicting hot, frequently used items.
- **Mitigation**:
  - **Midpoint Insertion (InnoDB)**: New pages are inserted into the middle (5/8 point) of the LRU list rather than the head.
  - **LRU-2 / 2Q**: Require an item to be accessed twice before promoting it to the primary cache.
