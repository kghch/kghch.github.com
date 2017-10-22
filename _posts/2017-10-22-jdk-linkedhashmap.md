---
layout: post
title: jdk-LinkedHashMap浅显探索
category: jdk
---
{% include JB/setup %}

LinkedHashMap是继承自HashMap的子类，功能上是迭代遍历时按插入顺序（默认）或LRU顺序，这个由accessOrder这个参数初始化LinkedHashMap时指定。

哈希表的实现复用了HashMap，迭代遍历顺序是通过给每个Node增加一个前后指针(before, after)并控制连接关系，在整个连接中保留首尾指针(head, tail)来实现顺序遍历，这个顺序在初始化时通过指定accessOrder就**决定**了是插入顺序还是LRU顺序。

**初始化**
跟HashMap类似，可以不指定、指定初始容量、指定初始容量+负载因子，除此之外，LinkedHashMap可指定accessOrder（布尔值），即访问顺序，默认false为插入顺序，true为LRU顺序。

**get方法**
重写了HashMap的get方法，`getNode`之后，如果`accessOrder`为true，调用`afterNodeAccess`方法。

**put方法**
没重写，复用HashMap的put方法。

**remove方法**
没重写，复用。

**clear方法**
重写，除调用HashMap.clear外，还把首尾指针都置空。

下面是HashMap源码中的三个空函数，注释中写明了是为了给LinkedHashMap回调用。
```java
// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

- `afterNodeAccess`，只有指定为LRU顺序时才会执行，把被访问的p节点从链表原位置删除，移到链表尾部。
- `afterNodeInsertion`，新增一个Node时，如果容量不够，需要把头结点删除，这里因为if条件中有一条是默认返回false，故不执行。
- `afterNodeRemoval`，把删除节点p的前后节点链起来。

以上，LinkedHashMap就是HashMap + 前后指针 维护出的顺序。`插入顺序` 和 `LRU顺序` 的主要区别在于，afterNodeAccess的实现，在get某Node p后，在LRU顺序下，会把p从链表中删除并移到尾部。