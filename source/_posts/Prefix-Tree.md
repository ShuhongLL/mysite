---
title: Prefix Tree
date: 2019-07-18 18:30:12
tags: [Prefix Tree, Algorithm, Data Structure]
photos: ["../images/prefixTree.JPG"]
---

> A trie, also called digital tree, radix tree or prefix tree, is a kind of search tree—an ordered tree data structure used to store a dynamic set or associative array where the keys are usually strings. 

Each node of the tree represents a string (prefix) and has multiple child nodes. A prefix tree is always used as search prompt when users browse through the search engine. The runtime of each query operation is independent with the size of the prefix tree; instead, it is decided by the length of the query target. <!-- more -->
</br>

## Sample
Here list an array of some random words:
> sin, sis, con, com, cin, cmd

Then we can construct a prefix tree as following:
![prefix tree sample](prefix_tree.JPG)
During the query operation, there is no need to iterate the whole tree; instead, simply following each node and recursively going through its children will return the target or none if the target does not exist in the tree.
</br>

## Construct A Prefix Tree
There are multiple ways to form a prefix tree. The most popular method is to maintain three arrays in the instance to record end, path and next.

- **Node**
```python
class TrieNode:
    def __init__(self):
        self.nodes = {}
        self.count = 0
        self.isword = False
```

- **CRUD**
```python
class Trie:
    def __init__(self):
        """
        Initialize data structure.
        """
        self.root = TrieNode()

    def insert(self, word: str):
        """
        Inserts a word into the trie.
        :type word: str
        :rtype: void
        """
        curr = self.root
        for char in word:
            if char not in curr.nodes:
                curr.nodes[char] = TrieNode()
            curr.nodes[char].count += 1
            curr = curr.nodes[char]
        curr.isword = True

    def query(self, target: str):
        """
        Returns if the word is in the trie.
        :type target: str
        :rtype: bool
        """
        curr = self.root
        for char in target:
            if char not in curr.nodes:
                return False
            curr = curr.nodes[char]
        return curr.isword

    def startWith(self, prefix: str):
        """
        Returns the number of words in the trie that start with the given prefix.   
        :type prefix: str
        :rtype: int
        """
        curr = self.root
        for char in prefix:
            if char not in curr.nodes:
                return 0
            curr = curr.nodes[char]
        return curr.count

    def delete(self, target: str):
        """
        Returns true if target exist and successfully delete from the trie.
        :type target: str
        :rtype: bool
        """
        curr = self.root
        for char in prefix:
            if char not in curr.nodes:
                return False
            curr = curr.nodes[char]
        if curr.isword:
            curr.isword = False
            return True
        return False

    def listAllMatches(self, prefix: str):
        """
        Returns all words started with prefix
        :param prefix:
        :return: List[str]
        """
        result = []
        def recursiveQuery(node: TrieNode, path: str):
            if not node.nodes:
                result.append(path)
            else:
                for key in node.nodes.keys():
                    recursiveQuery(node.nodes[key], path + key)
        curr = self.root
        for char in prefix:
            if char not in curr.nodes:
                return result
            curr = curr.nodes[char]
        recursiveQuery('')
        return result
```
