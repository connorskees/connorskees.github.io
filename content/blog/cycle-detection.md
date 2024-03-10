+++
title = "Cycle Detection"
+++

Cycle detection is a common interview problem, and one which I think has a number of interesting solutions. There are a few variations of the question: namely, whether you're finding a cycle in a linked-list, an undirected graph, or a directed graph and whether you're detecting the _start_ of a cycle, the nodes involved, the length, or just determining that one exists.

#### Linked Lists

The linked-list case is the simplest of these cases. The naive solution is to use a hash set and add traverse over the list, checking to see whether the node you're currently looking at has already been visited. 

For the sake of explanation, we will walk through a simple implementation of this algorithm, but in general it is suboptimal and uses much more memory than is required by more optimal algorithms.

We define our linked-list recursively like this:

```python
class Node:
    val: int
    next: Node | None
```

And to detect whether a cycle exists:

```python
def cycle_exists(head: Node) -> bool:
    visited = set()

    while head:
        if head in visited:
            return True

        visited.add(head)

        head = head.next

    return False
```

The time and space complexity of this algorithm are both O(n). 

A better solution uses O(1) space. Informally this algorithm is called Floyd's tortoise and hare algorithm, It involves traversing the list with two pointers, one traversing the list twice as fast as the other.

If there is no cycle, the faster pointer will reach the end of the list and terminate. If there _is_ a cycle, then at some point we can guarantee that the fast pointer will point to the same node as the slow pointer.

This is best demonstrated with a diagram:

```
a -> b -> c - > d
     ^----------|
```

This simple example describes a linked list of 4 nodes where d points back to b.

#### Undirected Graphs

#### Directed Graphs

##### Strongly Connected Components

#### Determining Cycle Length


<!--
https://en.wikipedia.org/wiki/Cycle_detection
https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm
https://en.wikipedia.org/wiki/Kosaraju%27s_algorithm
https://math.stackexchange.com/questions/1890620/finding-path-lengths-by-the-power-of-adjacency-matrix-of-an-undirected-graph
https://leetcode.com/discuss/study-guide/4283222/graph-beginners-to-advanced-all-algorithms-python
https://leetcode.com/problems/cycle-length-queries-in-a-tree/
-->
