---
title: LRU Cache Python Implementation
date: 2019-10-22 14:28:12
tags: [LRU, Algorithm, Data Structure]
photos: ["../images/LRU.png"]
---

This implementation is based on **double-linked-list** and **dictionary**. Runtime: O(1)
<!-- more -->

## Init
initialize the linked-list with a sentinel head and tail
![init sentinel node](LRU_init_node.png)
And an empty dictionary:
```python
dict = { }
```
<br/>

## Insert

Suppose we have a object to insert
```
{ keyA: value_a }
```
Firstly we check out if out dictionary already contains a key named `A` (if so remove it from the linked list). Then append the new node to the tail of our linked-list
![init sentinel node](LRU_first_insert.png)
Afterwards, add the new object to our dictionary key by `keyA`
```python
dict = { keyA: valueA }
```

If we have already constructed the follwing linked-list
![init sentinel node](LRU_before_second_insert.png)
Then to insert { keyA: valueA } once again (for example, the same object is used/modified again)
We will firstly remove it from the current linked-list and append the same node at the tail of the list
![init sentinel node](LRU_after_second_insert.png)

After the insertion, if the size of the list hit the maximum capacity, we need to pop the first element (which is not recently used), and assign sentinel head to the next element
<br/>

## Get

Checkout if our dictionary contains the key, if not, which means the element does not exist, return -1.
If the key does exist, get the target node, then remove it from the linked-list and append at the tail
<br/>

## Implementation

```python
class Node:
    def __init__(self, k, v):
        self.key = k
        self.val = v
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.dict = {}
        self.head = Node(0, 0)
        self.tail = Node(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key: int) -> int:
        if key in self.dict:
            node = self.dict[key]
            self._remove(node)
            self._append(node)
            return node.val
        return -1

    def put(self, key: int, value: int) -> None:
        if key in self.dict:
            self._remove(self.dict[key])
        node = Node(key, value)
        self._append(node)
        self.dict[key] = node
        if len(self.dict) > self.capacity:
            n = self.head.next
            self._remove(n)
            del self.dict[n.key]

    def _remove(self, node: Node):
        p = node.prev
        n = node.next
        p.next = n
        n.prev = p

    def _append(self, node: Node):
        p = self.tail.prev
        p.next = node
        node.prev = p
        node.next = self.tail
        self.tail.prev = node

```
