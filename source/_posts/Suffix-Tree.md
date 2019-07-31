---
title: Suffix Tree
date: 2019-07-22 20:07:12
tags: [Suffix Tree]
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

- **active point**: a structure indicates `parent_node`, `active_value` and `offset`
- **remainder**: remaining characters to be inserted

```
active_point = (root, '', 0)
remainder = 0
```

### The first character, 'a'

We append ` a ` directly to the root node
>1. before we append the node, set remainder = remainder + 1, states that we are going to insert a new node
>2. append 'a' to the root node
>3. after the insertion, a new node is appended to the tree, hence remainder = remainder - 1
>4. after the whole procudure, active_point = (root, '', 0), remainder = 0
![Sample "abcabx" -- abc](SuffixTree-Sample-a.JPG)

### The second character, 'b'

>1. before we append the node, set remainder = remainder + 1, states that we are going to insert a new node
>2. expend every leave node; hence change 'a' to 'ab'
>3. query the current node, which is the root node, since there is no prefix starts with 'b', we need to create a new node and insert 'b'. (The prefix for the previous prefix will be [a, ab] now)
>3. after the insertion, a new node is appended to the tree, hence remainder = remainder - 1
>4. after the whole procudure, active_point = (root, '', 0), remainder = 0
![Sample "abcabx" -- abc](SuffixTree-Sample-ab.JPG)

### The third character, 'c'

> same as the procedure of inserting 'b'
![Sample "abcabx" -- abc](SuffixTree-Sample-abc.JPG)

### The fourth character, 'a' again

This time we encounter the same character we have already inserted
>1. before we append the node, set remainder = remainder + 1, states that we are going to insert a new node
>2. expend every leave node; hence change 'abc' -> 'abca', 'bc' -> 'bca', 'c' -> 'ca'
>3. query the current node, which is the root node, find out the there already exist a prefix which starts from 'a'; hence we modify the `active_point`:
>> **parent_node**: the parent node is still root node hence remain unchanged
>> **active_value**: the current value will be set to 'a'
>> **offset**: the offset of the current value, which is 1 (states the first character in the prefix)
>3. after the insertion, no new node is inserted in the tree, hence remainder keep the same (remainder = 1)
>4. after the whole procudure, active_point = (root, 'a', 1), remainder = 1

### The fifth character, 'b' again

Still duplicated with the previous nodes
>1. before we append the node, set remainder = remainder + 1, states that we are going to insert a new node
>2. expend every leave node; hence change 'abca' -> 'abcab', 'bca' -> 'bcab', 'ca' -> 'cab', **active_node**[1] = 'a' -> 'ab'
>3. query the current node, which is the root node, find out the there already exist a prefix which starts from 'ab' (which is exactly 'ab'); hence we modify the `active_point`:
>> **parent_node**: the parent node is still root node hence remain unchanged
>> **active_value**: the active value will be 'ab'
>> **offset**: the offset of the current value, which is 2
>3. after the insertion, no new node is inserted in the tree, hence remainder keep the same (remainder = 1)
>4. after the whole procudure, active_point = (root, 'ab', 2), remainder = 2

By this step, we still have the same amount of nodes inserted into the tree (3 nodes)
![Sample "abcabx" -- abcab](SuffixTree-Sample-abcab.JPG)

### The sixth character, 'x'

>1. before we append the node, set remainder = remainder + 1, states that we are going to insert a new node (remainder = 3 now)
>2. expend every leave node; hence change 'abcab' -> 'abcabx', 'bcab' -> 'bcabx', 'cab' -> 'cabx', **active_node**[1] = 'ab' -> 'abx'
>3. query the current node, which is the root node, find out the there isn't a prefix which starts from 'abx'; hence we **split**:
>> **parent_node**: start from the parent node, query its sequential sub nodes
>> **active_value**: look for the sub prefix which starts from the active_value 'abx' (which will be 'abcabx')
>> **offset**: use the offset the locate the split point (which is the the second node)
>> modify **active_node**: parent_node = 'root', active_value = 'bx', offset = 1
![Sample "abcabx" -- abcabx](SuffixTree-Sample-abcabx.JPG)
>3. after the insertion, a new node is inserted in the tree, hence remainder = remainder - 1 (remainder = 2)
>4. The remainder is still greater than 0; hence we keep insert 'bx'
>> do the same thing here, start from the **parent_node**, query the sub prefix 'bx'. Found 'bx' in the prefix 'bcabx'; hence split the node according to **offset**
>> Split the target node, Since a new node is inserted, remainder = remainder - 1 (remainder = 1 now)
>> modify **active_node**: parent_node = 'root', active_value = 'x', offset = 0
>5. after the procudure, active_point = (root, 'x', 0), remainder = 1. Since remainder is still greater than 0, we have to do another insertion to reduce it to zero. Append 'x' directly to the current node (which is 'b').
> a new node is inserted, hence remainder = remainder - 1 (which is 0 now)
> **active_node**: parent_node = 'root', active_value = '', offset = 0; **remainder**: 0
![Sample "abcabx" -- abcabx - 2](SuffixTree-Sample-abcabx-2.JPG)

After the insertion of the sixth character 'x', we splitted two nodes.
> Is there a relationship between two splitted node 'ab' and 'b'?
> When we separated 'ab' and 'b', we modified the previous 'abcabx' node first, then modified the second node 'bcabx'. If we encounter another string starting with 'abk', then after 'abk' is inserted, we definately need to insert 'bk' as well. Repeating the same query will slow down the whole procedure; thus, we can add a link from 'ab' to 'b' to indicate that if an update is made in node 'ab', then another update will be followed on the link target 'b' afterwards.
![Sample "abcabx" -- abcabx - 3](SuffixTree-Sample-abcabx-3.JPG)

### The seventh character, 'a'

Same as the fourth insertion 'a'
> active_point = (root, 'a', 1), remainder = 1

### The eighth character, 'b'

Same as the fifth insertion 'b'
> active_point = (root, 'ab', 2), remainder = 2

### The ninth character, 'c'

When we try to insert 'c'
>1. remainder= remainder + 1 (3)
>2. Expand every leave node
![Sample "abcabxabc" -- abcabxabc](SuffixTree-Sample-abcabxabc.JPG)
>3. Query 'abc', find 'abc' under the parent node 'ab'
>> Extra Procedure:
>> **active_point**: The node 'cabxabc' constains the last character of the target prefix ('abc'), whose parent node is 'ab'; thus, set `active_point[0] = 'ab'`.
>> **active_node**: set `active_point[1] = 'abc'`.
>> **offset**: the offset of the character 'c' in its node string('cabxabc') which is 1. Set `active_point[2] = 1`
> After the modification, no new node is inserted, hence **remainder** keeps unchanged.
> active_point = ('ab', 'abc', 1), remainder = 3

### The last character, 'd'

We will use the link built-up in previous steps to simplify the updates.


