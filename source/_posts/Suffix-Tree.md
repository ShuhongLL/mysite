---
title: Suffix Tree
date: 2019-07-22 20:07:12
tags: [Suffix Tree]
photos: ["../images/SuffixTree_Cover.JPG"]
---
Suffix tree is a data structure aimming at solving **string** related problems in **linear time**
<!-- more -->

## String Match Algorithm

A common string match problem always contains:
- **Text** as an array of length n: T[n]
- **Pattern** as an array of length m which m <=n: P[m]
- **Valid Shift** which is the offset of the first character of the pattern showing up in the target text
![String Match Problem Sample](PrefixTree_StringProblem.JPG)

We usually use two methods to solve string match problem:
- Naive String Matching Algorithm
- Knuth-Morris-Pratt Algorithm（KMP Algorithm)

The algorithm always consists with two steps: **Preprocessing** and **Matching**, and the total runtime will be accordingly the sum of two procedures.
> Naive String Matching: O( (n-m)m )
> KMP: O( m + n )

Such algorithms are all doing preprocessings on the pattern to boost the searching procedure and the best runtime perfomance will be O(m) on preprocessing where m represents the length of the pattern. On contrary, is there a preprocessing which can be applied on **text** to speed up the whole process? This is the key reason why I am moving to **suffix tree**, which is a data structure doing preprocessing on text.
</br>

## Suffix Tree

My previous post used to introduce a prefix tree, for example:
![prefix tree sample](prefix_tree.JPG)
Individual nodes branch out from the same prefix. As you can see, there are some nodes which only have one child. Let's try to compress them together:
![prefix tree sample](Compressed_PrefixTree.JPG)
After the compression, we get a **Compressed Prefix Tree**. A compressed prefix tree, also called **Compressed Tire** is the fundamental of suffix tree. Besides, the key values stored in each nodes of a suffix tree is all the possible suffix.

For example, for a single word (**Text**) `banana\0`, we have the following set of suffix:
```
banana\0
anana\0
nana\0
ana\0
na\0
a\0
\0
```
Construct a prefix tree using the above key words:
![Banana Prefix Tree](Banana_PrefixTree.JPG)
Afterwards, compress it:
![Banana Suffix Tree](Banana_SuffixTree.JPG)
Here, by lising all the suffix and making a compressed prefix tree, we obtain a suffix tree.

However, there is a faster way to construct a suffix tree which can be done in the linear time (By Esko Ukkonen in 1997). Let's start from a simple example before exploring more complicated cases.
</br>

## "abc"
Unlike prefix tree, the edge in suffix tree will no longer represent a single character(key); instead, it will represent a list of integers [from, to] which interprets the indexes in the original text. Therefore, each edge can represent abitrary length of characters and consume constant memory.

Considering a simple string ` abc ` which has no duplicated character. Start from the first character ` a `, construct an edge from root node to its leaf:
![Sample "abc" -- a](PrefixTree_a.JPG)
Where [0, #] represents start from offset 0 and end up at offset #. Currently # equals to 1, which represents character ` a `.

After finishing the procudure of the first character, let's move forward to the second one ` b `:
![Sample "abc" -- ab](PrefixTree_ab.JPG)
Then do the same thing on the last character ` c `:
![Sample "abc" -- abc](PrefixTree_abc.JPG)
We maintain a pointer pointing to the current end character of the text. Each time when we add a new edge for a new character, we will update the # point. Thus, the runtime for each procedure is O(1) and hence the overall runtime to construct this "abc" suffix tree will be O(n).
</br>

## "abcabxabcd"
"abc" is a really simple example since there is no duplicated characters in the text. Now let's consider a more complicated text ` abcabxabcd `. Before we get started to construct, there are two more concepts that need to be introduced

- **active point**: a structure indicates `active_node`, `active_value` and `offset`
- **remainder**: remaining characters to be inserted

```
active_point = (root, '', 0)
remainder = 0
```
</br>

### The first character => `a`

We append ` a ` directly to the root node
1. Before we append the node, set `remainder = remainder + 1` (0 + 1), states that we are going to insert a new node
2. Append 'a' to the root node
3. After the insertion, a new node is appended to the tree, hence `remainder = remainder - 1` (1 - 1)
4. After the whole procudure:
```
active_point = (root, '', 0)
remainder = 0
```

![Sample "abcabx" -- abc](SuffixTree-Sample-a.JPG)

### The second character => `b`

1. Before we append the node, set `remainder = remainder + 1` (0 + 1), states that we are going to insert a new node
2. Expend every leave node; hence change 'a' to 'ab'
3. Query the current node, which is the root node, since there is no prefix starts with 'b', we need to create a new node and insert 'b'. (The prefix for the previous prefix will be [a, ab] now)
3. After the insertion, a new node is appended to the tree, hence `remainder = remainder - 1` (1 - 1)
4. After the whole procudure:
```
active_point = (root, '', 0)
remainder = 0
```

![Sample "abcabx" -- abc](SuffixTree-Sample-ab.JPG)

### The third character => `c`

Same as the procedure of inserting 'b'
![Sample "abcabx" -- abc](SuffixTree-Sample-abc.JPG)

### The fourth character => `a` again

This time we encounter the same character we have already inserted
1. Before we append the node, set `remainder = remainder + 1` (0 + 1)
2. Expend every leave node; hence change 'abc' -> 'abca', 'bc' -> 'bca', 'c' -> 'ca'
3. Query the current node, which is the root node, find out the there already exist a prefix which starts from 'a'; hence we modify the `active_point`:
```
active_point = (root, 'a', 1)
remainder = 1
```
> **active_node**: the parent node is still root node hence remain unchanged
> **active_value**: the current value will be set to 'a'
> **offset**: the offset of the current value, which is 1 (states the first character in the prefix)
> **remainder** after the insertion, no new node is inserted in the tree, hence remainder keep the same (remainder = 1)


### The fifth character => `b` again

Still duplicated with previous nodes
1. `remainder = remainder + 1` (1 + 1)
2. Expend every leave node; hence change 'abca' -> 'abcab', 'bca' -> 'bcab', 'ca' -> 'cab', `active_point[1] = 'a' -> 'ab'`
3. Start from **active_node**, query the **active_value** in its prefix and the prefix of its sub-nodes. Modify **active_point** to the following:
```
active_point = (root, 'ab', 2)
remainder = 2
```
> **active_node**: the active node is still root node hence remain unchanged.
> **active_value**: the active value has been expended to 'ab'.
> **offset**: 'ab' already exists in node 'ab' (which is the sub-node of root node), the offset of matched index is 2.
> **remainder**: after the insertion, no new node is inserted in the tree, hence remainder keep the same

By this step, we still have the same amount of nodes inserted into the tree (3 nodes)
![Sample "abcabx" -- abcab](SuffixTree-Sample-abcab.JPG)

### The sixth character => `x`

1. `remainder = remainder + 1` (2 + 1).
2. Expend every leave node; hence change 'abcab' -> 'abcabx', 'bcab' -> 'bcabx', 'cab' -> 'cabx', `active_point[1] = 'ab' -> 'abx'`.
3. Start from **active_node**, query the **active_value** in its prefix and the prefix of its sub-nodes. Since there isn't a prefix which perfectly matches the pattern 'abx'; hence this is the point where to **split**:

### Split

When we find out the exact place where the prefix starts to mismatch the pattern, then we need to insert new nodes to our tree to append this new suffix character. For example, prefix 'abc' and pattern 'abx' share the same two characters 'ab' at the beginning and mismatch at the third character (offset = 3), then we can simply split at the third index to make a tree like 'ab' -> ['c', 'x']. We will do the same thing here:
1. We already know the offset (which is active_point[2]).
2. Split node **'abcabx' to 'ab' -> ['cabx', 'x']**.
3. Modify **active_point** and **remainder**:
```
active_point = (root, 'bx', 1)
remainder = 2
```
> **active_node**: still root node
> **active_value**: 'abx' has been inserted, then we need to insert 'bx', then 'x' and so on so far
> **offset**: active_point[1] changes from 'abx' to 'bx', then the offset will also shift from 2 to 1
> **remainder**: a new node is inserted, hence `remainder = remainder - 1` (3 - 1)
>> **Why do we change offset as well?**
>> This is actually a trick to reduce runtime. For each step of insertion, we will make an sub suffix tree by characters we already have (inserted). For example, if we have a string 'abc' and we have inserted the last character 'c' in the suffix tree, then all possible suffix of string (text) 'abc', which is 'abc', 'bc' and 'c' must exist in the suffix tree, either implicitly (have not splitted yet, like the fourth insertion above) or explicitly (has its own node). Hence, by inserting 'ab +x' and splitting the node, we know that 'b +x' must exist in the tree; therefore, we reduce the offset by 1 to state that the offset of the next split will start from the first character of the node 'b...'.

![Sample "abcabx" -- abcabx](SuffixTree-Sample-abcabx.JPG)
4. The remainder is still greater than 0; hence we will keep inserting 'bx'. Do the same thing here, start from the **parent_node**, query the sub prefix 'bx'. Found 'bx' in the prefix 'bcabx'; hence split the node **'bcabx' to 'b' -> ['cabx', 'x']**
5. Modify **active_point** and **remainder**:
```
active_point = (root, 'x', 0)
remainder = 1
```
> **active_node**: still root node
> **active_value**: 'bx' has been inserted, then we need to insert 'x'
> **offset**: active_point[1] changes from 'bx' to 'x', then the offset will also shift from 1 to 0
> **remainder**: a new node is inserted, hence `remainder = remainder - 1` (2 - 1)

![Sample "abcabx" -- abcabx - 2](SuffixTree-Sample-abcabx-2.JPG)

6. Since remainder is still greater than 0, we have to do another insertion to reduce it to zero. However, the character 'x' hasn't been inserted before, then we append 'x' directly to the root node.
7. Modify **active_point** and **remainder**:
```
active_point = (root, '', 0)
remainder = 0
```
> **active_node**: still root node
> **active_value**: the last character 'x' has been inserted, reduce to empty
> **remainder**: a new node is inserted, hence `remainder = remainder - 1` (1- 1)

![Sample "abcabx" -- abcabx - 3](SuffixTree-Sample-abcabx-3.JPG)

**Is there a relationship between two splitted node `ab` and `b`?**
When we separated 'ab' and 'b', we modified the previous 'abcabx' node first, then modified the second node 'bcabx'. If we encounter another string starting with 'abk', then after 'abk' is inserted, we definately need to insert 'bk' as well. Repeating the same query will slow down the whole procedure; thus, we can add a link from 'ab' to 'b' to indicate that if an update is made in node 'ab', then another insertion will be done on the link target 'b' afterwards.
![Sample "abcabx" -- abcabx - 4](SuffixTree-Sample-abcabx-4.JPG)

### The seventh character => `a`

Same as the fourth insertion 'a'
```
active_point = (root, 'a', 1)
remainder = 1
```

### The eighth character = `b`

Same as the fifth insertion 'b'
```
active_point = (root, 'ab', 2)
remainder = 2
```

### The ninth character => `c`

When we try to insert 'c'
1. `remainder= remainder + 1` (3)
2. Expand every leave node
![Sample "abcabxabc" -- abcabxabc](SuffixTree-Sample-abcabxabc.JPG)
3. Query 'abc', find 'abc' under the parent node 'ab'
> **Extra Procedure**:
> **active_point**: The node 'cabxabc' constains the last character of the target prefix ('abc'), whose parent node is 'ab'; thus, set `active_point[0] = 'ab'`.
> **active_node**: set `active_point[1] = 'abc'`.
> **offset**: the offset of the character 'c' in its node string('cabxabc') which is 1. Set `active_point[2] = 1`
> **remainder**: After the modification, no new node is inserted, hence **remainder** keeps unchanged.

```
active_point = ('ab', 'abc', 1)
remainder = 3
```

### The last character => `d`

We will use the link built-up in previous steps to simplify the updates.
1. `remainder = remainder + 1` (3 + 1)
2. Expand every leave nodes
3. Query the current parent node ('ab'), find if there exist any prefix contains the target 'abcd'? Since there isn't any prefix match the pattern, then this will be a split point.

### Split

> **active_point[0] = 'ab'**: start from the curent parent node ('ab'), looking into its sub nodes.
> **active_point[1] = 'abcd'**: 'd' is inserted into the active value .
> **active_point[2] = 1**: parent node 'ab' already contains 'ab', and its sub-node 'cabxabcd' starts from the third character 'c' at offset 0. Then the second character of 'cabxabcd', which is 'a' mismatches the following target 'd'; hence this is where to be splitted, at offset 1 (of 'cabxabcd').

1. Split **'cabxabcd' to c -> [abxabcd, d]**
2. Since a new node has been inserted, then `remainder = remainder - 1` (3). Remainder is still greater than 0, we still cannot stop here.
```
active_point = ('ab', 'abcd', 1)
remainder = 3
```

![Sample "abcabxabc" -- 10 - 1](SuffixTree-Sample-abcabxabc-10-1.JPG)
3. active_point[0] ('ab') has a link pointing to 'b'. We change the active_point according to the link target.
> **active_point[0] = 'b'**
> **active_point[1] = 'bcd'**
> **active_point[2] remains unchange**

4. start from active_point[0], query its sub-nodes. 'b' is the parent node and its sub-node 'cabxabcd' contains 'c' at offset 0 and mismatch at offset 1; hence we split its sub-node **'cabxabcd' to 'c' -> [abxabcd, d]**. Since a new node is inserted, `remainder = remainder - 1` (2) and `active_point[1] = 'cd'`
![Sample "abcabxabc" -- 10 - 2](SuffixTree-Sample-abcabxabc-10-2.JPG)
5. At this point, **active_node** reduce to empty and does not have any link, then we reset **active_point[0] = 'root'
6. Insert 'cd', start from the current node (active_point[0]), which is the 'root', query its sub-nose for active_point[1] ('cd'). Node 'cabxabcd' contains target character 'c' and mismatches at offset 1. Split **'cabxabcd' to c -> [abxabcd, d]**. **remainder = remainder - 1** (1).                         
![Sample "abcabxabc" -- 10 - 3](SuffixTree-Sample-abcabxabc-10-3.JPG)

7. Finally insert 'd' directly on root node and `remainder = remainder - 1` which is 0
![Sample "abcabxabc" -- 10 - 4](SuffixTree-Sample-abcabxabc-10-4.JPG)
