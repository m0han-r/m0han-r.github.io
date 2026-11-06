← [Back to Index](../README.md)

# Java HashMap Internals & Red-Black Tree Treeification — Interview Guide

## 1. Concept Overview

`java.util.HashMap<K, V>` is a hash-table-based implementation of the `Map` interface. It stores key-value pairs using a bucket array (`Node<K, V>[] table`) and provides average **$O(1)$** time complexity for basic operations (`get` and `put`).

Starting in **Java 8**, Java introduced a critical architectural enhancement: when bucket collision chains become excessively long, the bucket converts from a singly linked list to a self-balancing **Red-Black Tree**, reducing the worst-case lookup time from **$O(n)$** down to **$O(\log n)$**.

```
HashMap Internal Bucket Array (table):
Index:
 [0]  ---> [ Node A ] -> null
 [1]  ---> null
 [2]  ---> [ Node B ] -> [ Node C ] -> [ Node D ] -> null (Chaining)
 [3]  ---> null
 ...
 [7]  ---> [ TreeNode Root ]  <-- Treeification (Red-Black Tree)
            /             \
      [ TreeNode L ]   [ TreeNode R ]
```

---

## 2. Hashing Mechanism & Index Calculation

When a key-value pair is inserted via `map.put(key, value)`, HashMap executes three distinct mathematical steps to determine the bucket index.

```
+---------------+      +---------------------------------+      +------------------------+
| key.hashCode()| ---> | Hash Perturbation (Spread bits) | ---> | Bucket Index           |
| (32-bit int)  |      | h ^ (h >>> 16)                  |      | (n - 1) & hash         |
+---------------+      +---------------------------------+      +------------------------+
```

### Step 1: Raw Hash Code
```java
int h = key.hashCode(); // Generates 32-bit integer (-2^31 to 2^31 - 1)
```

### Step 2: Perturbation Function (Bit-Spreading)
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
- **Why?** Many bucket tables are small (e.g., initial size 16). Without perturbation, only the lowest 4 bits ($16 - 1 = 15 = 00001111_2$) are used to compute the index.
- If two distinct keys have different hash codes in their top 16 bits but identical lowest 4 bits, they will collide 100% of the time.
- By shifting the upper 16 bits down by 16 positions (`h >>> 16`) and XORing with the original hash (`h ^ (h >>> 16)`), the higher-order bits are mixed into the lower-order bits, drastically minimizing collisions for small table sizes.

### Step 3: Fast Index Calculation `(n - 1) & hash`
```java
int index = (n - 1) & hash; // where n is the capacity of table (power of 2)
```
- **Why bitwise AND instead of Modulo (`hash % n`)?**
  - Modulo arithmetic (`%`) requires CPU division, which takes ~20–40 CPU cycles.
  - Bitwise AND (`&`) takes only 1 CPU cycle.
- **Why MUST capacity $n$ be a power of 2 ($2^k$)?**
  - When $n = 2^k$, $n - 1$ in binary is composed entirely of $k$ consecutive `1` bits.
  - Example ($n = 16 = 10000_2 \implies n - 1 = 15 = 01111_2$):
    $$hash \ \& \ 01111_2 \equiv hash \pmod{16}$$
  - If $n$ were not a power of 2 (e.g. 15), $n - 1 = 14 = 01110_2$. The lowest bit is 0, meaning no key could ever hash to an odd bucket index! Half of all buckets would remain permanently empty.

---

## 3. Collision Handling & Treeification (Java 7 vs Java 8+)

### 3.1 Java 7: Pure Separate Chaining
- Buckets were exclusively singly linked lists.
- New entries were inserted at the **HEAD** of the list (assumed newly added items were accessed most frequently).
- **Flaws**:
  - Head-insertion during concurrent resizing could reverse list pointers and cause an **infinite circular loop** (100% CPU lockup).
  - Malicious attackers could craft inputs with identical hash codes (Hash DoS attack), forcing all entries into a single bucket with $O(n)$ degradation.

### 3.2 Java 8+: Chaining + Red-Black Treeification
- New elements are inserted at the **TAIL** of the list (preserves node order, eliminates cyclic pointer race during resize).
- If the chain length exceeds a threshold, the bucket converts to a balanced Red-Black Tree (`TreeNode`).

```
                    Bucket Growth in Java 8+
                    
  Length <= 8                   Length > 8 & Capacity >= 64
+-------------+               +-------------------------------+
| Linked List |  =========>   | Red-Black Tree                |
| O(n) lookup |  Treeification| O(log n) guaranteed worst-case|
+-------------+               +-------------------------------+
       ^                                     |
       |                                     | Shrink <= 6 (Untreeify)
       +-------------------------------------+
```

### 3.3 The Core HashMap Threshold Constants

| Constant | Value | Purpose / Rationale |
|---|---|---|
| `DEFAULT_INITIAL_CAPACITY` | `16` (`1 << 4`) | Default number of buckets; must be a power of 2. |
| `MAXIMUM_CAPACITY` | `1 << 30` | Maximum allowed table capacity ($2^{30} \approx 1.07$ billion). |
| `DEFAULT_LOAD_FACTOR` | `0.75f` | Default load factor trade-off between space and time. |
| `TREEIFY_THRESHOLD` | `8` | Chain length at which a bucket converts to a Red-Black Tree. |
| `UNTREEIFY_THRESHOLD` | `6` | Tree size at which a bucket converts back to a Linked List. |
| `MIN_TREEIFY_CAPACITY` | `64` | Minimum table capacity required before treeification is allowed. |

### 3.4 Why 8 and 6? (The Math Behind It)
Under a well-distributed hash function, the probability of collisions follows a **Poisson distribution**:
$$P(k) = \frac{e^{-\lambda} \cdot \lambda^k}{k!}$$
With default load factor 0.75, $\lambda \approx 0.5$.
The probability of a single bucket receiving $k$ collisions:
- $k = 0$: $0.6065$
- $k = 1$: $0.3032$
- $k = 2$: $0.0758$
- $k = 3$: $0.0126$
- $k = 4$: $0.0016$
- $k = 5$: $0.00016$
- $k = 6$: $0.000013$
- $k = 7$: $0.0000009$
- **$k = 8$: $0.00000006$ (less than 1 in 10 million!)**

Thus, treeification is an emergency defensive mechanism against poor hash functions or DoS attacks.

**Why untreeify at 6 instead of 8?**
- To introduce a **hysteresis buffer** (gap of 2).
- If `UNTREEIFY_THRESHOLD` were 7 or 8, repeatedly adding and removing an element around boundary size 8 would cause constant expensive flipping between linked list and Red-Black tree ("thrashing").

**Why `MIN_TREEIFY_CAPACITY = 64`?**
- If the table size is small (e.g. 16 or 32), collisions are more likely due to table shortage rather than hash code saturation.
- Resizing the table (`resize()`) spreads collisions more effectively and is cheaper than maintaining tree nodes.

---

## 4. Capacity, Load Factor, and Java 8 Rehashing Optimization

### 4.1 Load Factor & Threshold
$$\text{Threshold} = \text{Capacity} \times \text{Load Factor}$$
- When `size > threshold`, `resize()` is triggered (capacity doubles: $16 \to 32 \to 64 \dots$).
- `0.75` load factor represents the mathematical sweet spot between memory space overhead and cache hit / lookup latency.

### 4.2 Java 8 High/Low Bit Rehashing (Zero Hash Recomputation)
In Java 7, rehashing required recomputing `hash % newCapacity` for every single element.
In Java 8, because capacity always doubles ($newCap = oldCap \times 2$), the capacity power-of-2 increases by exactly 1 bit.
When computing `(newCap - 1) & hash`:
- The new bit in `(newCap - 1)` is either `0` or `1`.
- If the corresponding bit in `hash` is `0`: `index` remains **identical** (`index = oldIndex`).
- If the corresponding bit in `hash` is `1`: `index` shifts to **`oldIndex + oldCap`**.

```
Example: oldCap = 16 (bitmask 01111), newCap = 32 (bitmask 11111)
Bit check: hash & oldCap (i.e. hash & 16)

If (hash & 16) == 0:
  newIndex = oldIndex
If (hash & 16) != 0:
  newIndex = oldIndex + 16
```
This splits each collision bucket into two orderly linked lists (`loHead`/`loTail` and `hiHead`/`hiTail`) in $O(1)$ operations per node without recomputing hashes.

---

## 5. Thread Safety Concerns

`HashMap` is **not thread-safe**.

### 5.1 Java 7 Race Condition: The Infinite Loop Bug
In Java 7's `transfer()` method during resize:
```java
void transfer(Entry[] newTable, boolean rehash) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next; // Thread 1 paused here!
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];     // Head-insertion reverses order!
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```
If Thread 1 pauses right before moving nodes while Thread 2 completes resizing, the reversal of links creates a circular reference: `A.next = B` and `B.next = A`. When a future `get()` searches that bucket, it enters an **infinite loop**, pinning the CPU at 100%.

### 5.2 Java 8 Concurrency Issues
Although Java 8 eliminates the circular loop via tail-insertion, it is **still unsafe for concurrent use**:
1. **Lost Updates**: If two threads execute `put()` simultaneously in the same empty bucket, both read `null` and write their node; the second thread overwrites the first thread's node.
2. **Size Inconsistency**: `size++` is not atomic, leading to incorrect element counts.
3. **Corrupted Red-Black Tree**: Concurrent tree rebalancing rotations can lead to cycles or null pointers.

### 5.3 Thread-Safe Alternatives

| Map Implementation | Concurrency Mechanism | Performance / Scalability |
|---|---|---|
| `HashMap` | None (Non-thread-safe) | Fastest for single thread |
| `Hashtable` | Synchronized methods (coarse lock on entire table) | Very poor under contention |
| `Collections.synchronizedMap()` | Synchronized wrapper object | Poor under contention |
| `ConcurrentHashMap` (Java 8+) | CAS operations + synchronized bucket head locks + lock-free reads | High throughput and scalability |

---

## 6. The `equals()` and `hashCode()` Contract

When custom objects are used as HashMap keys, you **must override both `equals()` and `hashCode()`**.

```java
public class Employee {
    private final int id;
    private final String name;

    public Employee(int id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Employee employee = (Employee) o;
        return id == employee.id && java.util.Objects.equals(name, employee.name);
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(id, name);
    }
}
```

### The 3 Contract Rules:
1. **Consistency**: If the object has not changed, multiple calls to `hashCode()` must consistently return the same integer.
2. **Equal Objects Must Have Equal Hash Codes**: If `a.equals(b) == true`, then `a.hashCode() == b.hashCode()` MUST be true.
3. **Unequal Objects May Share Hash Codes**: If `a.equals(b) == false`, `a.hashCode()` does not have to be distinct (a hash collision).

### The Mutable Key Danger:
If a key is mutated after insertion such that its `hashCode()` changes:
- `map.get(key)` calculates a new bucket index and searches the wrong bucket.
- The entry becomes an **unreachable memory leak** inside the map.
- **Rule of Thumb**: Always make HashMap keys **immutable** (e.g. `final` fields, `record`, `String`, `Integer`).

---

## 7. Step-by-Step `put()` Logic Flowchart

```
                 map.put(key, value)
                          |
                          v
                   Calculate hash
             (key.hashCode() ^ (h >>> 16))
                          |
                          v
                 table == null or empty?
                 /                     \
             YES                        NO
             /                            \
      Call resize()                 Calculate bucket index
                                     i = (n - 1) & hash
                                           |
                                  table[i] == null?
                                  /               \
                              YES                  NO
                              /                      \
                     Create Node at table[i]     Collision!
                                                      |
                                           Is key equal to first node?
                                           /                         \
                                       YES                            NO
                                       /                                \
                            Overwrite value               table[i] instanceof TreeNode?
                                                          /                           \
                                                      YES                              NO
                                                      /                                  \
                                       Insert into RB Tree                     Iterate Linked List
                                       (or update value)                       (Tail Insertion)
                                                                                         |
                                                                               Chain length >= 8?
                                                                               /                \
                                                                           YES                   NO
                                                                           /                       \
                                                          table.length >= 64?               Normal append
                                                          /                 \
                                                      YES                    NO
                                                      /                        \
                                               treeifyBin()                 resize()
                                                      |
                                      Check if size > threshold -> resize()
```

---

## 8. Time & Space Complexity Summary

| Operation | Average Case | Worst Case (Java 7) | Worst Case (Java 8+) |
|---|---|---|---|
| `put(K, V)` | $O(1)$ | $O(n)$ | $O(\log n)$ |
| `get(Object K)` | $O(1)$ | $O(n)$ | $O(\log n)$ |
| `remove(Object K)` | $O(1)$ | $O(n)$ | $O(\log n)$ |
| `containsKey(Object K)` | $O(1)$ | $O(n)$ | $O(\log n)$ |
| Iteration | $O(n + \text{capacity})$ | $O(n + \text{capacity})$ | $O(n + \text{capacity})$ |
| Space Complexity | $O(n)$ | $O(n)$ | $O(n)$ (TreeNodes use $\approx 2\times$ memory of Nodes) |

---

## 9. Common Interview Questions & Gotchas

### Q1: Walk me through what happens when `map.get(key)` is invoked.
1. Compute hash: `h = (key == null) ? 0 : hash(key)`.
2. Compute index: `index = (n - 1) & h`.
3. Check bucket `first = table[index]`. If `first == null`, return `null`.
4. If `first.hash == h` and `(first.key == key || key.equals(first.key))`, return `first.value`.
5. If `first` is collision node:
   - If `first instanceof TreeNode`, call `treeNode.getTreeNode(h, key)` ($O(\log n)$).
   - Else, iterate linked list using `equals()` check ($O(k)$).
6. If no match found, return `null`.

### Q2: What happens if two distinct keys produce the exact same `hashCode()`?
- Both keys hash to the same bucket index (collision).
- HashMap stores both entries in that bucket:
  - First compares keys using reference equality (`k1 == k2`).
  - Then compares using `k1.equals(k2)`.
  - Since `equals()` returns `false`, they are retained as separate nodes in the linked list or Red-Black Tree.

### Q3: Why does HashMap not use a Red-Black Tree for all buckets from the beginning?
- Memory overhead: A regular `Node` stores 4 fields (`hash, key, value, next`). A `TreeNode` extends `LinkedHashMap.Entry` and stores parent, left, right, prev, and red/black boolean, consuming more than **double the memory**.
- Tree balance rotations also incur performance overhead for small node counts.
- With good hashing, the Poisson probability of having $> 2$ collisions is under $1\%$. Using lists first preserves lean memory usage.
