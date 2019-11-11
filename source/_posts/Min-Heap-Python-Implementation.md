---
title: Min Heap
date: 2019-10-23 14:47:16
tags: [Min Heap, Algorithm, Data Structure]
photos: ["../images/Min_Heap.png"]
---

This is a python implementation of min-heap. It takes runtime of **O(n)** to build/heapify and **O(lgn)** for each of the operation (push or pop).
<!-- more -->
<br/>

## Heap Queue

We use a complete binary tree to represent our heap. A complete binary tree can be stored in an array with list[parent] being the parent node of two children list[parent \* 2] and list[parent \* 2 + 1]
![Min Heap in List](Min_Heap_List.png)

<br/>

## <heapq.py> Implementation

**heapq** is a python library provides an implementation of min heap, also known as the priority queue. This implementation uses zero-based indexing (the first element stored in heapq[0]). A full documentation can be referred [here](https://docs.python.org/2/library/heapq.html)

We will mainly use **heappush** and **heappop** to create our own wrapper.

> **heapq.heappush(heap, item)**
> Push the value item onto the heap, maintaining the heap invariant.

> **heapq.heappop(heap)**
> Pop and return the smallest item from the heap, maintaining the heap invariant. If the heap is empty, IndexError is raised. To access the smallest item without popping it, use heap[0].

```python
import heapq

class MinHeap:
    def __init__(self):
        self._heap = []

    def __len__(self):
        return len(self._heap)

    def push(self, priority, item):
        """
        Push an item with priority into the heap.
        Priority 0 is the highest.
        """
        assert priority >= 0
        heapq.heappush(self._heap, (priority, item))

    def pop():
        """
        Returns the item with lowest priority
        """
        return heapq.heappop(self._heap)[1] # (priority, item)[1] == item

```

<br/>

## Raw Implementation

We maintain the heap in a index-one-based list (always keep heap[0] to be None). For all elements in the heap, heap[parent] will ideally have two children on heap[parent \* 2] and heap[parent \* 2 + 1]. Besides, in order to maintain the heap to be a complete binary tree, we should always insert to the end of the list and do some procedures to rearrange the new element to a proper place in our heap.

For the insertion, we will always append to the list. Afterwards, bubble up the new element. Start from the last element (which is the newly inserted one), compare its value with its parent (index divided by two), if less than its parent, swap parent and child until its value is no longer less than its parent or hit the root of the heap.
![Insert in Min Heap](Min_Heap_Insert.png)

When calling pop, we pop heap[1], which is the minimum value out of the heap, replace heap[1] with the last element of the heap to maintain a complete binary tree. Afterwards, sink down the new root element to a proper palce in the heap. Precisely, compare the value of the root (previously the last element) and two of its child, if any of them is less than the root value, swap root and that child. Keeping doing this until none of its two child is less than itself or reach the leaf of the tree.
![Pop in Min Heap](Min_Heap_Pop.png)

Below is a complete implementation

```python
class MinHeap:
    def __init__(self):
        """
        Init with index-one-based list
        """
        self._heap = [None]

    def __len__(self):
        return len(self._heap) - 1

    def _swap(self, t1, t2):
        self._heap[t1], self._heap[t2] = self._heap[t2], self._heap[t1]

    def _up_heap(self, index):
        """
        Bubble up the inserted element to the proper place
        """
        while index > 1:
            parent = index // 2
            if self._heap[parent] > self._heap[index]:
                self._swap(parent, index)
            index = parent

    def _down_heap(self, index):
        """
        Sink down the root element
        """
        length = len(self._heap)
        while index * 2 < length:
            child = index * 2
            # compare two children, assign target child to be the smaller one
            if child + 1 < length and self._heap[child+1] < self._heap[child]:      
                child += 1
            if self._heap[child] < self._heap[index]:
                self._swap(child, index)
            index = child

    def empty():
        return len(self._heap) == 1

    def push(self, value):
        """
        Push a new element at the tail of the heap
        And bubble up
        """
        if not value:
            raise TypeError('"value" cannot be of NoneType')
        self._heap.append(value)
        self._up_heap(len(self._heap) - 1)
    
    def pop(self):
        """
        Pop the root element, which is the minimum value of the heap
        Assign the last element from the list to be the new root
        Then sink down
        """
        if leb(self._heap) <= 1:
            return None
        result = self._heap[1]
        last = self._heap.pop(-1)
        if len(self._heap) > 1:
            self._heap[1] = last
            self._down_heap(1)
        return result
    
    def peak(self):
        if len(self._heap) > 1:
            return self._heap[1]
        return None

```
