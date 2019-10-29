---
title: Segment Tree
date: 2019-10-29 12:01:45
tags: [Segment Tree, Algorithm, Data Structure]
photos: ["../images/Segment_Tree.png"]
---

A Segment tree is used for storing information about intervals. The general query and insertion operations have a runtime of **O(logn)**. It will also take extra **O(nlogn)** time to build the tree.
<!-- more -->
<br/>

## Introduction

A segment tree is always a full binary tree which every node other than the leaves has two children (each node has exactly zero or two children). Therefore, if there are n nodes in total (n is always an odd number), there will be (n+1)/2 leaves.

![A Full Binary Tree](Full_Binary_Tree.png)
Each of the leaf will represent an unit interval [n:n], which has a length of 1 and assigned weight. A parent node [n:n+1] will always have two children [n:n] and [n+1:n+1], with sum_length to be set as the sum of lengths of its children and sum_weight to be the sum of its children's weights. Hence the sum_length of the root node will represent the length converage of the given intervals and the sum_weight will show the accumulative weight.
<br/>

## Initialization

> **Note**: For this implementation, we fix the boundaries of the tree (assign lower and upper bound while initializing). Thus, if the boundary was assinged to be [0:100], then the insertion of [90:110] will only insert the valid partition of the target (which is [90:100]).

First of all, we decide the boundary of our tree and assign the boundary as the root interval. Then find the middle point of the root interval, assign left child to be [lower_bound:mid] and right child to be [mid:upper_bound]. Recursively calling the init function until start == end in [start:end] which is eactly the leave node of the tree.

```python
def __init__(self, start, end):
        self.start = start
        self.end = end
        self.sum_value = {}
        self.length = {}
        self._init(start, end)


def _init(self, start, end):
    self.sum_value[(start, end)] = 0
    self.length[(start, end)] = 0
    if start < end:
        mid = start + (end - start)//2
        self._init(start, mid)
        self._init(mid+1, end)

```
<br/>

## Insertion

Before the insertion, we validate the boundary of the target interval
```python
def check_bound(self, start, end):
        _start = max(self.start, start)
        _end = min(self.end, end)
        if _start > _end:
            return None, None
        return _start, _end

```
Afterwards, behave the insertion. We may encounter three cases:

1. The target upper bound is less than mid value, hence insert to the left child
![Insert to the left child](Segment_Tree_Left_Insert.png)

2. The target lower bound is greater than mid value, hence insert to the right child
![Insert to the right child](Segment_Tree_Right_Insert.png)

3. The target crosses over two children, thus split it into two intervals and insert to the corresponding child
![Split and insert to both children](Segment_Tree_Split_Insert.png)

Recursively calling insert function and update sum_length and sum_weight of the current root node. Return true if insert target successfully.

```python
def _add(self, start, end, weight, total_start, total_end):
    key = (total_start, total_end)
    if total_start == total_end:
        self.sum_value[key] += weight
        self.length[key] = 1 if self.sum_value[key] != 0 else 0
        return
    mid = self.start + (self.end - self.start)//2
    # if segment is on the left hand side of mid point
    if mid >= end:
        self._add(start, end, weight, total_start, mid)
    # if segment is on the right hand side of mid point
    elif mid < start:
        self._add(start, end, weight, mid+1, total_end)
    # if segment cross over the mid point
    else:
        self._add(start, mid, weight, total_start, mid)
        self._add(mid+1, end, weight, mid+1, total_end)
    self.sum_value[key] = self.sum_value[(total_start, mid)] + self.sum_value[(mid+1, total_end)]   
    self.length[key] = self.length[(total_start, mid)] + self.length[(mid+1, total_end)]


def add(self, start, end, weight = 1):
    _start, _end = self.check_bound(start, end)
    if _start == None:
        return False
    self._add(_start, _end, weight, self.start, self.end)
    return True

```
<br/>

## Query

Same as insertion, query the target interval and return corresponding sum_weight or sum_length, for example:
```python
def _find_sum(self, start, end, total_start, total_end):
    if start == total_start and end == total_end:
        return self.sum_value([start, end])
    mid = total_start + (total_end - total_end)//2
    # if segment is on the left hand side of mid point
    if mid >= end:
        return self._find_sum(start, end, total_start, mid)
    # if segment is on the right hand side of mid point
    if mid < start:
        return self._find_sum(start, end, mid+1, total_end)
    # if segment cross over the mid point
    return self._find_sum(start, mid, total_start, mid) + self._find_sum(mid+1, end, mid+1, total_end)  


def find_sum(self, start, end):
    _start, _end = self.check_bound(start, end)
    if _start == None:
        return 0
    return self._find_sum(_start, _end, self.start, self.end)

```

A full implementation can be referred [here](https://github.com/OhYoooo/Segment-Tree/blob/master/segment_tree.py)
