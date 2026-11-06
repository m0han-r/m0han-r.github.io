← [Back to Index](../README.md)

# Java TreeMap & NavigableMap — Interview Guide

## 1. Concept Overview

`java.util.TreeMap<K, V>` is a **Red-Black Tree** based implementation of Java's `NavigableMap` interface. Keys are ordered according to their **natural ordering** (implementing `Comparable<K>`) or by an explicit `Comparator<? super K>` provided at map creation time.

Unlike `HashMap` which offers $O(1)$ average time complexity through hash buckets, `TreeMap` guarantees **$O(\log n)$** time cost for `containsKey`, `get`, `put`, and `remove` operations while keeping all keys sorted at all times.

### Class Hierarchy
```
               Map<K, V>
                  ^
                  |
             SortedMap<K, V>
                  ^
                  |
            NavigableMap<K, V>
                  ^
                  |
             TreeMap<K, V>
```

---

## 2. Internal Architecture: Red-Black Tree

`TreeMap` stores its entries as nodes in a self-balancing binary search tree called a **Red-Black Tree**.

### 2.1 The 5 Red-Black Tree Invariants
To guarantee that the tree remains balanced and the height never exceeds $2 \log_2(n + 1)$:
1. **Node Color**: Every node is either **RED** or **BLACK**.
2. **Root Property**: The root is always **BLACK**.
3. **Leaf Property**: Every leaf (`NIL` / `null`) is considered **BLACK**.
4. **Red Property**: If a node is **RED**, both its children must be **BLACK** (no two consecutive red nodes on any path).
5. **Black-Height Property**: For every node, all paths from that node to descendant `null` leaves must contain the exact same number of black nodes.

```
                    [ 20 : B ]
                   /          \
         [ 10 : R ]            [ 30 : B ]
        /          \          /          \
   [ 5 : B ]   [ 15 : B ] [ 25 : R ]   [ 35 : R ]
   /      \     /      \   /      \     /      \
  NIL    NIL   NIL    NIL NIL    NIL   NIL    NIL
```

### 2.2 Internal Node Structure (`TreeMap.Entry<K, V>`)
```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK; // true = BLACK, false = RED
}
```

### 2.3 Why Red-Black Tree over AVL Tree?
| Feature | Red-Black Tree (TreeMap) | AVL Tree |
|---|---|---|
| **Balance Strictness** | Looser: Height $\le 2 \log_2(n+1)$ | Stricter: Height $\le 1.44 \log_2(n)$ |
| **Lookup Performance** | Fast ($O(\log n)$) | Slightly faster (more balanced) |
| **Insertion Rotations** | $\le 2$ rotations | $\le 2$ rotations |
| **Deletion Rotations** | $\le 3$ rotations | Up to $O(\log n)$ rotations |
| **Use Case Fit** | Preferred for general-purpose maps with frequent inserts/deletes | Preferred for read-heavy lookup tables |

---

## 3. NavigableMap Interface in Depth

`NavigableMap` extends `SortedMap` with directional search and range queries.

```
                     NavigableMap Methods
   +---------------------------+---------------------------+
   |   Exact / Boundary        |   Nearest Matches         |
   |---------------------------|---------------------------|
   | firstKey() / lastKey()    | floorKey(k)   (<= k)      |
   | firstEntry() / lastEntry()| lowerKey(k)   (< k)       |
   | pollFirstEntry()          | ceilingKey(k) (>= k)      |
   | pollLastEntry()           | higherKey(k)  (> k)       |
   +---------------------------+---------------------------+
   |   Range Views             |   Descending Views        |
   |---------------------------|---------------------------|
   | subMap(from, fInc, to, tInc)| descendingMap()         |
   | headMap(to, inclusive)    | descendingKeySet()        |
   | tailMap(from, inclusive)  |                           |
   +---------------------------+---------------------------+
```

### Visualizing Nearest Match Lookups
Given keys: `[10, 20, 30, 40]`, querying around key `25`:

```
                20          25          30
                 |           |           |
lowerKey(25)  ---+           |           |  (< 25)  -> 20
floorKey(25)  ---+           |           |  (<= 25) -> 20
ceilingKey(25)---------------+-----------+  (>= 25) -> 30
higherKey(25) ---------------+-----------+  (> 25)  -> 30
```
Querying exact key `20`:
- `lowerKey(20)` $\to$ `10`
- `floorKey(20)` $\to$ `20`
- `ceilingKey(20)` $\to$ `20`
- `higherKey(20)` $\to$ `30`

---

## 4. Comparison: TreeMap vs HashMap vs LinkedHashMap

| Feature | `HashMap` | `LinkedHashMap` | `TreeMap` |
|---|---|---|---|
| **Underlying Structure** | Hash Table + Linked List / RB Tree | Hash Table + Doubly Linked List | Red-Black Tree |
| **Ordering** | None (arbitrary / nondeterministic) | Insertion-order or Access-order | Natural or Comparator sorted order |
| **`get` / `put` / `remove`** | $O(1)$ average | $O(1)$ average | $O(\log n)$ guaranteed |
| **`containsKey`** | $O(1)$ average | $O(1)$ average | $O(\log n)$ guaranteed |
| **Null Keys** | Allowed (1 `null` key at bucket 0) | Allowed (1 `null` key) | **Not allowed** (throws `NullPointerException`)* |
| **Null Values** | Allowed | Allowed | Allowed |
| **Key Comparison** | `hashCode()` & `equals()` | `hashCode()` & `equals()` | `compareTo()` or `compare()` |
| **Memory Overhead** | Low to moderate | Moderate (extra before/after pointers) | Higher (3 pointers + 1 color bit per node) |
| **Thread Safety** | Non-thread-safe | Non-thread-safe | Non-thread-safe |

*\*Note: In Java 7+, passing `null` to `put(null, val)` throws `NullPointerException` even on an empty map.*

---

## 5. When to Use TreeMap

### Ideal Use Cases:
1. **Sorted Output**: When entries must be iterated in natural or custom sorted order.
2. **Nearest-Neighbor Lookups**: When you need "closest match" operations (`floor`, `ceiling`).
3. **Range Queries**: Querying items within a time interval, price range, or prefix (e.g. `subMap("A", "M")`).
4. **Priority / Leaderboard Systems**: Dynamic rankings where both lookups and boundary extractions are required.

### When NOT to Use:
- When sorted order is unnecessary $\to$ use `HashMap` (much faster $O(1)$ operations).
- When insertion/access order is sufficient $\to$ use `LinkedHashMap`.
- Highly concurrent environments without proper synchronizers $\to$ use `ConcurrentSkipListMap`.

---

## 6. Comprehensive Java Code Examples

### 6.1 Basic Operations & Navigable Queries

```java
import java.util.Map;
import java.util.NavigableMap;
import java.util.TreeMap;

public class TreeMapDemo {

    public static void main(String[] args) {
        NavigableMap<Integer, String> scores = new TreeMap<>();

        scores.put(50, "Alice");
        scores.put(85, "Bob");
        scores.put(70, "Charlie");
        scores.put(95, "Diana");
        scores.put(60, "Evan");

        // 1. Natural In-Order Traversal: {50=Alice, 60=Evan, 70=Charlie, 85=Bob, 95=Diana}
        System.out.println("Sorted Map: " + scores);

        // 2. Boundary Elements
        System.out.println("Lowest Score: " + scores.firstEntry()); // 50=Alice
        System.out.println("Highest Score: " + scores.lastEntry()); // 95=Diana

        // 3. Nearest Match Lookups
        int target = 75;
        System.out.println("Floor of 75: " + scores.floorEntry(target));     // 70=Charlie
        System.out.println("Lower of 75: " + scores.lowerEntry(target));     // 70=Charlie
        System.out.println("Ceiling of 75: " + scores.ceilingEntry(target)); // 85=Bob
        System.out.println("Higher of 75: " + scores.higherEntry(target));   // 85=Bob

        // 4. Sub-range Views (60 inclusive to 85 inclusive)
        NavigableMap<Integer, String> range = scores.subMap(60, true, 85, true);
        System.out.println("Scores 60 to 85: " + range); // {60=Evan, 70=Charlie, 85=Bob}

        // 5. Descending View
        NavigableMap<Integer, String> reversed = scores.descendingMap();
        System.out.println("Reversed: " + reversed);
    }
}
```

### 6.2 Custom Comparator Example

```java
import java.util.Comparator;
import java.util.TreeMap;

class Student {
    String name;
    int rollNumber;

    Student(String name, int rollNumber) {
        this.name = name;
        this.rollNumber = rollNumber;
    }

    @Override
    public String toString() {
        return name + "(#" + rollNumber + ")";
    }
}

public class CustomComparatorExample {
    public static void main(String[] args) {
        // Sort students by roll number descending
        Comparator<Student> rollDescComparator = 
            (s1, s2) -> Integer.compare(s2.rollNumber, s1.rollNumber);

        TreeMap<Student, String> studentMap = new TreeMap<>(rollDescComparator);
        studentMap.put(new Student("John", 103), "Grade A");
        studentMap.put(new Student("Sarah", 101), "Grade A+");
        studentMap.put(new Student("Mike", 105), "Grade B");

        studentMap.forEach((student, grade) -> 
            System.out.println(student + " -> " + grade)
        );
        // Output order: Mike(#105), John(#103), Sarah(#101)
    }
}
```

### 6.3 Real-World Problem: Meeting Room / Interval Scheduling

```java
import java.util.TreeMap;

public class CalendarScheduler {

    // Key: startTime, Value: endTime
    private final TreeMap<Integer, Integer> calendar = new TreeMap<>();

    /**
     * Book a new event [start, end) without overlap.
     * Returns true if booking succeeds, false if overlapping.
     * Time Complexity: O(log n)
     */
    public boolean book(int start, int end) {
        // Check event that starts before or at 'start'
        Integer prevStart = calendar.floorKey(start);
        if (prevStart != null && calendar.get(prevStart) > start) {
            return false; // Overlap with previous event
        }

        // Check event that starts after or at 'start'
        Integer nextStart = calendar.ceilingKey(start);
        if (nextStart != null && nextStart < end) {
            return false; // Overlap with next event
        }

        calendar.put(start, end);
        return true;
    }
}
```

---

## 7. Time & Space Complexity Summary

| Operation | Time Complexity | Space Complexity | Description |
|---|---|---|---|
| `put(K, V)` | $O(\log n)$ | $O(1)$ | Tree search + balance rotations |
| `get(Object K)` | $O(\log n)$ | $O(1)$ | Binary search tree traversal |
| `remove(Object K)` | $O(\log n)$ | $O(1)$ | Node removal + fix-up rotations |
| `containsKey(Object K)` | $O(\log n)$ | $O(1)$ | BST search |
| `firstKey()` / `lastKey()` | $O(\log n)$ | $O(1)$ | Walk down to leftmost / rightmost node |
| `floorKey()` / `ceilingKey()` | $O(\log n)$ | $O(1)$ | Standard binary search tree path traversal |
| `subMap()` / `headMap()` | $O(1)$ | $O(1)$ | Returns a view backed by original map |
| Full Map Iteration | $O(n)$ | $O(\log n)$ stack | In-order traversal |
| **Total Memory Space** | — | $O(n)$ | 1 Entry node per key-value pair |

---

## 8. Common Interview Questions & Gotchas

### Q1: How does TreeMap define key equality? Does it call `.equals()`?
- **Answer**: **NO!** `TreeMap` does **NOT** use `equals()`. It uses `comparator.compare(k1, k2) == 0` or `k1.compareTo(k2) == 0`.
- **The Gotcha**: If a custom class implements `Comparable` where `compareTo` returns 0 for two distinct objects that have different `equals()` outputs, `TreeMap` treats them as the same key and overwrites the value!
  ```java
  // Example Gotcha:
  class Person implements Comparable<Person> {
      String name;
      int age;
      // compareTo only compares age!
      public int compareTo(Person o) { return Integer.compare(this.age, o.age); }
  }
  // Two people with the same age will overwrite each other in TreeMap!
  ```
- **Best Practice**: Always keep `compareTo()` **consistent with `equals()`** according to Java's `Comparable` contract.

### Q2: Why does `TreeMap` not allow `null` keys?
- **Answer**: To place a key in the Red-Black tree, `TreeMap` must compare it against existing nodes via `key.compareTo(otherKey)` or `comparator.compare(key, otherKey)`. Calling `compareTo()` on `null` triggers a `NullPointerException`.
- *Exception*: A custom comparator that explicitly permits and handles `null` values can technically allow one `null` key, but standard usage strictly disallows it.

### Q3: How do you make a `TreeMap` thread-safe?
- **Answer**: `TreeMap` is non-synchronized. Options:
  1. Wrap it: `NavigableMap<K, V> syncMap = Collections.synchronizedNavigableMap(new TreeMap<>());`
     *(Requires manual synchronization block during iteration).*
  2. Use **`ConcurrentSkipListMap<K, V>`**: A lock-free, concurrent, sorted map based on Skip Lists that provides $O(\log n)$ operations with high concurrent throughput.

### Q4: What is the difference between `lowerKey()` and `floorKey()`?
- **Answer**:
  - `lowerKey(K)`: Strictly **less than** $K$ ($< K$).
  - `floorKey(K)`: **Less than or equal to** $K$ ($\le K$).
  - Analogously: `higherKey(K)` ($> K$) vs `ceilingKey(K)` ($\ge K$).
