---
title: "Union-Find"
date: 2021-05-31T15:23:41-05:00
draft: false
---

Union-Find answers the question "Are two nodes in the same group?"
so if we have something like `A->B->C->D` it's a way to ask
"Is D connected to A".

Typically, a Union-Find instance itself does not store the objects
in question themselves. Usually, each object is associated with a
number from 0 to N, and that data is used by Union-Find.

The API for Union-Find usually looks something like:

* `union(int p, int q) -> int`
  * Links node p to node q.
* `find(int p) -> int`
  * Finds the root node for node p.
* `connected(int p, int q) -> bool`
  * Tests if nodes p and q are connected (i.e. have the same root
    node)
* `count() -> int`
  * Returns the number of unconnected components represented by the
    Union-Find instance.

It does this, by storing a few arrays internally. One representing
the "parents" of each group (initially filled completely with unique
values, because at the start each node is its own parent), and a
"rank" array that tracks the size of each group (this is used for
making decisions about weather to link p to q or q to p).

Let's say we create an instance of size 10, those arrays may
initially look like:

    parents: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    rank:    [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

Each index in parents represents a node, and it's value represents
it's parent node's index. So if we call `union(2, 5)`, we would end
up with something like this

    parents: [0, 1, 2, 3, 4, 2, 6, 7, 8, 9]
    rank:    [0, 0, 1, 0, 0, 0, 0, 0, 0, 0]

Then we could call `union(2, 8)`:

    parents: [0, 1, 2, 3, 4, 2, 6, 7, 2, 9]
    rank:    [0, 0, 2, 0, 0, 0, 0, 0, 0, 0]

`union(4, 6)`

    parents: [0, 1, 2, 3, 4, 2, 4, 7, 2, 9]
    rank:    [0, 0, 2, 0, 1, 0, 0, 0, 0, 0]

Now, see what happens when we call `union(4, 2)`

    parents: [0, 1, 2, 3, 2, 2, 4, 7, 2, 9]
    rank:    [0, 0, 3, 0, 1, 0, 0, 0, 0, 0]

You'll notice we made 2 the parent. This is because 2 has a higher
rank than 4. We do this to prevent large trees from being added as
children to small trees, which could increase our traversal time.

There's one more optimization we could apply here, and that is path
compression. This essentially flattens out the tree. None of the ops
we've don so far would've triggered it, but let's see it in action
with `union(9, 5)`

    parents: [0, 1, 2, 3, 2, 2, 4, 7, 2, 2]
    rank:    [0, 0, 4, 0, 1, 0, 0, 0, 0, 0]

The important thing to notice, is that index 9's value became 2,
not 5. That is because 2 is 5's parent, so we've flattened out
the tree.

Let's look at each method in detail.

### Union

Takes two parameters, p and q.

1. First, find the root nodes of p and q.
2. Determine which node has the highest rank

This is done by looking up that root node in the ranks array. If p
has a higher rank, then p becomes the parent, otherwise q becomes
the parent. If both are equal, then we pick p arbitrarily. In the
case that both are equal, we ALSO increase the rank.

3. Decrease the connected component count.

We've just connected a component, meaning we now have fewer
independent components.

```python
def union(self, p: int, q: int) -> None:
    # Find p and q's root node.
    p_id = self.find(p)
    q_id = self.find(q)

    # If they are the same, then p and q are already unioned
    # together.
    if p_id == q_id:
      return

    # Rank will contain the size of a tree at a particular root,
    # we want to join the smaller tree to the larger tree, as this
    # means we have to do less traversals on find operations.
    if self._rank[p_id] > self._rank[q_id]:
      self._parents[p_id] = q_id
    elif self._rank[p_id] < self._rank[q_id]:
      self._parents[q_id] = p_id
    else:
      self._parents[q_id] = p_id
      self._rank[p_id] += 1

    # Since we've done a union, we now have one fewer indpendent
    # component.
    self._count -= 1
```


### Find

This is done by traversing the array. We look up the parent value
of `p`, by using `p` as an index. So if `p` is 2, we'll look at
array index 2. Let's say the value at array index 2 is 3. This means
that index 2 is not a root, so we set `p = 3`, and go to index 3.
The value in index 3, _is_ 3, so we've found a root and we can
return.

```python
def find(self, p: int) -> int:
  # Traverse up the tree until we find a root. A root is denoted by
  # having the same value as it's index.
  while p != self._parents[p]:
    p = self._parents[p]

  return p
```

Path compression is also done at this step. Let's say we had to
follow a chain of indexes to finally find a root value. While we
traverse that array, we would set each value to be equal to its
parent, so if we had `A->B->C->D`, and we performed `find(D)`,
the root value is `A`. So while we're traversing the array to find
`A`, we would set the parents of `B` and `C` to `A`. Leaving us
with:

    B -> A <- C
         ^
         |
         D

```python
# Find with Path Compression!
def find(self, p: int) -> int:
  # Same basic logic as above, but this time we flatten out the tree
  # as we traverse it!
  while p != self._parents[p]:
    tmp = self._parents[p]
    self._parents[p] = self._parents[self._parents[p]]
    p = tmp

  return p
```
   

### Connected

This is easy, we just call find on p and q and check if they're
root nodes are the same.

```python
def connected(self, p: int, q: int) -> bool:
  # Do both nodes have the same root?
  return self.find(p) == self.find(q)
```


## Applications

* Find connected components in a graph.
* Used as part of **Kruskal's Algorithm** for detecting cycles in
    a graph

## Implementation

Here's a basic Python3 implementation of Union-Find

```python
class UnionFind:
    def __init__(self, size):
        self._count = size
        self._parents = {}
        self._rank = {}
        for i in range(size):
            self._parents[i] = i
            self._rank[i] = 0

    @property
    def count(self):
        return self._count

    def union(self, p: int, q: int) -> None:
        p_id = self.find(p)
        q_id = self.find(q)
        if p_id == q_id:
            return

        if self._rank[p_id] < self._rank[q_id]:
            self._parents[p_id] = self._parents[q_id]
        elif self._rank[p_id] > self._rank[q_id]:
            self._parents[q_id] = self._parents[p_id]
        else:
            self._parents[q_id] = self._parents[p_id]
            self._rank[p_id] += 1

        self._count -= 1

    def find(self, p: int) -> int:
        # Path Compression
        # As we traverse the tree, we flatten it out by setting
        # each node to it's parent's parent.
        while p != self._parents[p]:
            tmp = self._parents[p]
            self._parents[p] = self._parents[self._parents[p]]
            p = tmp

        return p

    def connected(self, p: int, q: int) -> bool:
        return self.find(p) == self.find(q)
```
