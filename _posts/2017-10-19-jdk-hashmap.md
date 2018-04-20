---
layout: post
title: jdk-HashMap浅显探索
category: jdk
---

本文探索最常用到的方法的基本实现，其中关于红黑树的各种操作，这里不涉及。

先说下，HashMap底层是数组+链表/红黑树，即Node[]数组，Node是一个key-value的封装类，还有指向下一Node的指针。当HashMap中的元素数量超过capacity * loadFactor之后，进行resize，resize对于空数组，进行初始化分配，否则就进行2倍扩容和rehash。get时，直接拿table[key.hash & (n-1)]的值，若不是单Node，则通过链表/红黑树去找。put时，也是先看table[key.hash & (n-1)]是否为空，若不为空，则通过链表或红黑树插入，链表插入还需要考虑到>8转红黑树，以及插入完看是否需要resize。remove和put类似，只不过是个相反过程。


**几个关键数值**：
- 初始容量（2^4）
- 最大容量（2^30）
- 默认的负载因子（0.75）
- Treeify阈值，超过8后链表转红黑树
- UnTreeify阈值，小于6树转链表


Node类，实现了Map.Entry接口，把 <K, V>封装起来，构成Node链表。

**暴露出去的公共方法**：
- 初始化，多个重载版本，可不指定/指定初始容量/指定初始容量+负载因子，或以另一个map去初始化此map
- `get(key)`，调用final getNode
- `containsKey`，判断上面getNode返回值是否为空
- `put`，调用final putVal方法
- `remove`，调用final removeNode方法
- `clear`，遍历table，table[i] = null
- `containsValue`，遍历table，再遍历每个桶


**final方法**:
- `getNode(k.hashCode(), k)`，先通过hash&(table.length-1)找出在数组中的位置，然后判断其第一个元素是否hash相等且key相等，不然，若第一个元素 instance of TreeNode, 调用TreeNode的查找，不然就链表式一个个往后找。
- `putVal`，有则覆盖，没有则插入，若是本身就为Tree，则直接用TreeNode的插入，否则链表插入再判断是否Treeify，++size若大于阈值，则resize
- `reisze`，初始化 或 每次乘2。遍历table数组，
	- 若只有一个Node，重新hash，放在[hash & (newLen-1)]位置
	- 若是TreeNode，进行分裂
	- 若是链表，重新hash的值 & oldLen，若为0，则落入原位置，否则因原桶中很可能重叠，故在原来索引基础上便宜oldLen的位置。也就是说，通过把hash值大致二分来重新判断落入的位置，减少冲突的概率。
- `removeNode`，跟putVal逻辑类似，只不过是进行删除，如果红黑树的话，调用removeTreeNode，里面有判断是否进行UnTreeify。

***

注意到，HashMap的实现中，包括Node[] table本身，entrySet还有很多字段，都被声明为**`transient`**，这里有一篇[可能有用的解答](https://segmentfault.com/q/1010000000630486)，也就是出于两个原因：
1. 不同JVM有不同的hash实现，为了防止跨JVM的时候同一个字符串分布在不同位置，要阻止其序列化和反序列化。
2. 效率。因为HashMap本质是存储key-value，那么其他的一切空间都只是辅助。
