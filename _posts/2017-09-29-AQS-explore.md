---
layout: post
title: jdk-AQS探索
category: jdk
---
{% include JB/setup %}

AQS是一个基于FIFO的框架，我是在ReentrantLock的实现中去理解AQS的acquire过程。这里先看下AQS的acquire，再对ReentrantLock的公平锁/非公平锁尝试分析。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

1. `tryAcquire`是AQS的一个接口，在ReentrantLock中有 `FairTryAcquire`和`NonFairTryAcquire`两种，略有差别，但都是一次尝试返回true/false，这里如果一次获取不到，则执行后面。
2. addWaiter(node)，把当前node设成独占模式，插入到双向链表的尾部，CAS自旋来保证线程安全。
3. acquireQueued，CAS自旋等待，直到当前node为head的后继结点并tryAcquire成功。

***

非公平锁的`tryAcquire`
```java
protected final boolean tryAcquire(int acquires) {
    // 调用非公平锁的tryAcquire
    return nonfairTryAcquire(acquires);
}
```
再进入到nonfairTryAcquire
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // getState是AQS中的volatile int，表示加锁状态。0 是未被持有。
    int c = getState();
    if (c == 0) {
        // 锁未被持有，则通过CAS设置当前线程为持有者，如果有两个线程同时进入到这里，只有一个会成功，另一个失败。
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 锁被当前线程持有，可重入，上限是int的最大值
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
在我看来，c是判断是否被持有：若未被持有（==0）则通过CAS设置当前线程独占拥有;若被持有，先看是否被当前线程持有，因为可重入锁所以可以被同一线程继续持有，持有次数上限是Integer.MAX_VALUE;若非当前线程持有，则返回false。

再看下公平锁的tryAcquire
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 唯一区别是在锁未被持有情况尝试CAS前，判断是否有前驱节点在等待。
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
和非公平锁的tryAcquire类似，唯一区别是在未被持有尝试CAS获取的情况下，要先判断前面是否有排队，以此实现公平。

***

再来探索addWaiter
```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    //首次进来时，tail为空，直接enq(node)
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
首次执行addWaiter时，tail为空，进到enq(node)，在enq方法中，把node节点加入到双向链表的尾部。如下：
```java
private Node enq(final Node node) {
    // 无限循环配合CAS：CAS自旋，以实现线程安全
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            // 首次时，双向链表为空，新建head节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 再次进入时，把node加入到队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
enq做的事情就是把node插入到head<->node1<->node2<->.... 双向链表的队尾，以CAS自旋保证线程安全。

***

然后探索acquireQueued
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 表示是否被中断
        boolean interrupted = false;
        // 又见CAS自旋，只有获取成功时才会跳出，不然就一直挂在这里，具体是什么形式挂，我目前还不清楚
        for (;;) {
            final Node p = node.predecessor();
            // node的前驱节点为head 且 获取成功
            if (p == head && tryAcquire(arg)) {
                // 把node设成head，跳出
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
跳出循环的条件就是当前节点的前驱为head且tryAcquire成功；若不满足此条件，则进入第二个if，里面先后有两个方法，分别一窥。

在看以下两个方法前，应该先看下Node的几种状态域。
- SINGNAL，当前node的后继被阻塞，所以当前节点释放release或取消cancel时，必须unpark它的后继节点。
- CANCELLED(唯一大于0的)，由于超时或中断被取消，不再阻塞。我理解的是可以直接踢走了。
- CONDITION，暂时不懂。
- PROPAGATE，暂时不懂。
- 0

那么再看第一个方法`shouldParkAfterFailedAcquire`
```java
// the main signal control in all acquire loops
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        // 前驱是SIGNAL状态，所以可以放心地在它后面等待着被unpark
        return true;
    if (ws > 0) {
        // 前驱是CANCELLED状态，则一直往前找，直到非CANCELLED状态的节点，挂在它后面，中间的CANCELLED节点会被GC
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // CAS把前驱设成SIGNAL，等待前驱通知？这里不明白
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

在上面返回true的情况下，进入下面第二个方法
```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```
调用park使线程进入waiting状态。从waiting状态唤醒，可以被unpark，也可以通过中断。这里理解的不够清晰。

***

以上，是AQS的acquire实现。acquire在ReentrantLock中的调用入口有两处，都是在`lock`方法中。

非公平锁：
```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
这比较容易理解了，因为非公平锁，所以一上来就先尝试获取锁，获取不到才进acquire，但这里进了acquire也不是公平的，因为非公平锁tryAcquire的实现是不公平的，上面已经分析过了。

公平锁：
```java
final void lock() {
    acquire(1);
}
```
公平锁直接调acquire，同时因为公平锁tryAcquire相比于非公平锁多了一个CAS前先判断是否有前驱节点，因此实现公平。

对于addWaiter, acquireQueued部分，公平/非公平锁都一样。

***

QA
**AQS中FIFO的实现为何要用双向链表？**
单纯FIFO的话，单链表加尾指针就足够。但注意到`shouldParkAfterFailedAcquire`方法，需要通过前驱指针来剔除前面状态为CANCELLED的节点，单向链表显然不足够。

以上，是这两天读ReentrantLock和AQS后对lock实现的个人理解，参考过[这篇文章](http://www.cnblogs.com/waterystone/p/4920797.html)，对于疏漏或错误之处，在后期有更深刻认识后会尽快修复。