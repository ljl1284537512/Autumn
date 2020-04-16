# writeAndFlush原理

## 示例代码
我们尝试通过如下示例代码来看下writeAndFlush的整个过程
```java
public void start() throws InterruptedException {
    NioEventLoopGroup group = new NioEventLoopGroup();
    Bootstrap bootstrap = new Bootstrap();
    bootstrap.group(group)
            .channel(NioSocketChannel.class)
            .remoteAddress(new InetSocketAddress(host,port))
            .handler(new ChannelInboundHandlerAdapter(){
                int counter = 0;
                @Override
                public void channelActive(ChannelHandlerContext ctx) throws Exception {
                    // 当连接建立成功之后，向服务器写数据
                    byte[] req = ("I'm client").getBytes();
                    // 新建缓冲区
                    ByteBuf message = Unpooled.buffer(req.length);
                    // 将数据写入到缓冲区
                    message.writeBytes(req);
                    // ...
                    ctx.writeAndFlush(message);
                }
            });
    try {
        ChannelFuture channelFuture = bootstrap.connect().sync();
        channelFuture.channel().closeFuture().sync();
    }finally {
        group.shutdownGracefully();
    }
}
```
## 断点跟踪
首先我们启动服务器，然后在启动客户端；
```java
@Override
io.netty.channel.AbstractChannelHandlerContext#writeAndFlush(java.lang.Object)
public ChannelFuture writeAndFlush(Object msg) {
    return writeAndFlush(msg, newPromise());
}
```
主要是调用了`ChannelHandlerContext`执行`writeAndFlush`操作；
```java
@Override
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    if (msg == null) {
        throw new NullPointerException("msg");
    }
    // 校验promise
    if (isNotValidPromise(promise, true)) {
        ReferenceCountUtil.release(msg);
        // cancelled
        return promise;
    }
    // 写入数据
    write(msg, true, promise);

    return promise;
}
```
我们继续跟踪`write`操作，`write`第二个参数为是否需要刷新;
```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    // 寻找下一个HandlerContext
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    // 是否是EventLoop线程
    if (executor.inEventLoop()) {
        if (flush) {
            // 调用下一个ChannelHandlerContext执行刷新操作；
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
```
首先要获取到下一个`ChannelHandlerContext`， 然后调用它的`invokeWriteAndFlush`操作,初步猜测这是一个传播的过程。
我们执行到`next.invokeWriteAndFlush`里面：
```java
private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}
```
我们看目前已经进入的Context是`HeadContext`:
![当前执行的ChannelHandlerContext#HeadContext](_v_images/20191120073558015_11597.png =850x)

这里呈现了非常重要的两个部分, 我们分开介绍:
```java
// 写入数据
invokeWrite0(msg, promise);
// 刷新数据
invokeFlush0();
```

## `invokeWrite0()`
目前是在`HeadContext`中执行：
```java
private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
```
继续调用`HeadContext`来执行`write`操作：
```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```
如上所述，内部调用了`unsafe`来操作，我们看其实是`NioSocketChannelUnsafe`
> `NioSocketChannelUnsafe` 是存放在`Channel`中的，只是从`Channel`中取出来；

![](_v_images/20191120074300878_3862.png =850x)
我们再看下`Unsafe`内部的实现：
```java
@Override
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    // 1. 
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        // If the outboundBuffer is null we know the channel was closed and so
        // need to fail the future right away. If it is not null the handling of the rest
        // will be done in flush0()
        // See https://github.com/netty/netty/issues/2362
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        // release message now to prevent resource-leak
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        // 2. 
        msg = filterOutboundMessage(msg);
        // 3. 4.
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }

    // 5.
    outboundBuffer.addMessage(msg, size, promise);
}
```
1. 取出`outboundBuffer`，这个是由`Unsafe`内部定义的，并有给了默认的值：
```java
ChannelOutboundBuffer outboundBuffer = new ChannelOutboundBuffer(AbstractChannel.this);
```
ChannelOutBoundBuffer这个类，内部有几个属性，我们先看下这几个属性：
```java
    private final Channel channel;
    // Entry(flushedEntry) --> ... Entry(unflushedEntry) --> ... Entry(tailEntry)
    //
    // The Entry that is the first in the linked-list structure that was flushed
    // 已被刷新的Entry
    private Entry flushedEntry;
    // The Entry which is the first unflushed in the linked-list structure
    // 未被刷新的Entry
    private Entry unflushedEntry;
    // The Entry which represents the tail of the buffer
    // 缓冲区的尾部Entry
    private Entry tailEntry;
    // The number of flushed entries that are not written yet
    // 已经被执行刷新的Entry节点数量
    private int flushed;

    private int nioBufferCount;
    private long nioBufferSize;

    private boolean inFail;
```

我们接着看`write`操作
2. `filterOutboundMessage(msg);`像是要过滤一下出站的数据：
```java
@Override
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        // 校验是否是直接缓冲区
        if (buf.isDirect()) {
            // 如果是，直接返回
            return msg;
        }

        return newDirectBuffer(buf);
    }

    if (msg instanceof FileRegion) {
        return msg;
    }

    throw new UnsupportedOperationException(
            "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```
以上逻辑，我们先不用关注，直接跟着断点，返回原有的msg;
3. ` pipeline.estimatorHandle().size(msg);`操作中的`estimatorHandle()`方法，主要是为了获取`MessageSizeEstimator`的`handler`, 而这个`MessageSizeEstimator`主要是用于计算msg的大小用：
```java
/**
 * Returns {@link MessageSizeEstimator} which is used for the channel
 * to detect the size of a message.
 */
MessageSizeEstimator getMessageSizeEstimator();
```
4. 进入`DefaultMessageSizeEstimator`内部，计算msg的大小：
```java
@Override
public int size(Object msg) {
    if (msg instanceof ByteBuf) {
        // 如果是ByteBuf实例，则直接返回其可读的字节数
        return ((ByteBuf) msg).readableBytes();
    }
    if (msg instanceof ByteBufHolder) {
        return ((ByteBufHolder) msg).content().readableBytes();
    }
    if (msg instanceof FileRegion) {
        return 0;
    }
    return unknownSize;
}
/**
 * Returns the number of readable bytes which is equal to
 * {@code (this.writerIndex - this.readerIndex)}.
 */
public abstract int readableBytes();
```
5. `outboundBuffer.addMessage(msg, size, promise);`通过`outboundBuffer`增加数据：
```java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    // (1)根据msg 新建一个Entry
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        (1.1)
        Entry tail = tailEntry;
        tail.next = entry;
    }
    // (2)
    tailEntry = entry;
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    // (3)
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```
(1) 将msg封装成一个Entry;
(1.1) 如果当前tailEntry == null ，说明是最后一个entry == null，则标识当前flushedEntry = null ，否则，说明当前outboundBuffer的最后一个tail是有值得，比如第二次进行写入时；这个时候需要把新创建的entry，连接到最后一个tail的next指针之后。
(2) 将`outboundBuffer`的`tailEntry`置为当前新创建的`entry`;
(3) 标识当前的`entry`为`unflushedEntry`;
(4) 在把entry添加到`outboundbuffer`中之后，`incrementPendingOutboundBytes`操作通过CAS原理，对`outboundBuffer`中的`totalPendingSize`属性值进行累加；

整个`write`就结束了；

## `invokeFlush0()`
`invokeFlush0`操作内部实现为：
```java
io.netty.channel.AbstractChannelHandlerContext#invokeFlush0
private void invokeFlush0() {
    try {
        ((ChannelOutboundHandler) handler()).flush(this);
    } catch (Throwable t) {
        notifyHandlerException(t);
    }
}
```
我们直接看`handler`中`flush`的实现如下, 也是调用了`unsafe#NioSocketChannelUnsafe`的flush操作：
```java
@Override
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}
```
跟进来之后
```java
@Override
public final void flush() {
    assertEventLoop();
    // 取出内部的outboundBuffer
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }
    outboundBuffer.addFlush();
    flush0();
}
```
接下来我们重点看下`addFlush()`操作与`flush0`操作
1. addFlush()
```java
public void addFlush() {
    // There is no need to process all entries if there was already a flush before and no new messages
    // where added in the meantime.
    //
    // See https://github.com/netty/netty/issues/2577
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            // there is no flushedEntry yet, so start with the entry
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                // Was cancelled so make sure we free up memory and notify about the freed bytes
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}
```
首先取出未刷新的entry, 这里的`unflushEntry`对应我们上一步写入的entry; 然后判断
```java
if (flushedEntry == null) {
    // there is no flushedEntry yet, so start with the entry
    flushedEntry = entry;
}
```
这里主要是要说明，如果已经有执行flush操作后的entry, 那么要从那个entry开始，否则就从当前取出的未flush的entry开始；暂时不用考虑`(!entry.promise.setUncancellable()`里面的逻辑，跟着断点直接跳出，当前`outboundBuffer`中的flushedEntry被置为上一步写入的entry, 而flushed值 = 1；

2. flush0()
```java
@Override
protected final void flush0() {
    // Flush immediately only when there's no pending flush.
    // If there's a pending flush operation, event loop will call forceFlush() later,
    // and thus there's no need to call it now.
    if (!isFlushPending()) {
        super.flush0();
    }
}
```
`isFlushPending`操作用于判断当前是否有正需要冲刷的操作，如果有的话，可以暂停当前操作，等会儿一起冲刷；
```java
private boolean isFlushPending() {
    SelectionKey selectionKey = selectionKey();
    // 校验当前是否有write事件
    return selectionKey.isValid() && (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0;
}
```
执行`super.flush0()`之后，调用`AbstractChannel`的`flush0`方法，如下：
```java
protected void flush0() {
    if (inFlush0) {
        // Avoid re-entrance
        return;
    }

    // 1.
    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }

    // 标记inFlush0标识，表示正在进行flush操作
    inFlush0 = true;

    ...
    doWrite(outboundBuffer);
    ...

}
```
(1). 取出当前channel的Unsafe中的outboundBuffer;
(2). `doWrite`操作进入了`NioSocketChannel`类中，如下:
```java
@Override
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    // 取出封装好的java channel
    SocketChannel ch = javaChannel();
    int writeSpinCount = config().getWriteSpinCount();
    do {
        ...

            // Ensure the pending writes are made of ByteBufs only.
            int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
            ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
            int nioBufferCnt = in.nioBufferCount();

            // Always us nioBuffers() to workaround data-corruption.
            // See https://github.com/netty/netty/issues/2761
            switch (nioBufferCnt) {
                case 0:
                    // We have something else beside ByteBuffers to write so fallback to normal writes.
                    writeSpinCount -= doWrite0(in);
                    break;
                case 1: {
                    // Only one ByteBuf so use non-gathering write
                    // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
                    // to check if the total size of all the buffers is non-zero.
                    ByteBuffer buffer = nioBuffers[0];
                    int attemptedBytes = buffer.remaining();
                    final int localWrittenBytes = ch.write(buffer);
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
                default: {
                    ...
                    break;
                }
            }
        } while (writeSpinCount > 0);

        incompleteWrite(writeSpinCount < 0);
    }
```
我们先不管`nioBufferCnt`的值，先跟着断电走到`case 1`的位置：
(1). 取出第一个ByteBuffer;
(2). 调用java管道`SocketChannelImpl`的`write`操作，完成写出；
