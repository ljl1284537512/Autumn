### ReentrantLock(可重入锁) 原理与实践



## 目录 

- [简介](#简介)
- [性能考虑因素](#性能考虑因素)
- [公平性](#公平性)
- [锁释放](#锁释放)



## 简介

ReentrantLock 自java 1.5引入，ReentrantLock并不是一种替代内置加锁的方法，而是当内置锁机制不适用时，作为一种可选择的高级功能。ReentrantLock 实现了Lock接口，并提供了与synchronized相同的互斥性和内存可见性。

为什么要创建一种与内置锁如此相似的加锁机制？ 在某些情况下，例如：

1. 无法中断一个正在等待获取锁的线程；
2. 无法在请求获取一个锁时无限地等待下去；



## 性能考虑因素

> 性能是一个不断变化的指标，如果在昨天的测试基准中发现X比Y更快，那么在今天就可以能已经过时了。

当把ReentrantLock添加到Java 5.0时，它能比内置锁提供更好的竞争性能。

> 对于同步资源来说，竞争性能时可伸缩性的关键要素：如果有越多的资源被耗费在锁的管理和调度上，那么应用程序得到的资源就越少。锁的实现方式越好，将需要越少的系统调用和上下文切换，并且在共享内存总线的内存同步通信量也越少，而一些耗时的操作将占用应用程序的计算资源。



## 公平性

> 在公平锁中，线程将按照它们发出请求的顺序来获得锁；在非公平锁中，当一个线程请求非公平锁时，如果在发出请求的同时该锁的状态变为可用，那么这个线程将跳过队列中所有的等待线程并获得锁。

ReentrantLock的构造函数中提供了两种公平性选择，默认为非公平锁：

```java
//默认创建非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

##### 非公平锁实现：

```java
static final class NonfairSync extends Sync {

    final void lock() {
        if (compareAndSetState(0, 1))
            //标识当前线程已经占用了排它锁
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

lock方法首先通过改变状态来占用锁，如果设置成功了，则表明获得了锁，如果占用失败了，则执行acquire()方法, acquire方法由AQS实现：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && //再次尝试获取锁，如果还是失败了，则准备加入队列
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire在NonfairSync中实现，最终调用Sync中的nonfairTryAcquire方法：

```java
//线程再次尝试获取锁，如果仍然失败了，则返回false，如果成功了返回true;
//这里返回true，还有另外一种意思，当前线程已经占用了锁，那么重入次数+1
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //获取一次状态，如果状态为0，则说明锁还没有被占用
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            //第二次占用锁成功
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { //当前占用线程的锁，就是它自己
        //nextc为锁重入的次数，每次调用nonfairTryAcquire方法，自增acquires
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded"); //锁重入次数达到最大限制，默认为int最大值
        //设置当前状态 = 重入的次数
        setState(nextc);
        //返回成功
        return true;
    }
    return false;
}
```

如果锁在tryAcquire之后，仍然获取失败，则准备加入队列

```java
acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
```

将当前线程封装成一个Node，**加入到队列的尾部**：

```java
private Node addWaiter(Node mode) {
    // 封装当前线程 -> Node
    Node node = new Node(Thread.currentThread(), mode);
    // 获取尾部节点，尝试将Node直接链接到尾节点上
    Node pred = tail;
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

如果当前线程无法通过链接到尾部节点，则通过自旋，要么直接作为等待的第一个线程(head = tail = currentThread) ， 要么链接到队列的尾部：

```java'
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

节点已经封装好了，这个时候再看acquireQueued的实现：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { //获取锁成功了
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //判断是否需要阻塞当前线程
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

shouldParkAfterFailedAcquire判断在获取锁失败之后，是否需要阻塞当前线程；

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```



##### 公平锁实现

```java
static final class FairSync extends Sync {
	//流程是在AQS中定义的：
    //1. 尝试获取锁；
    //2. 将当前线程封装成Node节点，然后加入队列尾部；
    //3. 判断当前线程是否需要被park，或者interupt
    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
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
}
```

公平锁，在acquire中，不会像非公平锁一样-上来就直接加锁：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //重点：在没有前置等待线程的情况下，才会取尝试获取锁
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //处理锁重入
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

tryAcquire里的hasQueuedPredecessors实现，在尝试获取锁之前，要保证没有前置节点已经处于等待的状态；

```java
//当队列中等待的第一个节点就是当前线程，或者是等待队列为空
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    //1. 头尾节点不一致
    return h != t &&
        //2. 头节点之后没有节点 或者 是头节点之后的节点 不是 当前线程
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

公平锁，后续的步骤与非公平锁一致，这里使用了典型的模块方法模式，由AQS定义模板：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

由FairSync与NonFairSync各自实现tryAcquire，公平锁需要等待前无等待节点，才可以做获取锁操作，而非公平锁可以在尝试获取锁的时候，不用考虑当前锁是否已经有线程在等待的情况；在上述锁获取失败的情况下，都是要将当前线程封装成Node节点，并加入等待队列。通过当前状态，来判断是否要



## 锁释放

那些等待队列中的线程怎么办？









