← [Back to Index](../README.md)

# Cyclic & Circular Linked Lists — Interview Guide

## 1. Concept Overview

A **Circular Linked List (CLL)** is a variation of a linked list in which the last node points back to the first node (or another preceding node), creating a closed loop. Unlike standard singly or doubly linked lists where the terminal node points to `null`, a circular linked list has no `null` termination.

A **Cyclic Linked List** refers more broadly to any linked list that contains a cycle (loop), whether intentionally designed as a circular data structure or inadvertently formed as a bug/loop in a general linked list.

```
Standard Singly Linked List:
[ Head ] -> [ Node 1 ] -> [ Node 2 ] -> [ Node 3 ] -> null

Circular Singly Linked List:
+---> [ Head ] -> [ Node 1 ] -> [ Node 2 ] -> [ Node 3 ] ---+
|                                                          |
+----------------------------------------------------------+

Cyclic Singly Linked List (Cycle begins at Node 1):
[ Head ] -> [ Node 1 ] -> [ Node 2 ] -> [ Node 3 ]
               ^                           |
               +---------------------------+
```

---

## 2. Types of Circular Linked Lists

### 2.1 Circular Singly Linked List (CSLL)
Each node contains data and a `next` pointer. The `next` pointer of the last node points to the head node.

```
       +---------------------------------------------+
       |                                             |
       v                                             |
+-------------+     +-------------+     +------------+--+
| data | next | --> | data | next | --> | data |   next |
+-------------+     +-------------+     +---------------+
      Head                                    Tail
```

### 2.2 Circular Doubly Linked List (CDLL)
Each node contains data, a `next` pointer, and a `prev` pointer.
- `head.prev` points to `tail`
- `tail.next` points to `head`

```
       +---------------------------------------------------------+
       |                                                         |
       |    +-----------------------------------------------+    |
       |    |                                               |    |
       v    v                                               |    |
+----+--------+----+      +----+--------+----+      +----+--+-----+----+
|prev|  data  |next| <--> |prev|  data  |next| <--> |prev|  data  |next|
+----+--------+----+      +----+--------+----+      +----+--------+----+
     Head                                                Tail
```

---

## 3. Core Operations on Circular Singly Linked List

In a circular singly linked list, maintaining a pointer to `tail` instead of `head` is an industry-standard optimization:
- `tail.next` is the `head` node ($O(1)$ access to head).
- `tail` is the last node ($O(1)$ access to tail).
- This allows **$O(1)$ insertion at both the beginning and the end**.

```
Tail Pointer Optimization:
tail = Node 3
tail.next = Node 1 (Head)

            tail.next (Head)
                 v
+--------+     +--------+     +--------+
| Node 1 | --> | Node 2 | --> | Node 3 | (Tail)
+--------+     +--------+     +--------+
    ^                             |
    +-----------------------------+
```

### 3.1 Java Implementation of Circular Singly Linked List

```java
public class CircularSinglyLinkedList<T> {

    private static class Node<T> {
        T data;
        Node<T> next;

        Node(T data) {
            this.data = data;
            this.next = null;
        }
    }

    private Node<T> tail; // Points to the last node
    private int size;

    public CircularSinglyLinkedList() {
        this.tail = null;
        this.size = 0;
    }

    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * Inserts at the beginning: O(1) time
     */
    public void addFirst(T data) {
        Node<T> newNode = new Node<>(data);
        if (isEmpty()) {
            newNode.next = newNode;
            tail = newNode;
        } else {
            newNode.next = tail.next;
            tail.next = newNode;
        }
        size++;
    }

    /**
     * Inserts at the end: O(1) time (using tail pointer)
     */
    public void addLast(T data) {
        addFirst(data);
        tail = tail.next; // Advance tail to the newly added node
    }

    /**
     * Removes the first node: O(1) time
     */
    public T removeFirst() {
        if (isEmpty()) {
            throw new java.util.NoSuchElementException("List is empty");
        }
        Node<T> head = tail.next;
        T removedData = head.data;

        if (tail == head) { // Only one element
            tail = null;
        } else {
            tail.next = head.next;
            head.next = null; // Help GC
        }
        size--;
        return removedData;
    }

    /**
     * Removes the last node: O(n) time (need to find predecessor of tail)
     */
    public T removeLast() {
        if (isEmpty()) {
            throw new java.util.NoSuchElementException("List is empty");
        }
        Node<T> head = tail.next;
        T removedData = tail.data;

        if (tail == head) {
            tail = null;
        } else {
            Node<T> curr = head;
            while (curr.next != tail) {
                curr = curr.next;
            }
            curr.next = tail.next;
            tail.next = null; // Help GC
            tail = curr;
        }
        size--;
        return removedData;
    }

    /**
     * Traverses and prints all elements using do-while
     */
    public void printList() {
        if (isEmpty()) {
            System.out.println("List is empty");
            return;
        }
        Node<T> curr = tail.next; // Start from head
        do {
            System.out.print(curr.data + " -> ");
            curr = curr.next;
        } while (curr != tail.next);
        System.out.println("(head)");
    }
}
```

---

## 4. Cycle Detection & Floyd's Cycle-Finding Algorithm

Floyd's Cycle-Finding Algorithm (also known as the **Tortoise and the Hare** algorithm) is the standard method for cycle detection in linked lists.

### 4.1 Concept & Intuition
- Use two pointers: `slow` (advances 1 step at a time) and `fast` (advances 2 steps at a time).
- If there is no cycle, `fast` or `fast.next` reaches `null`.
- If there is a cycle, `fast` enters the cycle first, followed by `slow`. Once both are in the cycle, the relative speed of `fast` with respect to `slow` is:
  $$\Delta v = 2 - 1 = 1 \text{ step/iteration}$$
  Since the relative distance decreases by 1 on every iteration, `fast` is guaranteed to catch up to `slow` without jumping over it.

```
Cycle Detection Trace:
Non-cyclic part length = F
Cycle length = C

[head] ---> ... ---> [start of cycle] ---> ... ---> [meeting point]
|<------- F ------->|<------------- a ------------->|
                    |<--------------------- C -------------------->|
```

### 4.2 Algorithm 1: Detect if a Cycle Exists (LeetCode #141)

```java
public class CycleDetection {

    public static class ListNode {
        int val;
        ListNode next;
        ListNode(int val) {
            this.val = val;
            this.next = null;
        }
    }

    /**
     * Returns true if the linked list contains a cycle.
     * Time Complexity: O(n)
     * Space Complexity: O(1)
     */
    public static boolean hasCycle(ListNode head) {
        if (head == null || head.next == null) {
            return false;
        }

        ListNode slow = head;
        ListNode fast = head;

        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;

            if (slow == fast) {
                return true; // Cycle detected
            }
        }

        return false; // Reached end of list -> No cycle
    }
}
```

### 4.3 Algorithm 2: Find the Starting Node of the Cycle (LeetCode #142)

#### Mathematical Proof:
1. Let $F$ = distance from `head` to cycle entry point.
2. Let $a$ = distance from cycle entry point to meeting point.
3. Let $C$ = circumference (length) of the cycle.
4. When `slow` and `fast` meet:
   $$\text{Distance traveled by } slow = F + a$$
   $$\text{Distance traveled by } fast = F + a + n \cdot C \quad (\text{where } n \ge 1)$$
5. Since `fast` travels at twice the speed of `slow`:
   $$2 \cdot (F + a) = F + a + n \cdot C$$
   $$F + a = n \cdot C$$
   $$F = n \cdot C - a = (n - 1) \cdot C + (C - a)$$
6. $(C - a)$ is the remaining distance from the meeting point to the cycle entry point.
7. Therefore, if one pointer starts at `head` and another pointer starts at the `meeting point`, advancing both at **1 step per iteration**, they will meet exactly at the **cycle entry point** after traversing distance $F$.

```java
public static ListNode detectCycleStart(ListNode head) {
    if (head == null || head.next == null) {
        return null;
    }

    ListNode slow = head;
    ListNode fast = head;
    boolean hasCycle = false;

    // Phase 1: Determine if a cycle exists and locate intersection
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            hasCycle = true;
            break;
        }
    }

    if (!hasCycle) {
        return null;
    }

    // Phase 2: Find the entrance to the cycle
    ListNode ptr1 = head;
    ListNode ptr2 = slow;

    while (ptr1 != ptr2) {
        ptr1 = ptr1.next;
        ptr2 = ptr2.next;
    }

    return ptr1; // Entrance node of the cycle
}
```

### 4.4 Algorithm 3: Find Length of the Cycle

Once `slow` and `fast` meet:
1. Fix one pointer at the meeting point.
2. Advance the other pointer by 1 step until it returns to the meeting point, counting steps.

```java
public static int getCycleLength(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            // Count loop length
            int length = 0;
            ListNode curr = slow;
            do {
                curr = curr.next;
                length++;
            } while (curr != slow);
            return length;
        }
    }
    return 0; // No cycle
}
```

### 4.5 Algorithm 4: Break / Remove the Cycle

```java
public static void removeCycle(ListNode head) {
    if (head == null || head.next == null) return;

    ListNode slow = head;
    ListNode fast = head;
    boolean hasCycle = false;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            hasCycle = true;
            break;
        }
    }

    if (!hasCycle) return;

    // Find cycle entrance
    ListNode ptr1 = head;
    ListNode ptr2 = slow;

    // Special case: cycle starts at head
    if (ptr1 == ptr2) {
        while (ptr2.next != ptr1) {
            ptr2 = ptr2.next;
        }
        ptr2.next = null; // Break loop
        return;
    }

    // General case: find node whose next points to cycle start
    while (ptr1.next != ptr2.next) {
        ptr1 = ptr1.next;
        ptr2 = ptr2.next;
    }

    ptr2.next = null; // Break loop
}
```

---

## 5. Time & Space Complexity Summary

| Operation / Algorithm | Time Complexity | Space Complexity | Notes |
|---|---|---|---|
| **Insertion at Head** (with tail pointer) | $O(1)$ | $O(1)$ | Direct pointer update |
| **Insertion at Tail** (with tail pointer) | $O(1)$ | $O(1)$ | Insert after tail, move tail |
| **Deletion at Head** | $O(1)$ | $O(1)$ | Update tail.next |
| **Deletion at Tail** | $O(n)$ | $O(1)$ | Singly CLL needs $O(n)$ to find predecessor ($O(1)$ in Doubly CLL) |
| **Search / Traversal** | $O(n)$ | $O(1)$ | Terminated when pointer returns to head |
| **Floyd's Cycle Detection** | $O(n)$ | $O(1)$ | Fast pointer takes at most $2(F + C)$ steps |
| **Find Cycle Start** | $O(n)$ | $O(1)$ | Two phases, each bounded by $O(n)$ |
| **Hash Set Approach (Alternative)** | $O(n)$ | $O(n)$ | Store visited nodes in `HashSet<ListNode>` |

---

## 6. Real-World Use Cases

1. **Round-Robin CPU Scheduling**: Operating systems allocate fixed time slices to running processes using a circular linked list so execution naturally cycles back to the first process.
2. **Multiplayer Turn-Based Games**: Players take turns in cyclic order (Player 1 $\to$ Player 2 $\to$ Player 3 $\to$ Player 1).
3. **Circular Buffers / Ring Buffers**: Audio playback, streaming buffers, and producer-consumer message queues where new data wraps around when the buffer capacity is filled.
4. **Media Player Playlists**: Continuous "Repeat All" playback mode.
5. **Fibonacci Heap / Pairing Heap**: Internal child lists in advanced priority queues use circular doubly linked lists for $O(1)$ node concatenation and deletion.

---

## 7. Common Interview Questions & Gotchas

### Q1: Why does the fast pointer move by 2 steps instead of 3 or more?
- **Answer**: With a step difference of 1 ($\Delta v = 2 - 1 = 1$), the relative distance between `fast` and `slow` decreases by exactly 1 in every iteration inside the loop. This guarantees that `fast` will never "jump over" `slow`.
- If `fast` moved by 3 steps ($\Delta v = 2$), it could skip over `slow` when the loop length is even, taking more iterations to meet or requiring modular arithmetic edge handling.

### Q2: What is the maximum number of iterations Floyd's algorithm will run?
- **Answer**:
  - `slow` enters the cycle after $F$ steps.
  - When `slow` enters the cycle, `fast` is already in the cycle at some position.
  - The distance between them inside the cycle is at most $C - 1$.
  - Since the gap closes by 1 step per iteration, they will meet in at most $C$ iterations after `slow` enters.
  - Total steps $\le F + C = n$. Time complexity is strictly linear $O(n)$.

### Q3: What is the classic gotcha when traversing a circular linked list?
- **Gotcha**: Using a standard `while (curr != null)` loop causes an **infinite loop**.
- **Correct approach**: Use a `do-while` loop or track the starting node:
  ```java
  Node curr = head;
  if (curr != null) {
      do {
          // process curr
          curr = curr.next;
      } while (curr != head);
  }
  ```

### Q4: How do you handle edge cases in cycle detection?
- `head == null`: Return `false` / `null`.
- `head.next == null`: Single node pointing to `null` $\to$ Return `false`.
- Single node pointing to itself (`head.next == head`): Cycle of length 1 $\to$ Return `true`.
- Two nodes pointing to each other (`A -> B -> A`): Handled seamlessly by Floyd's algorithm.
