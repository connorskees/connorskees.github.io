+++
title = "Iterative Binary Tree Traversal"
date = "2023-10-06"
+++

There are two primary algorithms for traversing trees: breadth-first search (BFS) and depth-first search (DFS).

In the iterative case, both algorithms rely on creating an array-like structure that stores the nodes to be visited. The main difference between these two is that in breadth-first search you pop from the front (requiring a double-ended queue), while in depth first search you pop from the back (requiring a stack).

DFS is trivial to implement recursively; since it relies on a stack, it can reuse the call-stack. As with all recursive algorithms, in general the iterative form is faster because it avoids the overhead of function calls and has better cache-coherence. BFS, in contrast, is most naturally implemented iteratively.

The recursive implementation of DFS can be more elegant, and often uses fewer lines, but I tend to find it harder to extend to more complex problems.

DFS can be further broken down into pre-order, in-order, and post-order traversal. These traversals change the order in which the root and its left and right children are visited. The specific orderings are:

```
pre-order: root -> left -> right
in-order: left -> root -> right
post-order: left -> right -> root
```

The way that I remember this is that the prefix (i.e. pre, in, post) give the position of `root` in between `left -> right`. In other words, pre-order traversal has the root at the start, in-order in the middle, and post-order at the end. Left always comes before right.

Pre-order traversal is useful when you're searching for a particular element in a BST, you are copying the tree, or if you want to visit the root nodes before visiting the leaf nodes.

Post-order traversal is useful if you're deleting a tree (in e.g. C or C++) or if you want to visit the leaf nodes before visiting root nodes.

In-order traversal is useful if you wish to get a sorted array of nodes, in the case that the tree being traversed over is a valid binary search tree.

For all examples, I'll use a recursive tree structure that is defined as:

```python
class TreeNode:
    def __init__(self, val: int, left: Optional[TreeNode], right: Optional[TreeNode]):
        self.val = val
        self.left = left
        self.right = right
```

A simple implementation of DFS looks like this:

```python
def dfs(root: TreeNode):
    queue = [root]

    while queue:
        node = queue.pop()

        if not node:
            continue

        # ... do something with node ...

        queue.append(node.left)
        queue.append(node.right)
```

This isn't any particular traversal, and you're free to change it up quite a bit. This is the basic code I use when I need to visit every node in a tree using DFS.

As mentioned previously, BFS is the exact same except that you pop from the front:

```python
from collections import deque

def bfs(root: TreeNode):
    queue = deque([root])

    while queue:
        node = queue.popleft()

        if not node:
            continue

        # ... do something with node ...

        queue.append(node.left)
        queue.append(node.right)
```

In python, we use the `deque` data structure to get efficient left-popping. If we were to use a regular array, popping from the left would be an O(n) operation, since all the elements in the array would have to be shifted down by 1. The equivalent data structure in Rust is [`VecDeque`](https://doc.rust-lang.org/std/collections/struct.VecDeque.html).

Breadth-first search is also called level-order search, as you visit all the nodes in a particular level or row before moving on to the next level. 

In general it's quite rare to encounter a problem that benefits significantly more with either BFS or DFS. BFS is very useful for finding the shortest path between two nodes. DFS is useful for finding a particular element within the tree. If there is high width and low depth, generally DFS is preferable. Whereas if there is a low width and high depth, BFS is preferable. For a specific input one algorithm may perform better, but in the generic case of all inputs, unless there is some bias in the input, it tends not to matter whether you choose BFS or DFS.

Returning to the different kinds of DFS traversals:

For a DFS that is explicitly pre-order:

```python
def dfs_pre_order(root: TreeNode):
    queue = [root]

    while queue:
        node = queue.pop()

        if not node:
            continue

        # ... do something with node ...

        queue.append(node.right)
        queue.append(node.left)
```

This is quite similar to the above DFS example, except that we push the right node on the stack before the left node.

For a DFS that is explicitly in-order:

```python
def dfs_in_order(root: TreeNode):
    current = root
    queue = []

    while queue or current:
        if current:
            queue.append(current)
            current = current.left
        else:
            node = queue.pop()

            # ... do something with node ...

            current = node.right
```

The difference here is that we now keep track of a new value, `current`, which we use to traverse the left child nodes prior to the root and right child nodes.

It is also possible to implement in-order traversal iteratively using O(1) extra space (i.e. with no stack) using [Morris traversal](https://stackoverflow.com/questions/5502916/explain-morris-inorder-tree-traversal-without-using-stacks-or-recursion). This algorithm works by temporarily modifying the tree as it is traversed over. It is rare to see this algorithm asked about, so I will omit discussing the particulars here.

Post-order traversal is slightly more complex than pre- and in- order traversal. In order to visit the leaf nodes before the root nodes, we have to traverse the entire tree to get to the leaf nodes. The simplest implementation is one which uses two stacks. The first to perform a basic DFS, and then the second which is built up during the first DFS:

```python
def dfs_post_order(root: TreeNode):
    queue1 = [root]
    queue2 = []

    while queue1:
        node = queue1.pop()

        if not node:
            continue

        queue2.append(node)

        queue1.append(node.left)
        queue1.append(node.right)

    while queue2:
        node = queue2.pop()

        # ... do something with node ...
```

Note that in the second loop, this is the same as `for node in reversed(queue2)`.

It is also possible to implement post-order traversal using a single stack, though I think the implementation is a bit more annoying to remember.

```python
def dfs_post_order_one_stack(root: TreeNode):
    current = root
    queue = []
     
    while current or queue:
        while current:
            queue.append(current)
            queue.append(current)
            current = current.left
         
        node = queue.pop()
 
        if queue and queue[-1] == node:
            current = node.right
        else:
            # ... do something with node ...
            current = None
```
