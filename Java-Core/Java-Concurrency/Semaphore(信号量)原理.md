### Semaphore(信号量)原理

## 目录
- [简介](#简介)
- [简单示例](#简单示例)
- [Semaphore原理](#Semaphore原理)
- [Semaphore源码分析](#Semaphore源码分析)


## 简介
计数信号量(counting semaphore)用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。计数信号量还可以用来实现某种资源池，或者对容器实行编解；Semaphore为共享锁实现。

## 简单示例

一个简单的例子，同一个时刻只允许最多两个线程执行某个代码块；
```java
package com.risesun.test;

import java.util.concurrent.Semaphore;

public class SemaphoreTest implements Runnable{
	// 允许共享锁的线程数
    Semaphore semaphore = new Semaphore(2);
    public static void main(String[] args) {
        new SemaphoreTest().build();
    }
    public void build(){
        for(int i= 0 ; i< 10 ;i ++)
            new Thread(this,"Thread-"+i).start();

        new Thread(()->{
            while(true){
                System.out.println("过去1s....");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

    @Override
    public void run() {
        try {
            semaphore.acquire();
            System.out.println("已经获取数据库连接..."+Thread.currentThread().getName());
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            semaphore.release();
        }
    }
}
 
```
执行结果如下：
```java
已经获取数据库连接..Thread-0
已经获取数据库连接..Thread-1
过去1s....
过去1s....
已经获取数据库连接..Thread-2
已经获取数据库连接..Thread-3
过去1s....
过去1s....
已经获取数据库连接..Thread-5
已经获取数据库连接..Thread-4
```

### Semaphore原理
在实例化Semaphore时，默认使用非公平锁，以AQS内部的State变量作为许可总数，在尝试获取共享锁时，通过CAS递减State变量，如果State值不足以减去当前需要持有共享锁的数量，则获取锁失败，反之，则成功获取锁。

> 这个State的设计，真的很棒！既为ReentrantLock实现了重入功能，维护了线程的状态，还为Semaphore实现了共享锁功能。Semaphore不支持锁重入。

### Semaphore源码分析

Semphore实例化时，默认使用**非公平锁**；
```java
public Semaphore(int permits) {
	sync = new NonfairSync(permits);
}
```
先来看下NonfairSync的实现
```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
			// 设置State = permits
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
```

因为NonfairSync继承自Sync，整体先大致浏览下Sync的函数
```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
			// 同样是设置共享锁的数量
            setState(permits);
        }

        final int getPermits() {
			// 获取共享锁目前可再次共享的线程数量
			// 有点绕（每个线程执行一次）
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
			// 死循环
            for (;;) {
				// 获取当前可进入的线程数(共享锁)
                int available = getState();
                int remaining = available - acquires;
				// remaining<= 代表共享锁可同时进入的线程数不足以满足当前需要使用
				// 共享锁的acquires量（它不一定是1）
                if (remaining < 0 ||
					// 如果remaining >= 0 , 则代表目前还可以继续使用共享锁
                    compareAndSetState(available, remaining))
					// 返回可用共享锁线程数目结果
                    return remaining;
            }
        }
		// 尝试释放共享锁
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
				// 获取当前状态(还可使用共享锁的线程总数)
                int current = getState();
				// 把当前释放的线程数目加上去
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
					// 释放共享锁成功
                    return true;
            }
        }
		// 减少许可总数
		// 和nonfairTryAcquireShared的区别？当当只是减少许可，并没有
		// 新的线程获取许可，如果是nonfairTryAcquireShared, 那么线程在获取
		// 许可之后，肯定会释放许可；
        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }
		// 将该共享锁，变成再也占用不了的锁
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }
```
我们再来看Semphore中NonfairSync的tryAcquireShared实现就是尝试获取共享锁
```java
protected int tryAcquireShared(int acquires) {
	return nonfairTryAcquireShared(acquires);
}
```
以简单示例为主，Semaphore实例化的时候，设定内部默认使用非公平锁，下一个调用的地方就是:
```java
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```
而sync.acquireSharedInterruptibly在AQS中实现
```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
		// 尝试获取锁，如上NonfairSync的tryAcquireShared实现
        if (tryAcquireShared(arg) < 0)
			// 如果获取共享锁失败
            doAcquireSharedInterruptibly(arg);
    }
```
获取共享锁失败之后，加入等待队列，等待再次获取共享锁；
```java
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
				// 再次尝试获取共享锁
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
				// 获取共享锁失败之后，需要park吗？
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
