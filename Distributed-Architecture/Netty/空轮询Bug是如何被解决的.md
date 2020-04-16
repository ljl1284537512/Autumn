# 空轮询Bug是如何解决的
## 什么是空轮训Bug?
## Netty如何解决？
我们都知道Netty服务端启动之后，是由EventLoop无限循环来接收客户端请求的，我们看NioEventLoop的run方法实现：
```java
@Override
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
            }
            ... 校验是否有事件被轮询出来
        } catch (Throwable t) {
            handleLoopException(t);
        }
        ...
    }
}
```
我们先看calculateStrategy的实现：
```java
final class DefaultSelectStrategy implements SelectStrategy {
    static final SelectStrategy INSTANCE = new DefaultSelectStrategy();

    private DefaultSelectStrategy() { }

    @Override
    public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
        return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
    }
}
```
这是一种选择的策略，由如上calculateStrategy方法的实现可看出其规则是：
任务队列中是否是任务？如果有的话，返回, 否则返回SelectStrategy.SELECT;而当返回SelectStrategy.SELECT之后，外部的switch检测到了，并执行：
```java
select(wakenUp.getAndSet(false));
if (wakenUp.get()) {
    selector.wakeup();
}
```
### select
```java
private void select(boolean oldWakenUp) throws IOException {
    // 获取selector
    Selector selector = this.selector;
    // 对选择进行计数
    int selectCnt = 0;
    // 记录系统当前时间
    long currentTimeNanos = System.nanoTime();
    // 计算最晚的选择时间
    // 如果有定时任务，则返回最近定时任务执行的时间
    long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

    for (;;) {
        // 判断其是否超时（或者定时任务是否还有0.5ms就要执行了）
        long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
        if (timeoutMillis <= 0) {
            if (selectCnt == 0) {
                // 还没做过select, 赶紧做一次
                selector.selectNow();
                // 累计选择计数值
                selectCnt = 1;
            }
            break;
        }

        // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
        // Selector#wakeup. So we need to check task queue again before executing select operation.
        // If we don't, the task might be pended until select operation was timed out.
        // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
        // 中途有新的任务加入队列，则唤醒，退出selector
        if (hasTasks() && wakenUp.compareAndSet(false, true)) {
            selector.selectNow();
            selectCnt = 1;
            break;
        }
        // 继续持续选择，同步等待；
        int selectedKeys = selector.select(timeoutMillis);
        selectCnt ++;
        // 捕获到事件发生，或者其他情况，则终止for循环
        if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
            // - Selected something,
            // - waken up by user, or
            // - the task queue has a pending task.
            // - a scheduled task is ready for processing
            break;
        }
        if (Thread.interrupted()) {
            // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
            // As this is most likely a bug in the handler of the user or it's client library we will
            // also log it.
            //
            // See https://github.com/netty/netty/issues/2426
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely because " +
                        "Thread.currentThread().interrupt() was called. Use " +
                        "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
            }
            selectCnt = 1;
            break;
        }

        long time = System.nanoTime();
        if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
            // 过了最后选择时间，并且仍然没有捕获到感兴趣的事件
            // timeoutMillis elapsed without anything selected.
            selectCnt = 1;
        } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
            // 不超时的情况下，超过了最多选择次数
            // The selector returned prematurely many times in a row.
            // Rebuild the selector to work around the problem.
            logger.warn(
                    "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                    selectCnt, selector);
            // 重新建立Selector
            rebuildSelector();
            selector = this.selector;

            // Select again to populate selectedKeys.
            // 再次尝试选择器
            selector.selectNow();
            selectCnt = 1;
            break;
        }
        currentTimeNanos = time;
    }
}
```
> netty为了保证任务队列能够及时执行，在进行阻塞select操作的时候会判断任务队列是否为空，如果不为空，就执行一次非阻塞select操作，跳出循环
###  delayNanos
```java
protected long delayNanos(long currentTimeNanos) {
    ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
    if (scheduledTask == null) {
        // 默认SCHEDULE_PURGE_INTERVAL = TimeUnit.SECONDS.toNanos(1);
        return SCHEDULE_PURGE_INTERVAL;
    }

    return scheduledTask.delayNanos(currentTimeNanos);
}
```
默认的持续时间为1s;

### 选择次数统计
Netty对选择的次数会进行统计，如果在不超时的情况下，选择的次数过多，则会有相应的处理，这个最多选择的次数是通过NioEventLoop初始化得来的：
```java
int selectorAutoRebuildThreshold = SystemPropertyUtil.getInt("io.netty.selectorAutoRebuildThreshold", 512);
if (selectorAutoRebuildThreshold < MIN_PREMATURE_SELECTOR_RETURNS) {
    selectorAutoRebuildThreshold = 0;
}

SELECTOR_AUTO_REBUILD_THRESHOLD = selectorAutoRebuildThreshold;
```
我们发现其可配置，并且默认值是512, 而最小的选择次数是3，如果配置的值<3, 则直接为0（为0的话就不会尝试重新建立Selector）.

### rebuildSelector
```java
io.netty.channel.nio.NioEventLoop#rebuildSelector
public void rebuildSelector() {
    if (!inEventLoop()) {
        execute(new Runnable() {
            @Override
            public void run() {
                rebuildSelector0();
            }
        });
        return;
    }
    rebuildSelector0();
}
io.netty.channel.nio.NioEventLoop#rebuildSelector0
private void rebuildSelector0() {
    final Selector oldSelector = selector;
    final SelectorTuple newSelectorTuple;
    try {
        // 1. 开一个新的Selector
        newSelectorTuple = openSelector();
    } catch (Exception e) {
        logger.warn("Failed to create a new Selector.", e);
        return;
    }

    // 将老选择器上注册的所有通道都注册到新的选择器上
    int nChannels = 0;
    for (SelectionKey key: oldSelector.keys()) {
        Object a = key.attachment();
        try {
            if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                continue;
            }

            int interestOps = key.interestOps();
            // 2. 取消key在旧的选择器上的注册
            key.cancel();
            // 3. 注册到新的选择器上
            SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
            if (a instanceof AbstractNioChannel) {
                // Update SelectionKey
                ((AbstractNioChannel) a).selectionKey = newKey;
            }
            nChannels ++;
        } catch (Exception e) {
            logger.warn("Failed to re-register a Channel to the new Selector.", e);
            if (a instanceof AbstractNioChannel) {
                AbstractNioChannel ch = (AbstractNioChannel) a;
                ch.unsafe().close(ch.unsafe().voidPromise());
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                invokeChannelUnregistered(task, key, e);
            }
        }
    }
    selector = newSelectorTuple.selector;
    unwrappedSelector = newSelectorTuple.unwrappedSelector;
    // 4. 最后关闭旧的选择器
    oldSelector.close();
}
```
整体的过程是：
1. 开一个新的Selector；
2. 取消该key在旧的selector上的事件注册；
3. 将key对应的channel注册到新的选择器上；
4. 最后关闭旧的选择器；

## 总结
Netty对选择器设定过期时间，如果选择的过程中超出了过期时间范围，则直接往后执行（后续有可能是执行queue中的任务或者是继续选择）;如果未超过过期时间范围，但是超过了选择的次数（默认是512），则直接重新建立Selector, 并且将旧的Selector上注册的Channel重新注册到新的Selector上，并关闭旧的Selector。

### 跳出循环select的条件
中断其循环select的条件有如下：
1. 定时任务截止时间快到了;
2. 有新任务加入队列；
3. 轮询到IO事件发生；
4. 用户通过wakeup主动去唤醒；
5. oldWakeUp参数=true;

### 用户如何通过wakeup主动唤醒eventLoop线程？
在select方法中，我們發現，如果timeoutMillis足夠長的話，那麽有可能會阻塞很長時間，這個時候，如果有新的任务加入，就无法即时处理了，真的是这样的吗？
```java
int selectedKeys = selector.select(timeoutMillis);
```
用户通过如下方法添加新的任务时：
```java
io.netty.util.concurrent.SingleThreadEventExecutor#execute
@Override
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```
由于最终会执行到wakeup上，我们发现它不仅将wakeUp改为true, 并且还手动唤醒了selector;
```java
@Override
protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop && wakenUp.compareAndSet(false, true)) {
        selector.wakeup();
    }
}
```
wakeup释义: 
> Causes the first selection operation that has not yet returned to return immediately.
