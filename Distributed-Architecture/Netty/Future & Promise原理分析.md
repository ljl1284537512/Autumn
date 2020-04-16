# Futrue & Promise

## 构造Promise
`Channel`注册过程， 会调用`NioEventLoopGroup`创建一个`DefaultChnnalPromise`, 下面的`this`代表`NioEventLoopGroup`。
```java
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```
对应的`DefaultChannelPromise`构造如下
```java
public DefaultChannelPromise(Channel channel, EventExecutor executor) {
    // 注入Executor
    super(executor);
    // 注入Channel
    this.channel = checkNotNull(channel, "channel");
}
```
![](_v_images/20191107074541386_31557.png =700x)
`DefaultChannelPromise`继承自`DefaultPromise`, 而设置`Exector`是在`DefaultPromise`中进行(初步猜测，Netty的`Promise`是基于`Exector`来实现的) , `Channel`属性是在`DefaultChannelPromise`中定义，所以由当前类-`DefaultChannelPromise`来设置`Channel`。
> 为什么要传入`Executor`?

继续看注册操作:
```java
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```
1. 首先获取`Promise`中设置的`Channel`, `NioServerSocketChannel`的`unsafe`是在其父类`AbstractNioChnanel`中实现；
![](_v_images/20191107075533863_15383.png =600x)
```java
io.netty.channel.nio.AbstractNioChannel#unsafe
@Override
public NioUnsafe unsafe() {
    return (NioUnsafe) super.unsafe();
}
```
`AbstractNioChannel`调用其父类`AbstractChannel`的`unsafe`方法
```java
@Override
public Unsafe unsafe() {
    return unsafe;
}
```
而`AbstractChannel`的`unsafe`是通过其构造方法得到的
```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```
注册操作接下来调用`io.netty.channel.AbstractChannel.AbstractUnsafe#register`
```java
 @Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        // 当前线程并发EventLoop线程，则不作register0操作，将它扔给EventLoop来执行
        eventLoop.execute(new Runnable() {
            @Override
            public void run() {
                register0(promise);
            }
        });
    }
}
```
这个时候`register`操作就完成了。

## 同步阻塞
当最终调用`sync`方法时：
```java
@Override
public ChannelPromise sync() throws InterruptedException {
    super.sync();
    return this;
}
```
`DefaultChannelPromise`首先调用父类`DefaultPromise`的`sync()`方法
```java
@Override
public Promise<V> sync() throws InterruptedException {
    // 线程阻塞
    await();
    rethrowIfFailed();
    return this;
}
```
来看下具体阻塞线程的部分
```java
@Override
public Promise<V> await() throws InterruptedException {
    if (isDone()) {
        // 已经完成了，就直接返回
        return this;
    }

    if (Thread.interrupted()) {
        // 线程被中断过，直接抛出异常
        throw new InterruptedException(toString());
    }

    // 检查死锁(暂不清楚)
    checkDeadLock();

    synchronized (this) {
        // 只要还没有完成
        while (!isDone()) {
            // 递增waiters数值
            incWaiters();
            try {
                // Object.wait()方法执行阻塞
                wait();
            } finally {
                decWaiters();
            }
        }
    }
    return this;
}
```

## 线程唤醒
当然注册最终会调用`AbstractChannel`中的`register0`方法
```java
private void register0(ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }
    boolean firstRegistration = neverRegistered;
    doRegister();
    neverRegistered = false;
    registered = true;

    pipeline.invokeHandlerAddedIfNeeded();

    safeSetSuccess(promise);
    pipeline.fireChannelRegistered();

    if (isActive()) {
        if (firstRegistration) {
            pipeline.fireChannelActive();
        } else if (config().isAutoRead()) {
            beginRead();
        }
    }
}
```
`safeSetSuccess(promise)`中尝试设置`Promise`的成功标识
```java
protected final void safeSetSuccess(ChannelPromise promise) {
    if (!(promise instanceof VoidChannelPromise) && !promise.trySuccess()) {
        logger.warn("Failed to mark a promise as success because it is done already: {}", promise);
    }
}
```
`trySuccess()`最终会调用`DefaultPromise`类，操作如下：
```java
@Override
public boolean trySuccess(V result) {
    // 设置当前promise的状态
    if (setSuccess0(result)) {
        // 唤醒监听者
        notifyListeners();
        return true;
    }
    return false;
}
```
`notifyListeners`操作如下：
```java
private void notifyListeners() {
    EventExecutor executor = executor();
    if (executor.inEventLoop()) {
        final InternalThreadLocalMap threadLocals = InternalThreadLocalMap.get();
        final int stackDepth = threadLocals.futureListenerStackDepth();
        if (stackDepth < MAX_LISTENER_STACK_DEPTH) {
            threadLocals.setFutureListenerStackDepth(stackDepth + 1);
            try {
                notifyListenersNow();
            } finally {
                threadLocals.setFutureListenerStackDepth(stackDepth);
            }
            return;
        }
    }

    safeExecute(executor, new Runnable() {
        @Override
        public void run() {
            notifyListenersNow();
        }
    });
}
```

## 疑问
1. Promise中的Executor是干嘛用的？
>  the {@link EventExecutor} which is used to notify the promise once it is complete. It is assumed this executor will protect against {@link StackOverflowError} exceptions. The executor may be used to avoid {@link StackOverflowError} by executing a {@link Runnable} if the stack depth exceeds a threshold.
