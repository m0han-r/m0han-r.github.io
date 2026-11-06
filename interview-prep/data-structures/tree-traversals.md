← [Back to Index](../README.md)

# Binary Tree Traversals — Complete Interview Guide

## 1. Concept Overview & Traversal Taxonomy

Tree traversal is the process of visiting every node in a tree data structure exactly once. Binary tree traversals are categorized into two primary paradigms:

```
                            Tree Traversals
                                   |
         +-------------------------+-------------------------+
         |                                                   |
   Depth-First Search (DFS)                        Breadth-First Search (BFS)
         |                                                   |
   +-----+-----+-----+                             +---------+---------+
   |           |     |                             |                   |
Inorder    Preorder Postorder                 Level-Order           Zigzag
(L-Root-R) (Root-L-R)(L-R-Root)               (Queue-based)         (Spiral)
```

### Reference Binary Tree for Tracing
```
              1
            /   \
           2     3
          / \   / \
         4   5 6   7
```
- **Inorder** (Left $\to$ Root $\to$ Right): `[4, 2, 5, 1, 6, 3, 7]`
- **Preorder** (Root $\to$ Left $\to$ Right): `[1, 2, 4, 5, 3, 6, 7]`
- **Postorder** (Left $\to$ Right $\to$ Root): `[4, 5, 2, 6, 7, 3, 1]`
- **Level-Order** (Breadth-First by level): `[[1], [2, 3], [4, 5, 6, 7]]`

### Common `TreeNode` Class Definition
```java
public class TreeNode {
    public int val;
    public TreeNode left;
    public TreeNode right;

    public TreeNode(int val) {
        this.val = val;
    }
}
```

---

## 2. Inorder Traversal (Left $\to$ Root $\to$ Right)

In a Binary Search Tree (BST), an inorder traversal visits nodes in **strictly ascending sorted order**.

### 2.1 Recursive Inorder
```java
import java.util.ArrayList;
import java.util.List;

public class InorderTraversal {

    public List<Integer> inorderRecursive(TreeNode root) {
        List<Integer> result = new ArrayList<>();
        helper(root, result);
        return result;
    }

    private void helper(TreeNode node, List<Integer> result) {
        if (node == null) return;
        helper(node.left, result);    // Left
        result.add(node.val);         // Root
        helper(node.right, result);   // Right
    }
}
```

### 2.2 Iterative Inorder (Using Stack)
**Algorithm**:
1. Push the current node and all its left descendants onto a stack until reaching `null`.
2. Pop a node from the stack, record its value (visit root).
3. Set the current node to the popped node's right child and repeat.

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

public List<Integer> inorderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;

    while (curr != null || !stack.isEmpty()) {
        // Step 1: Reach the leftmost node of the current subtree
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }

        // Step 2: Current must be null at this point, pop from stack
        curr = stack.pop();
        result.add(curr.val); // Visit node

        // Step 3: We have visited the node and its left subtree; now visit right
        curr = curr.right;
    }

    return result;
}
```

---

## 3. Preorder Traversal (Root $\to$ Left $\to$ Right)

**Use cases**: Tree cloning, serialization/deserialization, prefix notation generation.

### 3.1 Recursive Preorder
```java
public List<Integer> preorderRecursive(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    helper(root, result);
    return result;
}

private void helper(TreeNode node, List<Integer> result) {
    if (node == null) return;
    result.add(node.val);         // Root
    helper(node.left, result);    // Left
    helper(node.right, result);   // Right
}
```

### 3.2 Iterative Preorder (Using Stack)
**Algorithm**:
1. Push root to stack.
2. While stack is not empty: pop node, visit it.
3. Push **right** child first, then **left** child (because stack is LIFO, left child will be processed first).

```java
public List<Integer> preorderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode curr = stack.pop();
        result.add(curr.val); // Visit Root

        // Push right first so left is popped first
        if (curr.right != null) {
            stack.push(curr.right);
        }
        if (curr.left != null) {
            stack.push(curr.left);
        }
    }

    return result;
}
```

---

## 4. Postorder Traversal (Left $\to$ Right $\to$ Root)

**Use cases**: Bottom-up computation (calculating subtree heights, directory tree sizes), deleting a tree (freeing children before parent), evaluating postfix expressions.

### 4.1 Recursive Postorder
```java
public List<Integer> postorderRecursive(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    helper(root, result);
    return result;
}

private void helper(TreeNode node, List<Integer> result) {
    if (node == null) return;
    helper(node.left, result);    // Left
    helper(node.right, result);   // Right
    result.add(node.val);         // Root
}
```

### 4.2 Iterative Postorder (Single Stack with `prev` Pointer)
A node can be visited only if:
- It has no right child, OR
- Its right child was the immediately preceding visited node (`prev`).

```java
public List<Integer> postorderIterativeOneStack(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;
    TreeNode prev = null;

    while (curr != null || !stack.isEmpty()) {
        // Go all the way to the leftmost node
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }

        curr = stack.peek();

        // If right child exists and hasn't been visited yet, explore right subtree
        if (curr.right != null && curr.right != prev) {
            curr = curr.right;
        } else {
            // Both left and right subtrees are visited; now visit root
            result.add(curr.val);
            prev = stack.pop();
            curr = null; // Reset curr so we don't traverse down left again
        }
    }

    return result;
}
```

### 4.3 Iterative Postorder (Two Stacks / Modified Preorder Trick)
Since Preorder is `Root -> Left -> Right`:
- Reverse child push order to get: `Root -> Right -> Left`.
- Reversing that entire sequence produces: `Left -> Right -> Root` (Postorder).

```java
import java.util.LinkedList;

public List<Integer> postorderTwoStacks(TreeNode root) {
    LinkedList<Integer> result = new LinkedList<>();
    if (root == null) return result;

    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode curr = stack.pop();
        result.addFirst(curr.val); // Reverse order insertion

        if (curr.left != null) {
            stack.push(curr.left);
        }
        if (curr.right != null) {
            stack.push(curr.right);
        }
    }

    return result;
}
```

---

## 5. Level-Order Traversal (BFS)

BFS explores nodes horizontally, level by level, using a **FIFO Queue**.

```
Level 0:           [ 1 ]
                 /       \
Level 1:     [ 2 ]       [ 3 ]
             /   \       /   \
Level 2:   [ 4 ] [ 5 ] [ 6 ] [ 7 ]
```

### 5.1 Level-by-Level (LeetCode #102)
```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.List;
import java.util.Queue;

public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> currentLevel = new ArrayList<>(levelSize);

        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            currentLevel.add(node.val);

            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }

        result.add(currentLevel);
    }

    return result;
}
```

### 5.2 Zigzag / Spiral Level Order (LeetCode #103)
Alternate between adding values to the front or back of each level list:

```java
import java.util.LinkedList;

public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);
    boolean leftToRight = true;

    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        LinkedList<Integer> currentLevel = new LinkedList<>();

        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();

            if (leftToRight) {
                currentLevel.addLast(node.val);
            } else {
                currentLevel.addFirst(node.val);
            }

            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }

        result.add(currentLevel);
        leftToRight = !leftToRight;
    }

    return result;
}
```

---

## 6. Deep Dive: Morris Traversal ($O(1)$ Auxiliary Space)

Both recursion and stack-based iterative traversals consume **$O(h)$ auxiliary space** on the call stack / explicit stack (where $h = \log n$ for balanced trees, $h = n$ for skewed trees).

**Morris Traversal** achieves **$O(n)$ time** and strictly **$O(1)$ space** without modifying the tree permanently.

### Core Concept: Threaded Binary Tree
Before descending into the left subtree, locate the **inorder predecessor** (the rightmost node in the left subtree):
1. **Thread Creation**: If `predecessor.right == null`, make it point back to `curr` (`predecessor.right = curr`), then step `curr = curr.left`.
2. **Thread Removal**: If `predecessor.right == curr`, the left subtree has already been fully visited! Break the thread (`predecessor.right = null`), visit `curr`, and advance `curr = curr.right`.

```
Morris Threading:
          curr (1)
          /      \
        (2)      (3)
        / \
      (4) (5) ---> [thread to curr (1)]
           ^
           |
       predecessor of (1)
```

### 6.1 Morris Inorder Traversal
```java
public List<Integer> morrisInorder(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    TreeNode curr = root;

    while (curr != null) {
        if (curr.left == null) {
            // No left subtree: visit curr and go right
            result.add(curr.val);
            curr = curr.right;
        } else {
            // Find inorder predecessor: rightmost node in left subtree
            TreeNode pred = curr.left;
            while (pred.right != null && pred.right != curr) {
                pred = pred.right;
            }

            if (pred.right == null) {
                // Step 1: Create thread back to curr and go left
                pred.right = curr;
                curr = curr.left;
            } else {
                // Step 2: Left subtree already visited. Restore tree.
                pred.right = null;
                result.add(curr.val); // Visit Root
                curr = curr.right;
            }
        }
    }

    return result;
}
```

### 6.2 Morris Preorder Traversal
The only difference from inorder is **when** the node is visited: visit before establishing the thread and advancing left.

```java
public List<Integer> morrisPreorder(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    TreeNode curr = root;

    while (curr != null) {
        if (curr.left == null) {
            result.add(curr.val);
            curr = curr.right;
        } else {
            TreeNode pred = curr.left;
            while (pred.right != null && pred.right != curr) {
                pred = pred.right;
            }

            if (pred.right == null) {
                result.add(curr.val); // Visit Root BEFORE descending left
                pred.right = curr;
                curr = curr.left;
            } else {
                pred.right = null; // Revert thread
                curr = curr.right;
            }
        }
    }

    return result;
}
```

---

## 7. Complexity Summary Table

| Traversal Strategy | Time Complexity | Auxiliary Space (Balanced Tree) | Auxiliary Space (Skewed Tree) | Memory Allocation |
|---|---|---|---|---|
| **Recursive Inorder / Pre / Post** | $O(n)$ | $O(\log n)$ | $O(n)$ | Call stack frames |
| **Iterative Inorder / Pre (Stack)** | $O(n)$ | $O(\log n)$ | $O(n)$ | Heap (`ArrayDeque`) |
| **Iterative Postorder (1 Stack)** | $O(n)$ | $O(\log n)$ | $O(n)$ | Heap (`ArrayDeque`) |
| **Iterative Postorder (2 Stacks)** | $O(n)$ | $O(n)$ | $O(n)$ | 2 Stacks |
| **Level-Order BFS (Queue)** | $O(n)$ | $O(w) = O(n)$ | $O(1)$ | Queue (max level width $w \approx n/2$) |
| **Morris Traversal (Inorder / Pre)**| $O(n)$ | **$O(1)$** | **$O(1)$** | None (pointers modified & restored) |

*Note on Morris Time*: Each tree edge is traversed at most 3 times (once locating predecessor, once descending, once resetting pointer), maintaining $O(n)$ total time.

---

## 8. Common Interview Questions & Gotchas

### Q1: Why can't we uniquely reconstruct a binary tree using ONLY Preorder and Postorder traversals?
- **Answer**: Preorder and Postorder convey information about parent-child relationships, but they **cannot distinguish between a single left child versus a single right child**.
- Example:
  ```
  Tree A:    1          Tree B:    1
            /                       \
           2                         2
  ```
  For both trees:
  - Preorder: `[1, 2]`
  - Postorder: `[2, 1]`
- You need **Inorder** to split the tree into unambiguous left and right subtrees (`Inorder + Preorder` or `Inorder + Postorder`).
- *Exception*: A Full Binary Tree (where every node has 0 or 2 children) CAN be uniquely reconstructed from Preorder and Postorder.

### Q2: What causes a `StackOverflowError` in recursive traversals and how do you prevent it?
- **Answer**: If the binary tree is heavily skewed (e.g. linked list shape of depth $10^5$), recursive traversal builds $10^5$ stack frames, exceeding the default JVM stack size (`-Xss`, typically 1MB).
- **Prevention**: Use iterative traversal with an explicit heap-based `Deque` (bounded by heap size, not thread stack), or use **Morris Traversal** for $O(1)$ space.

### Q3: When should you avoid Morris Traversal in production?
- **Answer**:
  1. **Multi-threaded environments**: Morris temporarily alters tree pointers (`pred.right = curr`). If another thread reads the tree concurrently, it encounters corrupt cycle references.
  2. **Read-only / Immutable trees**: Read-only memory or frozen objects cannot have pointers rewritten.
